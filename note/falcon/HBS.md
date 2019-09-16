[TOC]

# HBS

## 模块职责

### 1. 处理agent心跳请求，填充host表(agent version,plugin version etc)

### 2. 将ip白名单分发给agent（trustable）

agent有个run接口，可以接受远端机器的shell指令在本机执行。

### 3. 告诉agent应该执行哪些插件（允许用户自定义采集脚本）

配置repo地址，调用agent update接口触发pull，protal配置。

### 4. 告诉agent应该监听哪些端口，进程

ss -tln 可以拿到本机正在监听的端口；/proc以pid命名的各个子目录拿到进程；只有用户配置了才采集。

### 5. 缓存监控策略

获取所有报警规则缓存在内存里，juge向HBS请求。

agent有个run接口，可以接受远端机器的shell指令在本机执行。

## init

```go
//入口函数
func main() {
	cfg := flag.String("c", "cfg.json", "configuration file")
	version := flag.Bool("v", false, "show version")
	//cfg := "src/tuya/falcon-backend-s/modules/hbs/cfg.example.json"
	//version := false
	flag.Parse()

	if *version {
		fmt.Println(g.VERSION)
		os.Exit(0)
	}

	g.ParseConfig(*cfg)
	g.InitApolloConfig()
    //db init
	db.Init()
    //查询一些数据放入内存
	cache.Init()
    //删除过期agents
	go cache.DeleteStaleAgents()

	go http.Start()
	go rpc.Start()

	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigs
		fmt.Println()
		db.DB.Close()
		os.Exit(0)
	}()

	select {}
}

//cache init
func Init() {
	log.Println("cache begin")

	log.Println("#1 GroupPlugins...")
	GroupPlugins.Init()

	log.Println("#2 GroupTemplates...")
	GroupTemplates.Init()

	log.Println("#3 HostGroupsMap...")
	HostGroupsMap.Init()

	log.Println("#4 HostMap...")
	HostMap.Init()

	log.Println("#5 TemplateCache...")
	TemplateCache.Init()

	log.Println("#6 Strategies...")
	Strategies.Init(TemplateCache.GetMap())

	log.Println("#7 HostTemplateIds...")
	HostTemplateIds.Init()

	log.Println("#8 ExpressionCache...")
	ExpressionCache.Init()

	log.Println("#9 MonitoredHosts...")
	MonitoredHosts.Init()

	log.Println("cache done")

	go LoopInit()

}

//db init
func Init() {
	var err error
	DB, err = sql.Open("mysql", g.Config().Database.Addr)
	if err != nil {
		log.Fatalln("open db fail:", err)
	}

	DB.SetMaxOpenConns(g.Config().Database.MaxConn)
	DB.SetMaxIdleConns(g.Config().Database.MaxIdle)

	err = DB.Ping()
	if err != nil {
		log.Fatalln("ping db fail:", err)
	}
}
```

## db

```go
func UpdateAgent(agentInfo *model.AgentUpdateInfo) {
	sql := ""
    //hostname唯一索引，on duplicate key，重复则更新
	if g.Config().Hosts == "" {
		sql = fmt.Sprintf(
			"insert into host(hostname, ip, agent_version, plugin_version, gmt_modified) values ('%s', '%s', '%s', '%s', unix_timestamp() * 1000) on duplicate key update ip='%s', agent_version='%s', plugin_version='%s', gmt_modified=unix_timestamp() * 1000, deleted_at = null",
			agentInfo.ReportRequest.Hostname,
			agentInfo.ReportRequest.IP,
			agentInfo.ReportRequest.AgentVersion,
			agentInfo.ReportRequest.PluginVersion,
			agentInfo.ReportRequest.IP,
			agentInfo.ReportRequest.AgentVersion,
			agentInfo.ReportRequest.PluginVersion,
		)
	} else {
		// sync, just update
		sql = fmt.Sprintf(
			"update host set ip='%s', agent_version='%s', plugin_version='%s',gmt_modified=unix_timestamp() * 1000 where hostname='%s'",
			agentInfo.ReportRequest.IP,
			agentInfo.ReportRequest.AgentVersion,
			agentInfo.ReportRequest.PluginVersion,
			agentInfo.ReportRequest.Hostname,
		)
	}

	_, err := DB.Exec(sql)
	if err != nil {
		log.Println("exec", sql, "fail", err)
	}

}
//query hosts
func QueryHosts() (map[string]int, error) {
	m := make(map[string]int)

	sql := "select id, hostname from host where deleted_at is null"
	rows, err := DB.Query(sql)
	if err != nil {
		log.Println("ERROR:", err)
		return m, err
	}

	defer rows.Close()
	for rows.Next() {
		var (
			id       int
			hostname string
		)

		err = rows.Scan(&id, &hostname)
		if err != nil {
			log.Println("ERROR:", err)
			continue
		}

		m[hostname] = id
	}

	return m, nil
}
```

