[TOC]

# Graph

## 职责

Graph将采集与传送组件每次push上来的数据，进行采样存储，并提供查询接口。

## code

```go
	// rrdtool init
	rrdtool.InitChannel()
	// rrdtool before api for disable loopback connection
	rrdtool.Start()
	// start api
	go api.Start()
	// start indexing
	index.Start()
	// start http server
	go http.Start()
	go cron.CleanCache()
```

为了减少读写rrd文件的次数，会在本地缓存这个接收到的数据,而且为了快速查找文件在本地建立索引index，方便查找。

HandleItems是存储数据和建立index的:

```go
//发送数据rpc接口
func (this *Graph) Send(items []*cmodel.GraphItem, resp *cmodel.SimpleRpcResponse) error {
	go handleItems(items)
	return nil
}

func handleItems(items []*cmodel.GraphItem) {
	if items == nil {
		return
	}

	count := len(items)
	if count == 0 {
		return
	}

	cfg := g.Config()

	for i := 0; i < count; i++ {
		if items[i] == nil {
			continue
		}

		endpoint := items[i].Endpoint
		if !g.IsValidString(endpoint) {
			if cfg.Debug {
				log.Printf("invalid endpoint: %s", endpoint)
			}
			pfc.Meter("invalidEnpoint", 1)
			continue
		}

		counter := cutils.Counter(items[i].Metric, items[i].Tags)
		if !g.IsValidString(counter) {
			if cfg.Debug {
				log.Printf("invalid counter: %s/%s", endpoint, counter)
			}
			pfc.Meter("invalidCounter", 1)
			continue
		}

		dsType := items[i].DsType
		step := items[i].Step
		checksum := items[i].Checksum()
        //md5(endpoint/metric/project=falcon)_dstype_step
		key := g.FormRrdCacheKey(checksum, dsType, step)

		//statistics
		proc.GraphRpcRecvCnt.Incr()

		// To Graph
        //添加到item的本地缓存中
		first := store.GraphItems.First(key)
		if first != nil && items[i].Timestamp <= first.Timestamp {
			continue
		}
        //放入到本地缓存，定期刷写到磁盘中
		store.GraphItems.PushFront(key, items[i], checksum, cfg)

		// To Index 建立本地索引
		index.ReceiveItem(items[i], checksum)

		// To History
		store.AddItem(checksum, items[i])
	}
}
```

刷新到磁盘：

```go
func syncDisk() {
	time.Sleep(time.Second * g.CACHE_DELAY)
	ticker := time.NewTicker(time.Millisecond * g.FLUSH_DISK_STEP)
	defer ticker.Stop()
	var idx int = 0

	for {
		select {
		case <-ticker.C:
			idx = idx % store.GraphItems.Size
			FlushRRD(idx, false)
			idx += 1
		case <-Out_done_chan:
			log.Println("cron recv sigout and exit...")
			return
		}
	}
}
```

建立本地索引ReceiveItem，增量添加到数据库mysql中:

```go
// index收到一条新上报的监控数据,尝试用于增量更新索引
func ReceiveItem(item *cmodel.GraphItem, md5 string) {
	if item == nil {
		return
	}
    //endpoint/metric/dstype/step/project=falcon
	uuid := item.UUID()

	// 已上报过的数据
	if IndexedItemCache.ContainsKey(md5) {
		old := IndexedItemCache.Get(md5).(*IndexCacheItem)
		if uuid == old.UUID { // dsType+step没有发生变化,只更新缓存
			IndexedItemCache.Put(md5, NewIndexCacheItem(uuid, item))
		} else { // dsType+step变化了,当成一个新的增量来处理
			unIndexedItemCache.Put(md5, NewIndexCacheItem(uuid, item))
		}
		return
	}

	// 针对 mysql索引重建场景 做的优化，是否有rrdtool文件存在,如果有 则认为MySQL中已建立索引；
	rrdFileName := g.RrdFileName(g.Config().RRD.Storage, md5, item.DsType, item.Step)
	if g.IsRrdFileExist(rrdFileName) {
		IndexedItemCache.Put(md5, NewIndexCacheItem(uuid, item))
		return
	}

	// 缓存未命中, 放入增量更新队列
	unIndexedItemCache.Put(md5, NewIndexCacheItem(uuid, item))
}
```

