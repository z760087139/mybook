# Auto Dump profile

## 背景

为了实现在无人监控情况下，实现性能监控出现毛刺时能够实时进行 profile 内容导出，方便后续对毛刺实际情况进行排查。

除了 auto dump profile 以外，还需要日志记录进行配合，快速分析系统运行情况、业务执行情况和定位性能问题原因

关于日志打印时机及打印内容规范，有待后续进行研究和记录

## 参考项目

1. 普罗采集方式 github.com/prometheus/client\_golang
2. google cadvisor 项目
3. 国内开源golang项目 github.com/mosn/holmes
4. docker 项目
5. golang 性能采集工具 github.com/shirou/gopsutil

### cAdvisor

cAdvisor 在 kubelet 启动时同时启动，作为节点及容器的 cpu/ memory 的采集器，提供性能数据到普罗。因此了解 cAdvisor 如何采集容器的 cpu / memory 可以更准确根据采集内容触发 profile 采集

cAdvisor 会根据是否容器内使用不同的采集方式，最终都是通过 cgroup 内容进行分析和采集

manager/manager.go 为启动入口，采集器实现位于 resctrl/manager.go 和 resctrl/utils.go

```go
// manager/manager.go:148
// 创建资源采集器，
func New(memoryCache *memory.InMemoryCache, sysfs sysfs.SysFs, houskeepingConfig HouskeepingConfig, includedMetricsSet container.MetricSet, collectorHTTPClient *http.Client, rawContainerCgroupPathPrefixWhiteList, containerEnvMetadataWhiteList []string, perfEventsFile string, resctrlInterval time.Duration) (Manager, error) {
	if memoryCache == nil {
		return nil, fmt.Errorf("manager requires memory storage")
	}

	// Detect the container we are running on.
	selfContainer := "/"
	var err error
	// Avoid using GetOwnCgroupPath on cgroup v2 as it is not supported by libcontainer
	if !cgroups.IsCgroup2UnifiedMode() {
		selfContainer, err = cgroups.GetOwnCgroup("cpu")
		if err != nil {
			return nil, err
		}
		klog.V(2).Infof("cAdvisor running in container: %q", selfContainer)
	}

	context := fs.Context{}

	if err := container.InitializeFSContext(&context); err != nil {
		return nil, err
	}

	fsInfo, err := fs.NewFsInfo(context)
	if err != nil {
		return nil, err
	}

	// If cAdvisor was started with host's rootfs mounted, assume that its running
	// in its own namespaces.
	inHostNamespace := false
	if _, err := os.Stat("/rootfs/proc"); os.IsNotExist(err) {
		inHostNamespace = true
	}

	// Register for new subcontainers.
	eventsChannel := make(chan watcher.ContainerEvent, 16)

	newManager := &manager{
		containers:                            make(map[namespacedContainerName]*containerData),
		quitChannels:                          make([]chan error, 0, 2),
		memoryCache:                           memoryCache,
		fsInfo:                                fsInfo,
		sysFs:                                 sysfs,
		cadvisorContainer:                     selfContainer,
		inHostNamespace:                       inHostNamespace,
		startupTime:                           time.Now(),
		maxHousekeepingInterval:               *houskeepingConfig.Interval,
		allowDynamicHousekeeping:              *houskeepingConfig.AllowDynamic,
		includedMetrics:                       includedMetricsSet,
		containerWatchers:                     []watcher.ContainerWatcher{},
		eventsChannel:                         eventsChannel,
		collectorHTTPClient:                   collectorHTTPClient,
		rawContainerCgroupPathPrefixWhiteList: rawContainerCgroupPathPrefixWhiteList,
		containerEnvMetadataWhiteList:         containerEnvMetadataWhiteList,
	}

	machineInfo, err := machine.Info(sysfs, fsInfo, inHostNamespace)
	if err != nil {
		return nil, err
	}
	newManager.machineInfo = *machineInfo
	klog.V(1).Infof("Machine: %+v", newManager.machineInfo)

	newManager.perfManager, err = perf.NewManager(perfEventsFile, machineInfo.Topology)
	if err != nil {
		return nil, err
	}

	newManager.resctrlManager, err = resctrl.NewManager(resctrlInterval, resctrl.Setup, machineInfo.CPUVendorID, inHostNamespace)
	if err != nil {
		klog.V(4).Infof("Cannot gather resctrl metrics: %v", err)
	}

	versionInfo, err := getVersionInfo()
	if err != nil {
		return nil, err
	}
	klog.V(1).Infof("Version: %+v", *versionInfo)

	newManager.eventHandler = events.NewEventManager(parseEventsStoragePolicy())
	return newManager, nil
}
```

### holmes

#### 采集触发规则

* cpu / mem 使用率百分比
* cpu / mem 与历史某时间内平均值增幅百分比
* min cpu / mem 达到多少 cpu 以后再触发
* 采集 cooldown 设置，距离上次采集时间间隔是否已经足够
* max cpu 达到多少后不会再进行dump ，进行程序保护，特别是goroutine ，已知数量过渡时可能存在程序暂停的情况

