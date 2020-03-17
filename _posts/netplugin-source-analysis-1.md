---
title: contiv netplugin源码分析（1）
tags:
    - contiv
    - kubernetes
date: 2017-12-16
updated: 2017-12-16
---
contiv netplugin源码分析（1）

# netmaster启动过程

* 使用golang中的flag包解析命令行参数

<!-- more -->

```
func main() {
    opts := cliOpts{}
    if err := parseOpts(&opts); err != nil {
		log.Fatalf("Failed to parse cli options. Error: %s", err)
	} //定义flag
	// execute options
	execOpts(&opts) //解析flag参数

```

* 在`parseOpts`函数中，定义netmaster命令行参数:netplugin-name, cluster-store, control-url, cluster-mode等

```
func parseOpts(opts *cliOpts) error {
    flagSet = flag.NewFlagSet("netmaster", flag.ExitOnError)
    flagSet.StringVar(&opts.pluginName,  //定义plugin的名字,netplugin,
    kubernetes
        "plugin-name",
        "netplugin",
        "Plugin name used for docker v2 plugin")
    flagSet.StringVar(&opts.clusterStore,
    "cluster-store",
        "etcd://127.0.0.1:2379",
        "Etcd or Consul cluster store url.")
    flagSet.StringVar(&opts.controlURL,
        "control-url",
        defaultControlPort,
        "URL for control protocol")
    flagSet.StringVar(&opts.listenURL,
        "listen-url",
        defaultListenPort,
        "URL to listen http requests on")
    flagSet.StringVar(&opts.clusterMode,
        "cluster-mode",
        "docker",
        "{docker, kubernetes}")
    return flagSet.Parse(os.Args[1:])
    }
```
* 执行`execOpts`方法,解析命令行参数, 更新到结果提`opts`中
```

	// Validate listen and control URL options
	listenURL := strings.Split(opts.listenURL, ":")
	controlURL := strings.Split(opts.controlURL, ":")
	if len(listenURL) != 2 {
		log.Fatalf("Listen URL not in proper format. Valid format: [IP]:Port")
	}
	if len(controlURL) != 2 || (strings.Compare(controlURL[0], "0.0.0.0") == 0) {
		log.Fatalf("Control URL not in proper format. Valid format: IP:Port")
	}

	listenIP := listenURL[0]
	listenPort, err := strconv.Atoi(listenURL[1])
	if err != nil {
		log.Fatalf("Listen URL port not in valid format. Err: %+v", err)
	}
	log.Infof("Listen IP:Port %s:%d", listenIP, listenPort)

	controlIP := controlURL[0]
	controlPort, err := strconv.Atoi(controlURL[1])
	if err != nil {
		log.Fatalf("Control URL port not in valid format. Err: %+v", err)
	}
	if len(controlIP) == 0 {
		if strings.Compare(opts.controlURL, defaultControlPort) != 0 {
			// Case: If --control-url :XXXX, and :XXXX is not defaultControlPort; we error out
			log.Fatalf("Control URL not in proper format. Valid format: IP:Port")
		}
		if len(listenIP) != 0 && (strings.Compare(listenIP, "0.0.0.0") != 0) && (controlPort == listenPort) {
			// Case: --listen-url A.B.C.D:XXXX and --control-url :XXXX
			controlIP = listenIP
		} else {
			// Case: [--listen-url A.B.C.D:XXXX] --control-url defaultControlPort
			// Get the address to be used for local communication
			controlIP, err = daemon.GetLocalAddr()
			if err != nil {
				log.Fatalf("Error getting local IP address for Control URL. Err: %v", err)
			}
			controlURL[1] = listenURL[1]
		}
		opts.controlURL = controlIP + ":" + controlURL[1]
```
*   定义`MasterDaemon`对象, 并执行初始化`Init`
*   在MasterDaemon对象初始化`Init`的过程中,定义`stateDriver`,初始化资源管理.
```
    	// initialize state driver
	d.stateDriver, err = initStateDriver(d.ClusterStore)
	if err != nil {
		log.Fatalf("Failed to init state-store. Error: %s", err)
	}

	// Initialize resource manager
	d.resmgr, err = resources.NewStateResourceManager(d.stateDriver)
	if err != nil {
		log.Fatalf("Failed to init resource manager. Error: %s", err)
	}

	// Create an objdb client
	d.objdbClient, err = objdb.NewClient(d.ClusterStore)
	if err != nil {
		log.Fatalf("Error connecting to state store: %v. Err: %v", d.ClusterStore, err)
	}
```
* 在`initStateDriver`方法中,初始化`InstanceInfo`, 生成stateDriver
```

	// Setup instance info
	instInfo := core.InstanceInfo{
		DbURL: clusterStore,
	}

	return utils.NewStateDriver(stateStore, &instInfo)
}

```
* 在`NewStateDriver`函数中,根据`stateStore`参数值生成driver对象,并执行该对象的初始化操作
```
    // 生成driver的interface类型
	driver, err := initHelper(stateDriverRegistry, name)
	if err != nil {
		return nil, err
	}

	d := driver.(core.StateDriver)
	err = d.Init(instInfo) //执行driver的初始化
	if err != nil {
		return nil, err
	}

	gStateDriver = d
```
* 在`initHelper`方法中,从`stateDriverRegistry`中根据`name`获取reflect类型的driver,然后通过reflect.new操作生成driver对象,然后执行driver的`Init`方法.
```
	if _, ok := driverRegistry[driverName]; ok {
		driverType := driverRegistry[driverName].DriverType

		driver := reflect.New(driverType).Interface()
		return driver, nil

----------

	d := driver.(core.StateDriver)
	err = d.Init(instInfo)
	if err != nil {
		return nil, err
	}
```
* 当`name`值为**etcd**时, `d`为EtcdStateDriver类型的对象,在其`Init`方法中生成`Client`和`keysAPI`对象.
```

	d.Client, err = client.New(etcdConfig)
	if err != nil {
		log.Fatalf("Error creating etcd client. Err: %v", err)
	}

	// Create keys api
	d.KeysAPI = client.NewKeysAPI(d.Client)

```
* 在`MasterDaemon`的`Init`方法中,还会调用`NewStateResourceManager`方法,初始化资源管理器, 并且在`NewStateResourceManager`方法中,初始化全局变量`gStateResourceManager`并执行初始化.
```
// NewStateResourceManager instantiates a state based resource manager
func NewStateResourceManager(sd core.StateDriver) (*StateResourceManager, error) {
	if gStateResourceManager != nil {
		return nil, core.Errorf("state-based resource manager instance already exists.")
	}

	gStateResourceManager = &StateResourceManager{stateDriver: sd}
	err := gStateResourceManager.Init()
	if err != nil {
		return nil, err
	}

	return gStateResourceManager, nil

```
* 在`MasterDaemon`的`Init`方法中,还会调用`NewClient`方法生成objdb的client对象.`MasterDaemon`对象的初始化结束.
```
	// Create an objdb client
	d.objdbClient, err = objdb.NewClient(d.ClusterStore)

```
* 在`netmaster`中的main函数中,接着调用`MasterDaemon`对象的方法`RunMasterFsm`
* 在`netmaster`支持主备的方式运行,通过etcd或者consul中的*/contiv.io/lock/**键值确定主备状态,通过etcd或者consul控制netmaster的主备切换.
* 在`RunMasterFsm`中先初始化`NewOfnetMaster`对象,用来接收`netplugin`组件的rpc消息.
```
// RunMasterFsm runs netmaster FSM
func (d *MasterDaemon) RunMasterFsm() {

	// create new ofnet master
	d.ofnetMaster = ofnet.NewOfnetMaster(masterIP, ofnet.OFNET_MASTER_PORT)
	if d.ofnetMaster == nil {
		log.Fatalf("Error creating ofnet master")
	}

----------
// Create new Ofnet master
func NewOfnetMaster(myAddr string, portNo uint16) *OfnetMaster {
	// Create the master
	master := new(OfnetMaster)

	// Init params
	master.myAddr = myAddr
	master.myPort = portNo
	master.agentDb = make(map[string]*OfnetNode)
	master.endpointDb = make(map[string]*OfnetEndpoint)
	master.policyDb = make(map[string]*OfnetPolicyRule)

	// Create a new RPC server //定义master对象,包括端口号,地址等
	master.rpcServer, master.rpcListener = rpcHub.NewRpcServer(portNo)

	// Register RPC handler //向rpcServer注册,接收rpc消息,调用master的相关方法
	err := master.rpcServer.Register(master)
	if err != nil {
		log.Fatalf("Error Registering RPC callbacks. Err: %v", err)
		return nil
	}

	return master
}

```
* 启动一个协程agentDiscoveryLoop,查看etcd或者consul中各个netplugin注册的service,并通知各个agent的master节点的信息
```
// Find all netplugin nodes and add them to ofnet master
func (d *MasterDaemon) agentDiscoveryLoop() {

	// Create channels for watch thread
	agentEventCh := make(chan objdb.WatchServiceEvent, 1)
	watchStopCh := make(chan bool, 1)

	// Start a watch on netplugin service //监听/contiv.io/service/netplugin目录的变化,并更新service的信息至channel中
	err := d.objdbClient.WatchService("netplugin", agentEventCh, watchStopCh)
	if err != nil {
		log.Fatalf("Could not start a watch on netplugin service. Err: %v", err)
	}
    //从channel读取service信息,根据netplugin目录的变化,更新netplugin节点,对于ADD类型,向netplugin发送rpc消息,增加master节点,对于DELETE类型,删除内存的netplugin信息.
	for {
		agentEv := <-agentEventCh
		log.Debugf("Received netplugin watch event: %+v", agentEv)
		// build host info
		nodeInfo := ofnet.OfnetNode{
			HostAddr: agentEv.ServiceInfo.HostAddr,
			HostPort: uint16(agentEv.ServiceInfo.Port),
		}

		if agentEv.EventType == objdb.WatchServiceEventAdd {
			err = d.ofnetMaster.AddNode(nodeInfo)
			if err != nil {
				log.Errorf("Error adding node %v. Err: %v", nodeInfo, err)
			}
		} else if agentEv.EventType == objdb.WatchServiceEventDel {
			var res bool
			log.Infof("Unregister node %+v", nodeInfo)
			d.ofnetMaster.UnRegisterNode(&nodeInfo, &res)
		}

```
* 在`RunMasterFsm`方法中,在etcd或者consul中定义一个lock:*/contiv.io/lock/netmaster/leader*, 对于多个netmaster进程来说,获取到该lock的netmaster为主进程,其他为备进程
```
	// Create the lock
	leaderLock, err = d.objdbClient.NewLock("netmaster/leader", masterIP+":"+masterPort, leaderLockTTL)
	if err != nil {
		log.Fatalf("Could not create leader lock. Err: %v", err)
	}

	// Try to acquire the lock
	err = leaderLock.Acquire(0)
	if err != nil {
		// We dont expect any error during acquire.
		log.Fatalf("Error while acquiring lock. Err: %v", err)
	}

	// Initialize the stop channel
	d.stopLeaderChan = make(chan bool, 1)
	d.stopFollowerChan = make(chan bool, 1)

	// set current state
	d.currState = "follower"

	// Start off being a follower
	go d.runFollower()

	// Main run loop waiting on leader lock
	for {
		// Wait for lock events
		select {
		case event := <-leaderLock.EventChan():
			if event.EventType == objdb.LockAcquired {
				log.Infof("Leader lock acquired")

				d.becomeLeader()
			} else if event.EventType == objdb.LockLost {
				log.Infof("Leader lock lost. Becoming follower")

				d.becomeFollower()
			}
		}
	}
```
* 在runFollower方法中, 向etcd中更新service状态, 启动web server
![s1.png](https://i.loli.net/2020/03/17/1p4qjoBvczGTW2k.png)
* 当netmaster进程获取到lock: contiv.io/lock/netmaster/leader时, netmaster进程切换为Leader, 反之切换为Follower
![s2.png](https://i.loli.net/2020/03/17/3XMdO26b4hVeW19.png)
*在runLeader方法中,定义apiConroller,注册api的http routes, 从etcd中恢复servicelb的状态, 向etcd中更新service, 初始化策略管理器, 建立http routes, 启动web server等
![s3.png](https://i.loli.net/2020/03/17/nXFDJOlT6kexAKH.png)
* 当netmaster进程失去lock时,则netmaster进程切换为follower状态,在runFollower方法中, 向etcd更新service状态, 启动web server
![s4.png](https://i.loli.net/2020/03/17/Wu8I3AnrULdmqBC.png)