建立增量索引操作index_update_incr_task.go/StartIndexUpdateIncrTask 操作，他会定时的启动updateIndexIncr(）操作:

```go
// 进行一次增量更新
func updateIndexIncr() int {
	ret := 0
	if unIndexedItemCache == nil || unIndexedItemCache.Size() <= 0 {
		return ret
	}

	keys := unIndexedItemCache.Keys()
	for _, key := range keys {
		icitem := unIndexedItemCache.Get(key)
		unIndexedItemCache.Remove(key)
		if icitem != nil {
			// 并发更新mysql
			semaUpdateIndexIncr.Acquire()
			go func(key string, icitem *IndexCacheItem) {
				defer semaUpdateIndexIncr.Release()
				err := updateIndexFromOneItem(icitem.Item)
				if err != nil {
					proc.IndexUpdateIncrErrorCnt.Incr()
				} else {
					IndexedItemCache.Put(key, icitem)
				}
			}(key, icitem.(*IndexCacheItem))
			ret++
		}
	}

	return ret
}


// 根据item,更新mysql
func updateIndexFromOneItem(item *cmodel.GraphItem) error {
	if item == nil {
		return nil
	}

	ts := item.Timestamp
	var endpointId uint64 = 0

	// endpoint表
	url := fmt.Sprintf("%s/api/v1/graph/endpoint", g.Config().Api.Api)
	var endpoint = graph.Endpoint{
		Endpoint: item.Endpoint,
		Ts:       ts,
	}
	bytes, _ := json.Marshal(endpoint)
	request, b := g.HttpRequest(url, bytes, g.POST)
	if b {
		json.Unmarshal(request, &endpoint)
		proc.IndexUpdateIncrDbEndpointInsertCnt.Incr()
	}

	endpointId = endpoint.ID
	if endpointId <= 0 {
		log.Errorf("no such endpoint in db, endpoint=%s", item.Endpoint)
		return errors.New("no such endpoint")
	}

	// tag_endpoint表
	url = fmt.Sprintf("%s/api/v1/graph/tag_endpoint", g.Config().Api.Api)
	for tagKey, tagVal := range item.Tags {
		tag := fmt.Sprintf("%s=%s", tagKey, tagVal)
		var tagEndpoint = graph.TagEndpoint{
			Tag:        tag,
			EndpointID: endpointId,
			Ts:         ts,
		}
		bytes, _ := json.Marshal(tagEndpoint)
		_, b := g.HttpRequest(url, bytes, g.POST)
		if b {
			proc.IndexUpdateIncrDbTagEndpointInsertCnt.Incr()
		}
	}

	// endpoint_counter表
	counter := item.Metric
	if len(item.Tags) > 0 {
		counter = fmt.Sprintf("%s/%s", counter, cutils.SortedTags(item.Tags))
	}

	url = fmt.Sprintf("%s/api/v1/graph/endpoint_counter", g.Config().Api.Api)
	var endpointCounter = graph.EndpointCounter{
		EndpointID: endpointId,
		Counter:    counter,
		Step:       item.Step,
		Type:       item.DsType,
		Ts:         ts,
	}
	bytes, _ = json.Marshal(endpointCounter)
	_, b = g.HttpRequest(url, bytes, g.POST)
	if b {
		proc.IndexUpdateIncrDbEndpointCounterInsertCnt.Incr()
	}

	return nil
}
```

这里有三个表需要更新：

a、endpoint 表。该表记录了所有上报数据的endpoint，并且为每一个endpoint生成一个id即 endpoint_id。

b、tag_endpoint表。拆解item的每一个tag。用tag和endpoint形成一个主键的表。记录每个endpoint包含的tag。每条记录生成一个id，为tagendpoint_id

c、endpoint_counter表。counter是metric和tags组合后的名词。看作是一个整体。

其实这个index最后都转化成了这三个表。这三个表的意义呢？在于何处？ 答案是在与rrd文件的索引。表中并没有直接保存rrd文件的名字。如果查询的时候该怎么知道去查询哪一个rrd文件呢？不可能所有的rrd文件的头部都扫描一遍, filename=f（Endpoint，Metric，Tags，dstype，step）相关，所以要准确的找到rrd文件，必须凑齐这5个元素。

```go
type GraphQueryParam struct {
	Start     int64  `json:"start"`
	End       int64  `json:"end"`
	ConsolFun string `json:"consolFuc"`
	Endpoint  string `json:"endpoint"`
	Counter   string `json:"counter"`
}
```

有效的是 时间段start和end、endpoint、counter。counter是(Metric，Tags)的组合。为了凑齐5个条件组成rrdfilename缺少的是dstype和step。那么这时候可以根据endpoint和counter在endpoint_counter表中找到dstype和step。然后组合成rrdfilename。进行读取数据。

```go
func (this *Graph) Query(param cmodel.GraphQueryParam, resp *cmodel.GraphQueryResponse) error {
	// statistics
	proc.GraphQueryCnt.Incr()

	// form empty response
	resp.Values = []*cmodel.RRDData{}  //------》用于存放获取的数据
	resp.Endpoint = param.Endpoint          // -------》数据的endpoint
	resp.Counter = param.Counter         // ----------》数据的counter信息
	dsType, step, exists := index.GetTypeAndStep(param.Endpoint, param.Counter) // complete dsType and step //----------->从缓存或者DB中获取dstype和step。这里DB同样使用了一层自己的缓存
	if !exists {
		return nil
	}
	resp.DsType = dsType
	resp.Step = step

	start_ts := param.Start - param.Start%int64(step)      //------》根据step对齐整理start时间
	end_ts := param.End - param.End%int64(step) + int64(step) //-----》根据step对齐整理end时间
	if end_ts-start_ts-int64(step) < 1 {
		return nil
	}

	md5 := cutils.Md5(param.Endpoint + "/" + param.Counter) //------->计算md5值，用于计算key值
	ckey := g.FormRrdCacheKey(md5, dsType, step)              //-----》计算key值,用于缓存索引，这个缓存是数据缓存，不是index缓存
	filename := g.RrdFileName(g.Config().RRD.Storage, md5, dsType, step)  //还原rrd文件名字
	// read data from rrd file
	datas, _ := rrdtool.Fetch(filename, param.ConsolFun, start_ts, end_ts, step) //从rrd中获取数据，从rrd中获取数据需要指定获取数据的时间段。
	datas_size := len(datas)
	// read cached items
	items := store.GraphItems.FetchAll(ckey)  //根据key值，在数据缓存中获取数据。
	items_size := len(items)
```

最后根据 从rrd中获取的数据和从数据缓存中获取的数据进行合并，输出。完成查询。