#### 状态数据采集方式

**cpu 最大资源额度获取方式**

根据是否 docker 容器区分 **cpu 最大值的获取方式**，如果是 docker 的话会读取文件内容

```go
// docker cpu 配置读取方式

const(
	cgroupCpuQuotaPath  = "/sys/fs/cgroup/cpu/cpu.cfs_quota_us"
	cgroupCpuPeriodPath = "/sys/fs/cgroup/cpu/cpu.cfs_period_us"
)
// get cpu core number limited by CGroup.
func getCGroupCPUCore() (float64, error) {
	var cpuQuota uint64

	cpuPeriod, err := readUint(cgroupCpuPeriodPath)
	if cpuPeriod == 0 || err != nil {
		return 0, err
	}

	if cpuQuota, err = readUint(cgroupCpuQuotaPath); err != nil {
		return 0, err
	}

	return float64(cpuQuota) / float64(cpuPeriod), nil
}

// readUint 为 docker 项目的源码内容，用于从文件读取 cpu 信息
```

**cpu 当前使用值获取方式**

holmes 通过 gopsutil 方式获取当前程序的 cpu 使用率，具体见 gopsutil 描述

**memory 最大资源额度获取方式**

holmes 采用了 docker 文件读取并与 gopsutil 记录的虚拟内存大小比对，取小值为内存最大值

```go
const 	cgroupMemLimitPath  = "/sys/fs/cgroup/memory/memory.limit_in_bytes"

func getCGroupMemoryLimit() (uint64, error) {
	usage, err := readUint(cgroupMemLimitPath)
	if err != nil {
		return 0, err
	}
	machineMemory, err := mem_util.VirtualMemory()
	if err != nil {
		return 0, err
	}
	limit := uint64(math.Min(float64(usage), float64(machineMemory.Total)))
	return limit, nil
}
```

**memory 当前使用值获取方式**

holmes 采用 gopsutil 工具获取当前程序的 memory 使用率，并计算 RSS 部分，具体见 gopsutil 描述

**GC 模式的 memory 采样**

holmes 还允许通过 golang gc 前后的内存使用情况触发 memory 采集，主要目的是为了解决某些情况下临时申请的内存空间在普通采样检测到之前被回收，导致无法正常采样

