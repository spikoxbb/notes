[TOC]

# Alarm

## 职责

1. 报警事件对的读取

   按优先级轮训redis读取报警事件

   对高优先级报警事件生成报警邮件短信写入redis,由Sender模块读取

2. 对业务系统的回调

   对需要回调的时间回调业务系统接口

3. 报警合并逻辑分析

   对低优先级报警合并

4. 展示未恢复的报警

   提供web页面，报警时间处理之前存在Alarm内存中

## code

main.go:

```go
    go http.Start()
	go cron.ReadHighEvent()
	go cron.ReadLowEvent()
	go cron.CombineSms()
	go cron.CombineMail()
	go cron.CombineIM()
	go cron.ConsumeIM()
	go cron.ConsumeSms()
	go cron.ConsumeMail()
	go cron.CleanExpiredEvent()
	cron.Crontab_Task()   
```

读取报警事件：

```go
func ReadHighEvent() {
	queues := g.Config().Redis.HighQueues
	if len(queues) == 0 {
		return
	}

	for {
		event, err := popEvent(queues)
		if err != nil {
			time.Sleep(time.Second)
			continue
		}
		consume(event, true)
	}
}

func popEvent(queues []string) (*cmodel.Event, error) {
	count := len(queues)

	params := make([]interface{}, count+1)
	for i := 0; i < count; i++ {
		params[i] = queues[i]
	}
	// set timeout 0
	params[count] = 0

	rc := g.RedisConnPool.Get()
	defer rc.Close()
    //BRPOP event:p0 event:p1 event:p2 0 
	reply, err := redis.Strings(rc.Do("BRPOP", params...))
	if err != nil {
		log.Errorf("get alarm event from redis fail: %v", err)
		return nil, err
	}

	var event cmodel.Event
	err = json.Unmarshal([]byte(reply[1]), &event)
	if err != nil {
		log.Errorf("parse alarm event fail: %v", err)
		return nil, err
	}

	log.Debugf("pop event: %s", event.String())

	//insert event into database
	err = eventmodel.InsertEvent(&event)
	// events no longer saved in memory

	return &event, err
}
```

### 对业务系统的回调