## cache package

```go
// 一个HostGroup可以绑定多个Plugin
type SafeGroupPlugins struct {
	sync.RWMutex
	M map[int][]string
}

var GroupPlugins = &SafeGroupPlugins{M: make(map[int][]string)}

func (this *SafeGroupPlugins) GetPlugins(gid int) ([]string, bool) {
	this.RLock()
	defer this.RUnlock()
	plugins, exists := this.M[gid]
	return plugins, exists
}

func (this *SafeGroupPlugins) Init() {
	m, err := db.QueryPlugins()
	if err != nil {
		return
	}

	this.Lock()
	defer this.Unlock()
	this.M = m
}

// 根据hostname获取关联的插件
func GetPlugins(hostname string) []string {
	hid, exists := HostMap.GetID(hostname)
	if !exists {
		return []string{}
	}

	gids, exists := HostGroupsMap.GetGroupIds(hid)
	if !exists {
		return []string{}
	}

	// 因为机器关联了多个Group，每个Group可能关联多个Plugin，故而一个机器关联的Plugin可能重复
	pluginDirs := make(map[string]struct{})
	for _, gid := range gids {
		plugins, exists := GroupPlugins.GetPlugins(gid)
		if !exists {
			continue
		}

		for _, plugin := range plugins {
			pluginDirs[plugin] = struct{}{}
		}
	}

	size := len(pluginDirs)
	if size == 0 {
		return []string{}
	}

	dirs := make([]string, size)
	i := 0
	for dir := range pluginDirs {
		dirs[i] = dir
		i++
	}

	sort.Strings(dirs)
	return dirs
}

//expression
type SafeExpressionCache struct {
	sync.RWMutex
	L []*model.Expression
}

var ExpressionCache = &SafeExpressionCache{}

func (this *SafeExpressionCache) Get() []*model.Expression {
	this.RLock()
	defer this.RUnlock()
	return this.L
}

func (this *SafeExpressionCache) Init() {
	es, err := db.QueryExpressions()
	if err != nil {
		return
	}

	this.Lock()
	defer this.Unlock()
	this.L = es
}
```

## rpc package

```go
//获得策略表达式，judge请求hbs
func (t *Hbs) GetExpressions(req model.NullRpcRequest, reply *model.ExpressionResponse) error {
	reply.Expressions = cache.ExpressionCache.Get()
	return nil
}

//agent请求hbs
func (t *Agent) MinePlugins(args model.AgentHeartbeatRequest, reply *model.AgentPluginsResponse) error {
	if args.Hostname == "" {
		return nil
	}
    //是排好序的，避免md5结果不一致
	plugins = cache.GetPlugins(args.Hostname) 
    //签名
    checksum:=""
    if len(plugins) > 0{
        checksum=utils.Md5(strings.Join(plugins,""))
    }
    if args.chechsum == checksum{
        //agent拿到的签名一直不再见检查plugins
        reply.Plugins = []string{}
    }else{
        reply.Plugins = plugins
    }
    reply.checksum = checksum
	reply.Timestamp = time.Now().Unix()
	return nil
}
```

### go中的rpc

1. tcp
2. http
3. json

rpc.go:

```go
func Start() {
	addr := g.Config().Listen

	server := rpc.NewServer()
	// server.Register(new(filter.Filter))
	server.Register(new(Agent))
	server.Register(new(Hbs))

	l, e := net.Listen("tcp", addr)
	if e != nil {
		log.Fatalln("listen error:", e)
	} else {
		log.Println("listening", addr)
	}

	for {
		conn, err := l.Accept()
		if err != nil {
			log.Println("listener accept fail:", err)
			time.Sleep(time.Duration(100) * time.Millisecond)
			continue
		}
		go server.ServeCodec(jsonrpc.NewServerCodec(conn))
	}
}
```