有专门的 holmes gc 采集说明文档，内容比较复杂，还在研读中。[文档地址](https://uncledou.site/2022/go-pprof-heap/)

```go
// 对 gc 触发事件进行监听，通过 runtime.SetFinalizer 实现，并通过 chan 把事件往外传递
// holmes.go:196
func (h *Holmes) startGCCycleLoop(ch chan struct{}) {
	h.gcHeapStats = newRing(minCollectCyclesBeforeDumpStart)

	gc := &gcHeapFinalizer{
		h,
	}

	runtime.SetFinalizer(gc, finalizerCallback)

	go gc.h.gcHeapCheckLoop(ch)
}

// holmes.go:163
func finalizerCallback(gc *gcHeapFinalizer) {
	defer func() {
		if r := recover(); r != nil {
			gc.h.Errorf("Panic in finalizer callback: %v", r)
		}
	}()
	// disable or stop gc clean up normally
	if atomic.LoadInt64(&gc.h.stopped) == 1 {
		return
	}

	// register the finalizer again
	runtime.SetFinalizer(gc, finalizerCallback)

	// read channel should be atomic.
	ch := gc.h.gcEventsCh
	if ch == nil {
		return
	}
	// Notice: here may be a litte race, will panic when ch is closed now.
	// we just leave it since it is very small and there is a recover.
	select {
	case ch <- struct{}{}:
	default:
		gc.h.Errorf("can not send event to finalizer channel immediately, may be analyzer blocked?")
	}
}

// gc 采集逻辑
func (h *Holmes) gcHeapCheckLoop(ch chan struct{}) {
	for range ch {
		h.gcHeapCheckAndDump()
	}
}

func (h *Holmes) gcHeapCheckAndDump() {
	gcHeapOpts := h.opts.GetGcHeapOpts()

	if !gcHeapOpts.Enable || atomic.LoadInt64(&h.stopped) == 1 {
		return
	}

	memStats := new(runtime.MemStats)
	runtime.ReadMemStats(memStats)

	// TODO: we can only use NextGC for now since runtime haven't expose heapmarked yet
	// and we hard code the gcPercent is 100 here.
	// may introduce a new API debug.GCHeapMarked? it can also has better performance(no STW).
	nextGC := memStats.NextGC
	prevGC := nextGC / 2 //nolint:gomnd

	memoryLimit, err := h.getMemoryLimit()
	if memoryLimit == 0 || err != nil {
		h.Errorf("[Holmes] get memory limit failed, memory limit: %v, error: %v", memoryLimit, err)
		return
	}

	ratio := int(100 * float64(prevGC) / float64(memoryLimit))
	h.gcHeapStats.push(ratio)

	h.gcCycleCount++
	if h.gcCycleCount < minCollectCyclesBeforeDumpStart {
		// at least collect some cycles
		// before start to judge and dump
		h.Debugf("[Holmes] GC cycle warming up : %d", h.gcCycleCount)
		return
	}

	if h.gcHeapCoolDownTime.After(time.Now()) {
		h.Debugf("[Holmes] GC heap dump is in cooldown")
		return
	}

	if triggered := h.gcHeapProfile(ratio, h.gcHeapTriggered, gcHeapOpts); triggered {
		if h.gcHeapTriggered {
			// already dump twice, mark it false
			h.gcHeapTriggered = false
			h.gcHeapCoolDownTime = time.Now().Add(gcHeapOpts.CoolDown)
			h.gcHeapTriggerCount++
		} else {
			// force dump next time
			h.gcHeapTriggered = true
		}
	}
}
```

**goroutine 统计方式**

holmes 采用 runtime.NumGoroutine 获取当前使用的 goroutine 数量

### gopsutil

#### 获取CPU 并计算使用率百分比

gopsutil 通过 cgo 获取指定 id 的 user / system cpu 使用率，类似 top 命令中展示的内容

统计设定的时间差的 cpu，并通过 runtime 获取实际可使用的CPU数量（即最大可用CPU）该CPU无视 limits 限制，为主节点上最大可用CPU或者为程序启动设置的逻辑CPU数

得到的结果是 **指定pid进程的cpu使用率，使用率上限为 n \* 100% (n 为最大可用CPU）**

```go
// process/process.go:238
func (p *Process) PercentWithContext(ctx context.Context, interval time.Duration) (float64, error) {
	cpuTimes, err := p.TimesWithContext(ctx)
	if err != nil {
		return 0, err
	}
	now := time.Now()

	if interval > 0 {
		p.lastCPUTimes = cpuTimes
		p.lastCPUTime = now
		if err := common.Sleep(ctx, interval); err != nil {
			return 0, err
		}
		cpuTimes, err = p.TimesWithContext(ctx)
		now = time.Now()
		if err != nil {
			return 0, err
		}
	} else {
		if p.lastCPUTimes == nil {
			// invoked first time
			p.lastCPUTimes = cpuTimes
			p.lastCPUTime = now
			return 0, nil
		}
	}

	numcpu := runtime.NumCPU()
	delta := (now.Sub(p.lastCPUTime).Seconds()) * float64(numcpu)
	ret := calculatePercent(p.lastCPUTimes, cpuTimes, delta, numcpu)
	p.lastCPUTimes = cpuTimes
	p.lastCPUTime = now
	return ret, nil
}

// process/process.go:309
func calculatePercent(t1, t2 *cpu.TimesStat, delta float64, numcpu int) float64 {
	if delta == 0 {
		return 0
	}
	delta_proc := t2.Total() - t1.Total()
	overall_percent := ((delta_proc / delta) * 100) * float64(numcpu)
	return overall_percent
}

// process/process_darwin_cgo.go:188
func (p *Process) TimesWithContext(ctx context.Context) (*cpu.TimesStat, error) {
	const tiSize = C.sizeof_struct_proc_taskinfo
	ti := (*C.struct_proc_taskinfo)(C.malloc(tiSize))
	defer C.free(unsafe.Pointer(ti))

	_, err := C.proc_pidinfo(C.int(p.Pid), C.PROC_PIDTASKINFO, 0, unsafe.Pointer(ti), tiSize)
	if err != nil {
		return nil, err
	}

	ret := &cpu.TimesStat{
		CPU:    "cpu",
		User:   float64(ti.pti_total_user) * timescaleToNanoSeconds / 1e9,
		System: float64(ti.pti_total_system) * timescaleToNanoSeconds / 1e9,
	}
	return ret, nil
}
```

#### memroy 当前使用率获取方式

通过cgo 方式采集 指定的pid使用情况，包括 RSS / VMS / swap 内容， holmes 最终计算只使用了 RSS

```go
func (p *Process) MemoryInfoWithContext(ctx context.Context) (*MemoryInfoStat, error) {
	const tiSize = C.sizeof_struct_proc_taskinfo
	ti := (*C.struct_proc_taskinfo)(C.malloc(tiSize))
	defer C.free(unsafe.Pointer(ti))

	_, err := C.proc_pidinfo(C.int(p.Pid), C.PROC_PIDTASKINFO, 0, unsafe.Pointer(ti), tiSize)
	if err != nil {
		return nil, err
	}

	ret := &MemoryInfoStat{
		RSS:  uint64(ti.pti_resident_size),
		VMS:  uint64(ti.pti_virtual_size),
		Swap: uint64(ti.pti_pageins),
	}
	return ret, nil
}
```
