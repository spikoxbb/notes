[TOC]

# Sender

## 问题 

1. 大面积报警会压垮发送接口
2. 不能及时报警导致积压

## Sender职责

### 1.提供一个缓存Queue,缓存未能及时发出的报警，为了防止Sender挂掉而丢失未发出的报警，因此使用外部消息队列

### 2.支持用户配置接口所能承担的最大访问量，控制接口调用的gorountine数量

```go
//队列名
const (
	IM_QUEUE_NAME   = "/falcon/im"
	SMS_QUEUE_NAME  = "/falcon/sms"
	MAIL_QUEUE_NAME = "/falcon/mail"
)
//控制并发配置
  "worker": {
        "im": 10,
        "sms": 10,
        "mail": 50
    },
```

## 源码解读

首先初始化：

```go
func init() {
	runtime.GOMAXPROCS(runtime.NumCPU())
	log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)
}

func InitSenderWorker() {
	workerConfig := g.Config().Worker
	IMWorkerChan = make(chan int, workerConfig.IM)
	SmsWorkerChan = make(chan int, workerConfig.Sms)
	MailWorkerChan = make(chan int, workerConfig.Mail)
}

//parseConfig太过简单不多说

//http
func Start() {
	if !g.Config().Http.Enabled {
		return
	}
	addr := g.Config().Http.Listen
	if addr == "" {
		return
	}

	r := gin.Default()
	r.GET("/version", Version)
	r.GET("/health", Health)
	r.GET("/workdir", Workdir)
	r.POST("/syncUser", SyncUsers)
	r.POST("/syncGroup", SyncGroups)
	r.POST("/cleanCase", CleanCase)
	r.Run(addr)

	log.Println("http listening", addr)
}
//以version为例
func Version(c *gin.Context) {
	c.String(200, g.VERSION)
}

//监听操作系统的信号
sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigs
		fmt.Println()
		g.RedisConnPool.Close()
		os.Exit(0)
	}()
```

消费信息,并发控制

```go
//持续消费
func ConsumeSms() {
	for {
		L := redi.PopAllSms()
		if len(L) == 0 {
			time.Sleep(time.Millisecond * 200)
			continue
		}
		SendSmsList(L)
	}
}

func SendSmsList(L []*model.Sms) {
	for _, sms := range L {
		SmsWorkerChan <- 1
		go SendSms(sms)
	}
}

//与源码稍稍有些不同，控制报警频率

func SendSms(sms *model.Sms) {
	defer func() {
		<-SmsWorkerChan
	}()

	url := g.Config().Api.Sms

	phones := strings.Split(sms.Tos, ",")
	checkPhones := []string{}
	for _, phone := range phones {
		//电话报警每个人半小时收到一次，所以使用redis锁 每个电话号码锁半个小时
		ok, _ := redi.TryLock(REDIS_PHONE_FLAG+phone, 30*60)
		if ok {
			checkPhones = append(checkPhones, phone)
		}
	}
	if len(checkPhones) == 0 {
		log.Printf("[W]send sms:%v, url:%s, all phone in lock", sms, url)
		return
	}

	r := httplib.Post(url).SetTimeout(5*time.Second, 30*time.Second)
	r.Header("Tuya-Monitor", "Tuya@airtake@monitor@2017.04")
	r.Param("jobNos", strings.Join(checkPhones, ","))
	r.Param("message", sms.Content)
	resp, err := r.Response()
	if err != nil || resp.StatusCode != 200 {
		if err != nil {
			log.Printf("[E]send sms fail, tos:%s, cotent:%s, error:%v", sms.Tos, sms.Content, err)
		} else {
			log.Printf("[E]send sms fail, tos:%s, cotent:%s, statusCode:%d", sms.Tos, sms.Content, resp.StatusCode)
		}

		//失败的情况下解锁
		for _, phone := range checkPhones {
			redi.UnLock(REDIS_PHONE_FLAG + phone)
		}
	} else {
        //insert into db through api module
		api.SaveCall(alarm.APIEventCallInput{
			UserTags: sms.Tos,
			EventIds: sms.EventIds,
			CallType: 1,
		})
	}

	logrus.Debugf("send sms:%v, resp:%v, url:%s", sms, resp, url)
}

//取消息消费
func PopAllSms() []*model.Sms {
	ret := []*model.Sms{}
	queue := SMS_QUEUE_NAME

	rc := g.RedisConnPool.Get()
	defer rc.Close()

	for {
		reply, err := redis.String(rc.Do("RPOP", queue))
		if err != nil {
			if err != redis.ErrNil {
				log.Error(err)
			}
			break
		}

		if reply == "" || reply == "nil" {
			continue
		}

		var sms model.Sms
		err = json.Unmarshal([]byte(reply), &sms)
		if err != nil {
			log.Error(err, reply)
			continue
		}

		ret = append(ret, &sms)
	}

	return ret
}
```