```go
func consume(event *cmodel.Event, isHigh bool) {

	//使用ignored做报警关闭功能
	if ignored(event.Id) {
		return
	}

	actionId := event.ActionId()
	if actionId <= 0 {
		return
	}

	action := api.GetAction(actionId)
	if action == nil {
		return
	}
    //配置了callback地址
	if action.Callback == 1 {
		HandleCallback(event, action)
	}

	if isHigh {
		consumeHighEvents(event, action)
	} else {
		consumeLowEvents(event, action)
	}
}

func HandleCallback(event *model.Event, action *api.Action) {

	teams := action.Uic
	phones := []string{}
	mails := []string{}
	ims := []string{}
	lastEvent := event2.CaseCache.GetLastEvent(event.Id)
	eventIds := []string{strconv.FormatUint(lastEvent, 10)}

	if teams != "" {
		phones, mails, ims = api.ParseTeams(event.Endpoint, teams, event.Strategy)
		smsContent := GenerateSmsContent(event)
		mailContent := GenerateMailContent(event)
		imContent := GenerateIMContent(event)
		if action.BeforeCallbackSms == 1 {
			redi.WriteSms(phones, smsContent, eventIds)
			redi.WriteIM(ims, imContent, eventIds)
		}

		if action.BeforeCallbackMail == 1 {
			redi.WriteMail(mails, smsContent, mailContent)
		}
	}

	message := Callback(event, action)

	if teams != "" {
		if action.AfterCallbackSms == 1 {
			redi.WriteSms(phones, message, eventIds)
			redi.WriteIM(ims, message, eventIds)
		}

		if action.AfterCallbackMail == 1 {
			redi.WriteMail(mails, message, message)
		}
	}

}

//生成短信
func GenerateSmsContent(event *model.Event) string {
	return BuildCommonSMSContent(event)
}
func BuildCommonSMSContent(event *model.Event) string {
	return fmt.Sprintf(
		"[P%d][%s][%s][%s/%s][%s%s%s]",
		event.Priority(),
		event.Status,
		event.Endpoint,
		event.Metric(),
		utils.SortedTags(event.PushedTags),
		utils.ReadableFloat(event.LeftValue),
		event.Operator(),
		utils.ReadableFloat(event.RightValue()),
	)
}

func Callback(event *model.Event, action *api.Action) string {
	if action.Url == "" {
		return "callback url is blank"
	}

	L := make([]string, 0)
	if len(event.PushedTags) > 0 {
		for k, v := range event.PushedTags {
			L = append(L, fmt.Sprintf("%s:%s", k, v))
		}
	}
    //拼接tag a:b,c:d
	tags := ""
	if len(L) > 0 {
		tags = strings.Join(L, ",")
	}

	req := httplib.Get(action.Url).SetTimeout(3*time.Second, 20*time.Second)

	req.Param("endpoint", event.Endpoint)
	req.Param("metric", event.Metric())
	req.Param("status", event.Status)
	req.Param("step", fmt.Sprintf("%d", event.CurrentStep))
	req.Param("priority", fmt.Sprintf("%d", event.Priority()))
	req.Param("time", event.FormattedTime())
	req.Param("tpl_id", fmt.Sprintf("%d", event.TplId()))
	req.Param("exp_id", fmt.Sprintf("%d", event.ExpressionId()))
	req.Param("stra_id", fmt.Sprintf("%d", event.StrategyId()))
	req.Param("left_value", utils.ReadableFloat(event.LeftValue))
	req.Param("tags", tags)

	resp, e := req.String()

	success := "success"
	if e != nil {
		log.Errorf("callback fail, action:%v, event:%s, error:%s", action, event.String(), e.Error())
		success = fmt.Sprintf("fail:%s", e.Error())
	}
	message := fmt.Sprintf("curl %s %s. resp: %s", action.Url, success, resp)
	log.Debugf("callback to url:%s, event:%s, resp:%s", action.Url, event.String(), resp)

	return message
}

func consumeHighEvents(event *cmodel.Event, action *api.Action) {
	if action.Uic == "" {
		return
	}

	phones, mails, ims := api.ParseTeams(event.Endpoint, action.Uic, event.Strategy)

	smsContent := GeneratePhoneFormatContent(event)
	mailContent := GenerateMailContent(event)
	imContent := GenerateIMFormatContent(event)

	eventId := emodel.CaseCache.GetLastEvent(event.Id)
	// <=P2 才发送短信
	if event.Priority() < 3 && event.Status != "OK" {
		redi.WriteSms(phones, smsContent, []string{strconv.FormatUint(eventId, 10)})
	}

	redi.WriteIM(ims, imContent, []string{strconv.FormatUint(eventId, 10)})
	redi.WriteMail(mails, smsContent, mailContent)

}

// 低优先级的做报警合并
func consumeLowEvents(event *cmodel.Event, action *api.Action) {
	if action.Uic == "" {
		return
	}

	// <=P2 才发送短信
	if event.Priority() < 3 {
		//半小时电话一次，记录每次发送消息的时间
		ParseUserSms(event, action)
	}

	ParseUserIm(event, action)
	ParseUserMail(event, action)
}
//放入redis
func ParseUserSms(event *cmodel.Event, action *api.Action) {
	userMap := api.GetUsers(event.Endpoint, action.Uic, event.Strategy)

	content := GenerateSmsContent(event)
	metric := event.Metric()
	status := event.Status
	priority := event.Priority()

	queue := g.Config().Redis.UserSmsQueue
	eventId := emodel.CaseCache.GetLastEvent(event.Id)
	rc := g.RedisConnPool.Get()
	defer rc.Close()

	for _, user := range userMap {
		dto := SmsDto{
			Priority: priority,
			Metric:   metric,
			Content:  content,
			Phone:    user.IM,
			Status:   status,
			EventId:  eventId,
		}
		bs, err := json.Marshal(dto)
		if err != nil {
			log.Error("json marshal SmsDto fail:", err)
			continue
		}

		_, err = rc.Do("LPUSH", queue, string(bs))
		if err != nil {
			log.Error("LPUSH redis", queue, "fail:", err, "dto:", string(bs))
		}
	}
}

//合并报警

func CombineSms() {
	for {
		// 每分钟读取处理一次
		time.Sleep(time.Minute)
		combineSms()
	}
}

//取出所有DTO
func popAllSmsDto() []*SmsDto {
	ret := []*SmsDto{}
	queue := g.Config().Redis.UserSmsQueue

	rc := g.RedisConnPool.Get()
	defer rc.Close()

	for {
		reply, err := redis.String(rc.Do("RPOP", queue))
		if err != nil {
			if err != redis.ErrNil {
				log.Error("get SmsDto fail", err)
			}
			break
		}

		if reply == "" || reply == "nil" {
			continue
		}

		var smsDto SmsDto
		err = json.Unmarshal([]byte(reply), &smsDto)
		if err != nil {
			log.Errorf("json unmarshal SmsDto: %s fail: %v", reply, err)
			continue
		}

		ret = append(ret, &smsDto)
	}

	return ret
}

//合并插入数据库返回链接
func combineSms() {
	dtos := popAllSmsDto()
	count := len(dtos)
	if count == 0 {
		return
	}

	dtoMap := make(map[string][]*SmsDto)
	for i := 0; i < count; i++ {
		key := fmt.Sprintf("%d%s%s%s", dtos[i].Priority, dtos[i].Status, dtos[i].Phone, dtos[i].Metric)
		if _, ok := dtoMap[key]; ok {
			dtoMap[key] = append(dtoMap[key], dtos[i])
		} else {
			dtoMap[key] = []*SmsDto{dtos[i]}
		}
	}

	for _, arr := range dtoMap {
		size := len(arr)
		if size == 1 {
			redi.WriteSms([]string{arr[0].Phone}, arr[0].Content, []string{strconv.FormatUint(arr[0].EventId, 10)})
			continue
		}

		// 把多个sms内容写入数据库，只给用户提供一个链接
		contentArr := make([]string, size)
		eventArr := make([]string, size)
		for i := 0; i < size; i++ {
			contentArr[i] = arr[i].Content
			eventArr[i] = strconv.FormatUint(arr[i].EventId, 10)
		}
		content := strings.Join(contentArr, ",,")

		first := arr[0].Content
        //从报警内容中解析出一台机器
		t := strings.Split(first, "][")
		eg := ""
		if len(t) >= 3 {
			eg = t[2]
		}

		path, err := api.LinkToSMS(content)
		sms := ""
		if err != nil || path == "" {
			sms = fmt.Sprintf("[P%d][%s] %d %s.  e.g. %s. detail in email", arr[0].Priority, arr[0].Status, size, arr[0].Metric, eg)
			log.Error("get short link fail", err)
		} else {
			sms = fmt.Sprintf("[P%d][%s] %d %s e.g. %s %s/alarm/links/%s ",
				arr[0].Priority, arr[0].Status, size, arr[0].Metric, eg, g.Config().Api.Dashboard, path)
			log.Debugf("combined sms is:%s", sms)
		}

		redi.WriteSms([]string{arr[0].Phone}, sms, eventArr)
	}
}
//mail不需要返回link
```

