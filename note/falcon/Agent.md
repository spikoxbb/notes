[TOC]

# Agent

## 监控数据来源

1. 机器性能指标，比如cpu,内存，网卡，磁盘
2. 业务监控指标，比如某个接口调用的latency
3. 各种开源软件的状态指标，比如nginx,mysql等

## 心跳机制的实现

了解机器上的Agent版本，plugin的版本

获取本机要监听的进程端口以及执行的插件列表

```go
//汇报版本	
cron.ReportAgentStatus()
//同步插件
cron.SyncMinePlugins()
//同步内置metric
cron.SyncBuiltinMetrics()
//同步授信ip
cron.SyncTrustableIps()

func ReportAgentStatus() {
	if g.Config().Heartbeat.Enabled && g.Config().Heartbeat.Addr != "" {
		go reportAgentStatus(time.Duration(g.Config().Heartbeat.Interval) * time.Second)
	}
}

func reportAgentStatus(interval time.Duration) {
	for {
		hostname, err := g.Hostname()
		if err != nil {
			hostname = fmt.Sprintf("error:%s", err.Error())
		}

		req := model.AgentReportRequest{
			Hostname:      hostname,
			IP:            g.IP(),
			AgentVersion:  g.VERSION,
			PluginVersion: g.GetCurrPluginVersion(),
		}

		var resp model.SimpleRpcResponse
		err = g.HbsClient.Call("Agent.ReportStatus", req, &resp)
		if err != nil || resp.Code != 0 {
			log.Println("call Agent.ReportStatus fail:", err, "Request:", req, "Response:", resp)
		}

		time.Sleep(interval)
	}
}

func GetCurrPluginVersion() string {
	if !Config().Plugin.Enabled {
		return "plugin not enabled"
	}

	pluginDir := Config().Plugin.Dir
	if !file.IsExist(pluginDir) {
		return "plugin dir not existent"
	}

	versionFile := path.Join(pluginDir, ".version")
	if file.IsExist(versionFile) {
		version, err := file.ToTrimString(versionFile)
		if err != nil {
			return fmt.Sprintf("read %s fail: %v", versionFile, err)
		}
		return version
	}
    //cd 到目录执行shell指令获取git版本
	cmd := exec.Command("git", "rev-parse", "HEAD")
	cmd.Dir = pluginDir

	var out bytes.Buffer
	cmd.Stdout = &out
	err := cmd.Run()
	if err != nil {
		return fmt.Sprintf("Error:%s", err.Error())
	}

	return strings.TrimSpace(out.String())
}


func SyncMinePlugins() {
	if !g.Config().Plugin.Enabled {
		return
	}

	if !g.Config().Heartbeat.Enabled {
		return
	}

	if g.Config().Heartbeat.Addr == "" {
		return
	}

	go syncMinePlugins()
}

//获取需要监听的端口和进程
func SyncBuiltinMetrics() {
	if g.Config().Heartbeat.Enabled && g.Config().Heartbeat.Addr != "" {
		go syncBuiltinMetrics()
	}
}

func syncBuiltinMetrics() {

	var timestamp int64 = -1
	var checksum string = "nil"

	duration := time.Duration(g.Config().Heartbeat.Interval) * time.Second

	for {
		time.Sleep(duration)

		var ports = []int64{}
		var paths = []string{}
		var procs = make(map[string]map[int]string)
		var urls = make(map[string][]string)

		hostname, err := g.Hostname()
		if err != nil {
			continue
		}

		req := model.AgentHeartbeatRequest{
			Hostname: hostname,
			Checksum: checksum,
		}

		var resp model.BuiltinMetricResponse
		err = g.HbsClient.Call("Agent.BuiltinMetrics", req, &resp)
		if err != nil {
			log.Println("ERROR:", err)
			continue
		}

		if resp.Timestamp <= timestamp {
			continue
		}

		if resp.Checksum == checksum {
			continue
		}

		timestamp = resp.Timestamp
		checksum = resp.Checksum

		for _, metric := range resp.Metrics {
           
			if metric.Metric == g.URL_CHECK_HEALTH {
				arr := strings.Split(metric.Tags, ",")
				if len(arr) != 2 && len(arr) != 3 {
					continue
				}
				url := strings.Split(arr[0], "=")
				if len(url) != 2 {
					continue
				}
				stime := strings.Split(arr[1], "=")
				if len(stime) != 2 {
					continue
				}
				urlReturn := ""
				if len(arr) == 3 {
					ret := strings.Split(arr[2], "=")
					if len(ret) != 2 {
						continue
					}
					urlReturn = ret[1]
				}
				if _, err := strconv.ParseInt(stime[1], 10, 64); err == nil {
					if urlReturn == "" {
						urls[url[1]] = []string{stime[1]}
					} else {
						urls[url[1]] = []string{stime[1], urlReturn}
					}
				} else {
					log.Println("metric ParseInt timeout failed:", err)
				}
			}
            //获取端口
            //metric=net.port.listen tags: port=22
			if metric.Metric == g.NET_PORT_LISTEN {
				arr := strings.Split(metric.Tags, "=")
				if len(arr) != 2 {
					continue
				}

				if port, err := strconv.ParseInt(arr[1], 10, 64); err == nil {
					ports = append(ports, port)
				} else {
					log.Println("metrics ParseInt failed:", err)
				}

				continue
			}
            
            //获取目录大小　
            //metric=du.bs tags: path=/home/work/logs
            //du -bs /home/work/logs
			if metric.Metric == g.DU_BS {
				arr := strings.Split(metric.Tags, "=")
				if len(arr) != 2 {
					continue
				}

				paths = append(paths, strings.TrimSpace(arr[1]))
				continue
			}
            
            // 获取进程
            //metric=proc.num tags: cmdline=falcon-agent-ccfg.json
             //metric=proc.num tags: name=java
           // cmdline是从/proc/$pid/cmdline采集的。这个文件存放的是你启动进程的时候用到的命令，比如你用java -c uic.properties启动了一个Java进程，进程名是java，其实所有的java进程，进程名都是java，那我们是没法通过name字段做区分的。怎么办呢？此时就要求助于这个/proc/$pid/cmdline文件的内容了。
			if metric.Metric == g.PROC_NUM {
				arr := strings.Split(metric.Tags, ",")

				tmpMap := make(map[int]string)

				for i := 0; i < len(arr); i++ {
					if strings.HasPrefix(arr[i], "name=") {
						tmpMap[1] = strings.TrimSpace(arr[i][5:])
					} else if strings.HasPrefix(arr[i], "cmdline=") {
						tmpMap[2] = strings.TrimSpace(arr[i][8:])
					}
				}

				procs[metric.Tags] = tmpMap
			}
		}

		g.SetReportUrls(urls)
		g.SetReportPorts(ports)
		g.SetReportProcs(procs)
		g.SetDuPaths(paths)

	}
}

//ip白名单
func SyncTrustableIps() {
	if g.Config().Heartbeat.Enabled && g.Config().Heartbeat.Addr != "" {
		go syncTrustableIps()
	}
}

func syncTrustableIps() {

	duration := time.Duration(g.Config().Heartbeat.Interval) * time.Second

	for {
		time.Sleep(duration)

		var ips string
		err := g.HbsClient.Call("Agent.TrustableIps", model.NullRpcRequest{}, &ips)
		if err != nil {
			log.Println("ERROR: call Agent.TrustableIps fail", err)
			continue
		}

		g.SetTrustableIps(ips)
	}
}

```

## 无需历史点的采集

```go
//main.go:
funcs.BuildMappers()

type FuncsAndInterval struct {
	Fs       []func() []*model.MetricValue
	Interval int
}

var Mappers []FuncsAndInterval

//每隔interval时间执行函数,一个funcs一个gorountine防止互相影响
func BuildMappers() {
	interval := g.Config().Transfer.Interval
	Mappers = []FuncsAndInterval{
		{
			Fs: []func() []*model.MetricValue{
				AgentMetrics,
				CpuMetrics,
				NetMetrics,
				KernelMetrics,
				LoadAvgMetrics,
				MemMetrics,
				DiskIOMetrics,
				IOStatsMetrics,
				NetstatMetrics,
				ProcMetrics,
				UdpMetrics,
			},
			Interval: interval,
		},
		{
			Fs: []func() []*model.MetricValue{
				DeviceMetrics,
			},
			Interval: interval,
		},
		{
			Fs: []func() []*model.MetricValue{
				PortMetrics,
				SocketStatSummaryMetrics,
			},
			Interval: interval,
		},
		{
			Fs: []func() []*model.MetricValue{
				DuMetrics,
			},
			Interval: interval,
		},
		{
			Fs: []func() []*model.MetricValue{
				UrlMetrics,
			},
			Interval: interval,
		},
		{
			Fs: []func() []*model.MetricValue{
				GpuMetrics,
			},
			Interval: interval,
		},
	}
}

//main.go:
cron.Collect()

func Collect() {

	if !g.Config().Transfer.Enabled {
		return
	}

	if len(g.Config().Transfer.Addrs) == 0 {
		return
	}

	for _, v := range funcs.Mappers {
		go collect(int64(v.Interval), v.Fs)
	}
}

func Collect() {

	if !g.Config().Transfer.Enabled {
		return
	}

	if len(g.Config().Transfer.Addrs) == 0 {
		return
	}

	for _, v := range funcs.Mappers {
		go collect(int64(v.Interval), v.Fs)
	}
}

func collect(sec int64, fns []func() []*model.MetricValue) {
	t := time.NewTicker(time.Second * time.Duration(sec))
	defer t.Stop()
	for {
		<-t.C

		hostname, err := g.Hostname()
		if err != nil {
			continue
		}

		mvs := []*model.MetricValue{}
		ignoreMetrics := g.Config().IgnoreMetrics

		for _, fn := range fns {
			items := fn()
			if items == nil {
				continue
			}

			if len(items) == 0 {
				continue
			}

			for _, mv := range items {
				if b, ok := ignoreMetrics[mv.Metric]; ok && b {
					continue
				} else {
					mvs = append(mvs, mv)
				}
			}
		}

		now := time.Now().Unix()
		for j := 0; j < len(mvs); j++ {
			mvs[j].Step = sec
			mvs[j].Endpoint = hostname
			mvs[j].Timestamp = now
		}

		g.SendToTransfer(mvs)

	}
}

func SendToTransfer(metrics []*model.MetricValue) {
	if len(metrics) == 0 {
		return
	}

	dt := Config().DefaultTags
	if len(dt) > 0 {
		var buf bytes.Buffer
		default_tags_list := []string{}
		for k, v := range dt {
			buf.Reset()
			buf.WriteString(k)
			buf.WriteString("=")
			buf.WriteString(v)
			default_tags_list = append(default_tags_list, buf.String())
		}
		default_tags := strings.Join(default_tags_list, ",")

		for i, x := range metrics {
			buf.Reset()
			if x.Tags == "" {
				metrics[i].Tags = default_tags
			} else {
				buf.WriteString(metrics[i].Tags)
				buf.WriteString(",")
				buf.WriteString(default_tags)
				metrics[i].Tags = buf.String()
			}
		}
	}

	debug := Config().Debug

	if debug {
		log.Printf("=> <Total=%d> %v\n", len(metrics), metrics[0])
	}

	var resp model.TransferResponse
	SendMetrics(metrics, &resp)

	if debug {
		log.Println("<=", &resp)
	}
}

func SendMetrics(metrics []*model.MetricValue, resp *model.TransferResponse) {
	rand.Seed(time.Now().UnixNano())
	for _, i := range rand.Perm(len(Config().Transfer.Addrs)) {
		addr := Config().Transfer.Addrs[i]

		c := getTransferClient(addr)
		if c == nil {
			c = initTransferClient(addr)
		}

		if updateMetrics(c, metrics, resp) {
			break
		}
	}
}

func updateMetrics(c *SingleConnRpcClient, metrics []*model.MetricValue, resp *model.TransferResponse) bool {
	err := c.Call("Transfer.Update", metrics, resp)
	if err != nil {
		log.Println("call Transfer.Update fail:", c, err)
		return false
	}
	return true
}
//平均负载
func LoadAvgMetrics() []*model.MetricValue {
	load, err := nux.LoadAvg()
	if err != nil {
		log.Println(err)
		return nil
	}

	return []*model.MetricValue{
		GaugeValue("load.1min", load.Avg1min),
		GaugeValue("load.5min", load.Avg5min),
		GaugeValue("load.15min", load.Avg15min),
	}

    func LoadAvg() (*Loadavg, error) {

	loadAvg := Loadavg{}

	data, err := file.ToTrimString("/proc/loadavg")
	if err != nil {
		return nil, err
	}

	L := strings.Fields(data)
	if loadAvg.Avg1min, err = strconv.ParseFloat(L[0], 64); err != nil {
		return nil, err
	}
	if loadAvg.Avg5min, err = strconv.ParseFloat(L[1], 64); err != nil {
		return nil, err
	}
	if loadAvg.Avg15min, err = strconv.ParseFloat(L[2], 64); err != nil {
		return nil, err
	}

	return &loadAvg, nil
}
    
func GaugeValue(metric string, val interface{}, tags ...string) *model.MetricValue {
	return NewMetricValue(metric, val, "GAUGE", tags...)
}
    
//网卡
    func NetMetrics() []*model.MetricValue {
	return CoreNetMetrics(g.Config().Collector.IfacePrefix)
}

func CoreNetMetrics(ifacePrefix []string) []*model.MetricValue {

	netIfs, err := nux.NetIfs(ifacePrefix)
	if err != nil {
		log.Println(err)
		return []*model.MetricValue{}
	}

	cnt := len(netIfs)
	ret := make([]*model.MetricValue, cnt*23)

	for idx, netIf := range netIfs {
		iface := "iface=" + netIf.Iface
		ret[idx*23+0] = CounterValue("net.if.in.bytes", netIf.InBytes, iface)
		ret[idx*23+1] = CounterValue("net.if.in.packets", netIf.InPackages, iface)
		ret[idx*23+2] = CounterValue("net.if.in.errors", netIf.InErrors, iface)
		ret[idx*23+3] = CounterValue("net.if.in.dropped", netIf.InDropped, iface)
		ret[idx*23+4] = CounterValue("net.if.in.fifo.errs", netIf.InFifoErrs, iface)
		ret[idx*23+5] = CounterValue("net.if.in.frame.errs", netIf.InFrameErrs, iface)
		ret[idx*23+6] = CounterValue("net.if.in.compressed", netIf.InCompressed, iface)
		ret[idx*23+7] = CounterValue("net.if.in.multicast", netIf.InMulticast, iface)
		ret[idx*23+8] = CounterValue("net.if.out.bytes", netIf.OutBytes, iface)
		ret[idx*23+9] = CounterValue("net.if.out.packets", netIf.OutPackages, iface)
		ret[idx*23+10] = CounterValue("net.if.out.errors", netIf.OutErrors, iface)
		ret[idx*23+11] = CounterValue("net.if.out.dropped", netIf.OutDropped, iface)
		ret[idx*23+12] = CounterValue("net.if.out.fifo.errs", netIf.OutFifoErrs, iface)
		ret[idx*23+13] = CounterValue("net.if.out.collisions", netIf.OutCollisions, iface)
		ret[idx*23+14] = CounterValue("net.if.out.carrier.errs", netIf.OutCarrierErrs, iface)
		ret[idx*23+15] = CounterValue("net.if.out.compressed", netIf.OutCompressed, iface)
		ret[idx*23+16] = CounterValue("net.if.total.bytes", netIf.TotalBytes, iface)
		ret[idx*23+17] = CounterValue("net.if.total.packets", netIf.TotalPackages, iface)
		ret[idx*23+18] = CounterValue("net.if.total.errors", netIf.TotalErrors, iface)
		ret[idx*23+19] = CounterValue("net.if.total.dropped", netIf.TotalDropped, iface)
		ret[idx*23+20] = GaugeValue("net.if.speed.bits", netIf.SpeedBits, iface)
		ret[idx*23+21] = CounterValue("net.if.in.percent", netIf.InPercent, iface)
		ret[idx*23+22] = CounterValue("net.if.out.percent", netIf.OutPercent, iface)
	}
	return ret
}
    
    func NetIfs(onlyPrefix []string) ([]*NetIf, error) {
	contents, err := ioutil.ReadFile("/proc/net/dev")
	if err != nil {
		return nil, err
	}

	ret := []*NetIf{}

	reader := bufio.NewReader(bytes.NewBuffer(contents))
	for {
		lineBytes, err := file.ReadLine(reader)
		if err == io.EOF {
			err = nil
			break
		} else if err != nil {
			return nil, err
		}

		line := string(lineBytes)
		idx := strings.Index(line, ":")
		if idx < 0 {
			continue
		}

		netIf := NetIf{}

		eth := strings.TrimSpace(line[0:idx])
		if len(onlyPrefix) > 0 {
			found := false
			for _, prefix := range onlyPrefix {
				if strings.HasPrefix(eth, prefix) {
					found = true
					break
				}
			}

			if !found {
				continue
			}
		}

		netIf.Iface = eth

		fields := strings.Fields(line[idx+1:])

		if len(fields) != 16 {
			continue
		}

		netIf.InBytes, _ = strconv.ParseInt(fields[0], 10, 64)
		netIf.InPackages, _ = strconv.ParseInt(fields[1], 10, 64)
		netIf.InErrors, _ = strconv.ParseInt(fields[2], 10, 64)
		netIf.InDropped, _ = strconv.ParseInt(fields[3], 10, 64)
		netIf.InFifoErrs, _ = strconv.ParseInt(fields[4], 10, 64)
		netIf.InFrameErrs, _ = strconv.ParseInt(fields[5], 10, 64)
		netIf.InCompressed, _ = strconv.ParseInt(fields[6], 10, 64)
		netIf.InMulticast, _ = strconv.ParseInt(fields[7], 10, 64)

		netIf.OutBytes, _ = strconv.ParseInt(fields[8], 10, 64)
		netIf.OutPackages, _ = strconv.ParseInt(fields[9], 10, 64)
		netIf.OutErrors, _ = strconv.ParseInt(fields[10], 10, 64)
		netIf.OutDropped, _ = strconv.ParseInt(fields[11], 10, 64)
		netIf.OutFifoErrs, _ = strconv.ParseInt(fields[12], 10, 64)
		netIf.OutCollisions, _ = strconv.ParseInt(fields[13], 10, 64)
		netIf.OutCarrierErrs, _ = strconv.ParseInt(fields[14], 10, 64)
		netIf.OutCompressed, _ = strconv.ParseInt(fields[15], 10, 64)

		netIf.TotalBytes = netIf.InBytes + netIf.OutBytes
		netIf.TotalPackages = netIf.InPackages + netIf.OutPackages
		netIf.TotalErrors = netIf.InErrors + netIf.OutErrors
		netIf.TotalDropped = netIf.InDropped + netIf.OutDropped

		speedFile := fmt.Sprintf("/sys/class/net/%s/speed", netIf.Iface)
		if content, err := ioutil.ReadFile(speedFile); err == nil {
			var speed int64
			speed, err = strconv.ParseInt(strings.TrimSpace(string(content)), 10, 64)
			if err != nil {
				netIf.SpeedBits = int64(0)
				netIf.InPercent = float64(0)
				netIf.OutPercent = float64(0)
			} else if speed == 0 {
				netIf.SpeedBits = int64(0)
				netIf.InPercent = float64(0)
				netIf.OutPercent = float64(0)
			} else {
				netIf.SpeedBits = speed * MILLION_BIT
				netIf.InPercent = float64(netIf.InBytes*BITS_PER_BYTE) * 100.0 / float64(netIf.SpeedBits)
				netIf.OutPercent = float64(netIf.OutBytes*BITS_PER_BYTE) * 100.0 / float64(netIf.SpeedBits)
			}
		} else {
			netIf.SpeedBits = int64(0)
			netIf.InPercent = float64(0)
			netIf.OutPercent = float64(0)
		}

		ret = append(ret, &netIf)
	}

	return ret, nil
}
```

## 需要历史点的采集

```go
go cron.InitDataHistory()

func InitDataHistory() {
	for {
		funcs.UpdateCpuStat()
		funcs.UpdateDiskStats()
		time.Sleep(g.COLLECT_INTERVAL)
	}
}

func UpdateCpuStat() error {
	ps, err := nux.CurrentProcStat()
	if err != nil {
		return err
	}

	psLock.Lock()
	defer psLock.Unlock()
	for i := historyCount - 1; i > 0; i-- {
		procStatHistory[i] = procStatHistory[i-1]
		if procStatHistory[i] != nil && procStatHistoryLen <= i {
			procStatHistoryLen = i + 1
		}
	}

	procStatHistory[0] = ps

	return nil
}

//parse cpu etc
func CurrentProcStat() (*ProcStat, error) {
	f := "/proc/stat"
	bs, err := ioutil.ReadFile(f)
	if err != nil {
		return nil, err
	}

	ps := &ProcStat{Cpus: make([]*CpuUsage, NumCpu())}
	reader := bufio.NewReader(bytes.NewBuffer(bs))

	for {
		line, err := file.ReadLine(reader)
		if err == io.EOF {
			err = nil
			break
		} else if err != nil {
			return ps, err
		}
		parseLine(line, ps)
	}

	return ps, nil
}

func parseLine(line []byte, ps *ProcStat) {
	fields := strings.Fields(string(line))
	if len(fields) < 2 {
		return
	}

	fieldName := fields[0]
	if fieldName == "cpu" {
		ps.Cpu = parseCpuFields(fields)
		return
	}

	if strings.HasPrefix(fieldName, "cpu") {
		idx, err := strconv.Atoi(fieldName[3:])
		if err != nil || idx >= len(ps.Cpus) {
			return
		}

		ps.Cpus[idx] = parseCpuFields(fields)
		return
	}

	if fieldName == "ctxt" {
		ps.Ctxt, _ = strconv.ParseUint(fields[1], 10, 64)
		return
	}

	if fieldName == "processes" {
		ps.Processes, _ = strconv.ParseUint(fields[1], 10, 64)
		return
	}

	if fieldName == "procs_running" {
		ps.ProcsRunning, _ = strconv.ParseUint(fields[1], 10, 64)
		return
	}

	if fieldName == "procs_blocked" {
		ps.ProcsBlocked, _ = strconv.ParseUint(fields[1], 10, 64)
		return
	}
}

func parseCpuFields(fields []string) *CpuUsage {
	cu := new(CpuUsage)
	sz := len(fields)
	for i := 1; i < sz; i++ {
		val, err := strconv.ParseUint(fields[i], 10, 64)
		if err != nil {
			continue
		}

		cu.Total += val
		switch i {
		case 1:
			cu.User = val
		case 2:
			cu.Nice = val
		case 3:
			cu.System = val
		case 4:
			cu.Idle = val
		case 5:
			cu.Iowait = val
		case 6:
			cu.Irq = val
		case 7:
			cu.SoftIrq = val
		case 8:
			cu.Steal = val
		case 9:
			cu.Guest = val
		}
	}
	return cu
}

//只有一个值无法算出差值
func CpuPrepared() bool {
	return procStatHistory[1] != nil
}

func CpuUser() float64 {
	dt := deltaTotal()
	if dt == 0 {
		return 0.0
	}
	invQuotient := 100.00 / float64(dt)
    //在这个时间内user占用cpu时间的百分百
	return float64(procStatHistory[0].Cpu.User-procStatHistory[procStatHistoryLen-1].Cpu.User) * invQuotient
}

func deltaTotal() uint64 {
	if procStatHistory[1] == nil {
		return 0
	}
	return procStatHistory[0].Cpu.Total - procStatHistory[procStatHistoryLen-1].Cpu.Total
}

//disk
func UpdateDiskStats() error {
	dsList, err := nux.ListDiskStats()
	if err != nil {
		return err
	}

	dsLock.Lock()
	defer dsLock.Unlock()
    //多个盘
	for i := 0; i < len(dsList); i++ {
		device := dsList[i].Device
		diskStatsMap[device] = [2]*nux.DiskStats{dsList[i], diskStatsMap[device][0]}
	}
	return nil
}
```