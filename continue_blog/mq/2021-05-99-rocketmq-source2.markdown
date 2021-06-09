---
layout:     post
title:      "重看RocketMQ源码(二)"
date:       2021-05-10
author:     "ZhouJ000"
header-img: "img/in-post/2021/post-bg-2021-headbg.jpg"
catalog: true
tags:
    - mq
--- 


# Broker

## 启动

Broker是通过`mqbroker`这种脚本命令来启动的，最终脚本里一定会启动一个JVM进程，jvm进程里会执行一个main class的代码，也就是`BrokerStartup.main()`

```java
public static void main(String[] args) {
	start(createBrokerController(args));
}

public static BrokerController start(BrokerController controller) {
	try {
		controller.start();
		// log 略...
		return controller;
	} catch (Throwable e) {
		e.printStackTrace();
		System.exit(-1);
	}
	return null;
}
```
那就要看下BrokerController的创建：
```java
public static BrokerController createBrokerController(String[] args) {
	// ...
	try {
		//PackageConflictDetect.detectFastjson();
		// 解析命令行参数
		Options options = ServerUtil.buildCommandlineOptions(new Options());
		commandLine = ServerUtil.parseCmdLine("mqbroker", args, buildCommandlineOptions(options), new PosixParser());
		if (null == commandLine) {
			System.exit(-1);
		}

		final BrokerConfig brokerConfig = new BrokerConfig();
		// netty服务器配置，与生产者通信
		final NettyServerConfig nettyServerConfig = new NettyServerConfig();
		// netty客户端配置，与NameSever通信
		final NettyClientConfig nettyClientConfig = new NettyClientConfig();

		nettyClientConfig.setUseTLS(Boolean.parseBoolean(System.getProperty(TLS_ENABLE,
			String.valueOf(TlsSystemConfig.tlsMode == TlsMode.ENFORCING))));
		nettyServerConfig.setListenPort(10911);
		final MessageStoreConfig messageStoreConfig = new MessageStoreConfig();

		// 从节点
		if (BrokerRole.SLAVE == messageStoreConfig.getBrokerRole()) {
			int ratio = messageStoreConfig.getAccessMessageInMemoryMaxRatio() - 10;
			messageStoreConfig.setAccessMessageInMemoryMaxRatio(ratio);
		}
		// 解析命令行中 -c 的参数
		// 略...
		
		// 获取nameSever地址
		String namesrvAddr = brokerConfig.getNamesrvAddr();
		if (null != namesrvAddr) {
			try {
				String[] addrArray = namesrvAddr.split(";");
				for (String addr : addrArray) {
					// 设置地址
					RemotingUtil.string2SocketAddress(addr);
				}
			} catch (Exception e) {
				System.out.printf(
					"The Name Server Address[%s] illegal, please set it as follows, \"127.0.0.1:9876;192.168.0.1:9876\"%n",
					namesrvAddr);
				System.exit(-3);
			}
		}
		
		// 主从设置
		switch (messageStoreConfig.getBrokerRole()) {
			case ASYNC_MASTER:
			case SYNC_MASTER:
				brokerConfig.setBrokerId(MixAll.MASTER_ID);
				break;
			case SLAVE:
				if (brokerConfig.getBrokerId() <= 0) {
					System.out.printf("Slave's brokerId must be > 0");
					System.exit(-3);
				}
				break;
			default:
				break;
		}

		// 解析其他命令行参数等
		// 略...

		final BrokerController controller = new BrokerController(
			brokerConfig,
			nettyServerConfig,
			nettyClientConfig,
			messageStoreConfig);
		// remember all configs to prevent discard
		controller.getConfiguration().registerConfig(properties);

		// 初始化
		boolean initResult = controller.initialize();
		if (!initResult) {
			controller.shutdown();
			System.exit(-3);
		}

		// jvm关闭时的勾子函数
		Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
			private volatile boolean hasShutdown = false;
			private AtomicInteger shutdownTimes = new AtomicInteger(0);

			@Override
			public void run() {
				synchronized (this) {
					log.info("Shutdown hook was invoked, {}", this.shutdownTimes.incrementAndGet());
					if (!this.hasShutdown) {
						this.hasShutdown = true;
						long beginTime = System.currentTimeMillis();
						controller.shutdown();
						long consumingTimeTotal = System.currentTimeMillis() - beginTime;
						log.info("Shutdown hook over, consuming total time(ms): {}", consumingTimeTotal);
					}
				}
			}
		}, "ShutdownHook"));
		return controller;
	} catch (Throwable e) {
		e.printStackTrace();
		System.exit(-1);
	}
	return null;
}
```
初始化：
```java
public boolean initialize() throws CloneNotSupportedException {
	// 从磁盘加载配置文件
	boolean result = this.topicConfigManager.load();
	result = result && this.consumerOffsetManager.load();
	result = result && this.subscriptionGroupManager.load();
	result = result && this.consumerFilterManager.load();

	if (result) {
		try {
			// 创建消息存储管理组件
			this.messageStore =
				new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener,
					this.brokerConfig);
			// 是否开启dleger技术
			if (messageStoreConfig.isEnableDLegerCommitLog()) {
				DLedgerRoleChangeHandler roleChangeHandler = new DLedgerRoleChangeHandler(this, (DefaultMessageStore) messageStore);
				((DLedgerCommitLog)((DefaultMessageStore) messageStore).getCommitLog()).getdLedgerServer().getdLedgerLeaderElector().addRoleChangeHandler(roleChangeHandler);
			}
			// broker的统计组件
			this.brokerStats = new BrokerStats((DefaultMessageStore) this.messageStore);
			//load plugin
			MessageStorePluginContext context = new MessageStorePluginContext(messageStoreConfig, brokerStatsManager, messageArrivingListener, brokerConfig);
			this.messageStore = MessageStoreFactory.build(context, this.messageStore);
			this.messageStore.getDispatcherList().addFirst(new CommitLogDispatcherCalcBitMap(this.brokerConfig, this.consumerFilterManager));
		} catch (IOException e) {
			result = false;
			log.error("Failed to initialize", e);
		}
	}

	result = result && this.messageStore.load();

	if (result) {
		// 构建netty服务端
		this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.clientHousekeepingService);
		NettyServerConfig fastConfig = (NettyServerConfig) this.nettyServerConfig.clone();
		fastConfig.setListenPort(nettyServerConfig.getListenPort() - 2);
		this.fastRemotingServer = new NettyRemotingServer(fastConfig, this.clientHousekeepingService);
		// 创建各种线程池
		this.sendMessageExecutor = new BrokerFixedThreadPoolExecutor(
			this.brokerConfig.getSendMessageThreadPoolNums(),
			this.brokerConfig.getSendMessageThreadPoolNums(),
			1000 * 60,
			TimeUnit.MILLISECONDS,
			this.sendThreadPoolQueue,
			new ThreadFactoryImpl("SendMessageThread_"));
		
		// pull消息的线程池
		this.pullMessageExecutor = new BrokerFixedThreadPoolExecutor(
			this.brokerConfig.getPullMessageThreadPoolNums(),
			this.brokerConfig.getPullMessageThreadPoolNums(),
			1000 * 60,
			TimeUnit.MILLISECONDS,
			this.pullThreadPoolQueue,
			new ThreadFactoryImpl("PullMessageThread_"));

		// 略...
			
		// 心跳处理的线程池
		this.heartbeatExecutor = new BrokerFixedThreadPoolExecutor(
			this.brokerConfig.getHeartbeatThreadPoolNums(),
			this.brokerConfig.getHeartbeatThreadPoolNums(),
			1000 * 60,
			TimeUnit.MILLISECONDS,
			this.heartbeatThreadPoolQueue,
			new ThreadFactoryImpl("HeartbeatThread_", true));

		this.consumerManageExecutor =
			Executors.newFixedThreadPool(this.brokerConfig.getConsumerManageThreadPoolNums(), new ThreadFactoryImpl(
				"ConsumerManageThread_"));

		this.registerProcessor();
		
		// 各种后台定时任务
		final long initialDelay = UtilAll.computeNextMorningTimeMillis() - System.currentTimeMillis();
		final long period = 1000 * 60 * 60 * 24;
		// 检查broker的状态
		this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
			@Override
			public void run() {
				try {
					BrokerController.this.getBrokerStats().record();
				} catch (Throwable e) {
					log.error("schedule record error.", e);
				}
			}
		}, initialDelay, period, TimeUnit.MILLISECONDS);
		
		// consumerOffset
		this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
			@Override
			public void run() {
				try {
					BrokerController.this.consumerOffsetManager.persist();
				} catch (Throwable e) {
					log.error("schedule persist consumerOffset error.", e);
				}
			}
		}, 1000 * 10, this.brokerConfig.getFlushConsumerOffsetInterval(), TimeUnit.MILLISECONDS);
		
		// 略...

		// 设置nameSever地址列表
		if (this.brokerConfig.getNamesrvAddr() != null) {
			this.brokerOuterAPI.updateNameServerAddressList(this.brokerConfig.getNamesrvAddr());
			log.info("Set user specified name server address: {}", this.brokerConfig.getNamesrvAddr());
		} else if (this.brokerConfig.isFetchNamesrvAddrByAddressServer()) {
			this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
				// 支持通过请求加载namesever地址
				@Override
				public void run() {
					try {
						BrokerController.this.brokerOuterAPI.fetchNameServerAddr();
					} catch (Throwable e) {
						log.error("ScheduledTask fetchNameServerAddr exception", e);
					}
				}
			}, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
		}
		// dleger技术
		if (!messageStoreConfig.isEnableDLegerCommitLog()) {
			if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
				if (this.messageStoreConfig.getHaMasterAddress() != null && this.messageStoreConfig.getHaMasterAddress().length() >= 6) {
					this.messageStore.updateHaMasterAddress(this.messageStoreConfig.getHaMasterAddress());
					this.updateMasterHAServerAddrPeriodically = false;
				} else {
					this.updateMasterHAServerAddrPeriodically = true;
				}
			} else {
				this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
					@Override
					public void run() {
						try {
							BrokerController.this.printMasterAndSlaveDiff();
						} catch (Throwable e) {
							log.error("schedule printMasterAndSlaveDiff error.", e);
						}
					}
				}, 1000 * 10, 1000 * 60, TimeUnit.MILLISECONDS);
			}
		}

		// 略...
		// 事务
		initialTransaction();
		initialAcl();
		initialRpcHooks();
	}
	return result;
}
```
总结而言做了这几个事：  
1、首先从磁盘load配置文件  
2、创建消息存储组件DefaultMessageStore  
3、构建netty服务器  
4、创建各种线程池(接收消息、心跳检测等)  
5、创建各种后台定时任务  
初始化后看下BrokerController的启动：
```java
public void start() throws Exception {
	// 启动消息存储组件
	if (this.messageStore != null) {
		this.messageStore.start();
	}
	// 启动netty服务器
	if (this.remotingServer != null) {
		this.remotingServer.start();
	}
	if (this.fastRemotingServer != null) {
		this.fastRemotingServer.start();
	}
	if (this.fileWatchService != null) {
		this.fileWatchService.start();
	}
	// 对外通信组件，例如给namesever发心跳
	if (this.brokerOuterAPI != null) {
		this.brokerOuterAPI.start();
	}
	if (this.pullRequestHoldService != null) {
		this.pullRequestHoldService.start();
	}
	if (this.clientHousekeepingService != null) {
		this.clientHousekeepingService.start();
	}
	if (this.filterServerManager != null) {
		this.filterServerManager.start();
	}
	if (!messageStoreConfig.isEnableDLegerCommitLog()) {
		startProcessorByHa(messageStoreConfig.getBrokerRole());
		handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
		this.registerBrokerAll(true, false, true);
	}
	// 定时任务：去namesever中注册，即是注册也是心跳，30s一次
	this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

		@Override
		public void run() {
			try {
				BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
			} catch (Throwable e) {
				log.error("registerBrokerAll Exception", e);
			}
		}
	}, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);

	if (this.brokerStatsManager != null) {
		this.brokerStatsManager.start();
	}
	if (this.brokerFastFailure != null) {
		this.brokerFastFailure.start();
	}
}
```
![broker-start](broker-start.png)

## 注册NameSever

接上面启动时候的registerBrokerAll方法，里面做了去NameSever注册
```java
public synchronized void registerBrokerAll(final boolean checkOrderConfig, boolean oneway, boolean forceRegister) {
	TopicConfigSerializeWrapper topicConfigWrapper = this.getTopicConfigManager().buildTopicConfigSerializeWrapper();
	// 封装topic信息
	if (!PermName.isWriteable(this.getBrokerConfig().getBrokerPermission())
		|| !PermName.isReadable(this.getBrokerConfig().getBrokerPermission())) {
		ConcurrentHashMap<String, TopicConfig> topicConfigTable = new ConcurrentHashMap<String, TopicConfig>();
		for (TopicConfig topicConfig : topicConfigWrapper.getTopicConfigTable().values()) {
			TopicConfig tmp =
				new TopicConfig(topicConfig.getTopicName(), topicConfig.getReadQueueNums(), topicConfig.getWriteQueueNums(),
					this.brokerConfig.getBrokerPermission());
			topicConfigTable.put(topicConfig.getTopicName(), tmp);
		}
		topicConfigWrapper.setTopicConfigTable(topicConfigTable);
	}
	// 判断是否需要注册
	if (forceRegister || needRegister(this.brokerConfig.getBrokerClusterName(),
		this.getBrokerAddr(),
		this.brokerConfig.getBrokerName(),
		this.brokerConfig.getBrokerId(),
		this.brokerConfig.getRegisterBrokerTimeoutMills())) {
		// 注册
		doRegisterBrokerAll(checkOrderConfig, oneway, topicConfigWrapper);
	}
}

private void doRegisterBrokerAll(boolean checkOrderConfig, boolean oneway,
	TopicConfigSerializeWrapper topicConfigWrapper) {
	// 把自己注册给所有Broker
	List<RegisterBrokerResult> registerBrokerResultList = this.brokerOuterAPI.registerBrokerAll(
		this.brokerConfig.getBrokerClusterName(),
		this.getBrokerAddr(),
		this.brokerConfig.getBrokerName(),
		this.brokerConfig.getBrokerId(),
		this.getHAServerAddr(),
		topicConfigWrapper,
		this.filterServerManager.buildNewFilterServerList(),
		oneway,
		this.brokerConfig.getRegisterBrokerTimeoutMills(),
		this.brokerConfig.isCompressedRegister());
	// 如果注册结果大于0
	if (registerBrokerResultList.size() > 0) {
		RegisterBrokerResult registerBrokerResult = registerBrokerResultList.get(0);
		if (registerBrokerResult != null) {
			if (this.updateMasterHAServerAddrPeriodically && registerBrokerResult.getHaServerAddr() != null) {
				this.messageStore.updateHaMasterAddress(registerBrokerResult.getHaServerAddr());
			}
			this.slaveSynchronize.setMasterAddr(registerBrokerResult.getMasterAddr());
			if (checkOrderConfig) {
				this.getTopicConfigManager().updateOrderTopicConfig(registerBrokerResult.getKvTable());
			}
		}
	}
}
```
BrokerOuterAPI的组件方法
```java
public List<RegisterBrokerResult> registerBrokerAll(
	final String clusterName,
	final String brokerAddr,
	final String brokerName,
	final long brokerId,
	final String haServerAddr,
	final TopicConfigSerializeWrapper topicConfigWrapper,
	final List<String> filterServerList,
	final boolean oneway,
	final int timeoutMills,
	final boolean compressed) {

	final List<RegisterBrokerResult> registerBrokerResultList = new CopyOnWriteArrayList<>();
	// 获取namesever地址
	List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
	if (nameServerAddressList != null && nameServerAddressList.size() > 0) {
		// 请求头，把broker的信息存入请求头
		final RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
		requestHeader.setBrokerAddr(brokerAddr);
		requestHeader.setBrokerId(brokerId);
		requestHeader.setBrokerName(brokerName);
		requestHeader.setClusterName(clusterName);
		requestHeader.setHaServerAddr(haServerAddr);
		requestHeader.setCompressed(compressed);
		// 请求体
		RegisterBrokerBody requestBody = new RegisterBrokerBody();
		requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
		requestBody.setFilterServerList(filterServerList);
		final byte[] body = requestBody.encode(compressed);
		final int bodyCrc32 = UtilAll.crc32(body);
		requestHeader.setBodyCrc32(bodyCrc32);
		// 所有nameSever都注册成功了再返回
		final CountDownLatch countDownLatch = new CountDownLatch(nameServerAddressList.size());
		for (final String namesrvAddr : nameServerAddressList) {
			brokerOuterExecutor.execute(new Runnable() {
				@Override
				public void run() {
					try {
						// 注册
						RegisterBrokerResult result = registerBroker(namesrvAddr,oneway, timeoutMills,requestHeader,body);
						if (result != null) {
							registerBrokerResultList.add(result);
						}
						log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
					} catch (Exception e) {
						log.warn("registerBroker Exception, {}", namesrvAddr, e);
					} finally {
						countDownLatch.countDown();
					}
				}
			});
		}
		try {
			countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
		} catch (InterruptedException e) {
		}
	}
	return registerBrokerResultList;
}

private RegisterBrokerResult registerBroker(
	final String namesrvAddr,
	final boolean oneway,
	final int timeoutMills,
	final RegisterBrokerRequestHeader requestHeader,
	final byte[] body
) throws RemotingCommandException, MQBrokerException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException, InterruptedException {
	
	// 封装请求
	RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.REGISTER_BROKER, requestHeader);
	request.setBody(body);
	// 不用等待注册结果就返回
	if (oneway) {
		try {
			this.remotingClient.invokeOneway(namesrvAddr, request, timeoutMills);
		} catch (RemotingTooMuchRequestException e) {
			// Ignore
		}
		return null;
	}
	// 通过nettyClient发送请求：大致通过netty.bootstrap.connect建立连接，netty.channel.writeAndFlush请求
	RemotingCommand response = this.remotingClient.invokeSync(namesrvAddr, request, timeoutMills);
	assert response != null;
	// 处理返回结果
	switch (response.getCode()) {
		case ResponseCode.SUCCESS: {
			RegisterBrokerResponseHeader responseHeader = (RegisterBrokerResponseHeader) response.decodeCommandCustomHeader(RegisterBrokerResponseHeader.class);
			RegisterBrokerResult result = new RegisterBrokerResult();
			result.setMasterAddr(responseHeader.getMasterAddr());
			result.setHaServerAddr(responseHeader.getHaServerAddr());
			if (response.getBody() != null) {
				result.setKvTable(KVTable.decode(response.getBody(), KVTable.class));
			}
			return result;
		}
		default:
			break;
	}
	throw new MQBrokerException(response.getCode(), response.getRemark(), requestHeader == null ? null : requestHeader.getBrokerAddr());
}
```

## 接收消息

Broker的消息持久化依赖于两个文件CommitLog和ConsumeQueue。当Broker收到一条消息后，首先会把该消息写入磁盘文件CommitLog，顺序写入。CommitLog是很多磁盘文件，每个文件最多1GB，当一个文件写满之后，就新建一个
![cml](cml.png)

消息已经持久化在了磁盘上，消费者要消费一条消息时，怎么知道从CommitLog中具体获取哪个消息呢？这时就用到另一个磁盘文件ConsumeQueue，在Broker中，每个MessageQueue都有一系列ConsumeQueue文件，如：`$HOME/store/consumequeue/{topic}/{queueid}/{filename}`。queueid就是对应MessageQueue，这个ConsumeQueue文件存储的就是一条消息在CommitLog中的偏移量。即当Broker收到一条消息后，会把消息在CommitLog中的物理位置，也就是一个文件偏移量，记录在对应的MessageQueue的ConsumeQueue文件中

```java
// NettyRemotingAbstract
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
	final Pair<NettyRequestProcessor, ExecutorService> matched = this.processorTable.get(cmd.getCode());
	final Pair<NettyRequestProcessor, ExecutorService> pair = null == matched ? this.defaultRequestProcessor : matched;
	final int opaque = cmd.getOpaque();
	
	if (pair != null) {
		Runnable run = new Runnable() {
			@Override
			public void run() {
				try {
					doBeforeRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd);
					// 回调
					final RemotingResponseCallback callback = new RemotingResponseCallback() {
						@Override
						public void callback(RemotingCommand response) {
							doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
							if (!cmd.isOnewayRPC()) {
								if (response != null) {
									response.setOpaque(opaque);
									response.markResponseType();
									try {
										ctx.writeAndFlush(response);
									} catch (Throwable e) {
										log.error("process request over, but response failed", e);
										log.error(cmd.toString());
										log.error(response.toString());
									}
								} else {
								}
							}
						}
					};
					// 异步
					if (pair.getObject1() instanceof AsyncNettyRequestProcessor) {
						AsyncNettyRequestProcessor processor = (AsyncNettyRequestProcessor)pair.getObject1();
						processor.asyncProcessRequest(ctx, cmd, callback);
					} else {
						NettyRequestProcessor processor = pair.getObject1();
						RemotingCommand response = processor.processRequest(ctx, cmd);
						// 同步
						callback.callback(response);
					}
				} catch (Throwable e) {
					log.error("process request exception", e);
					log.error(cmd.toString());

					if (!cmd.isOnewayRPC()) {
						final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_ERROR,
							RemotingHelper.exceptionSimpleDesc(e));
						response.setOpaque(opaque);
						ctx.writeAndFlush(response);
					}
				}
			}
		};

		if (pair.getObject1().rejectRequest()) {
			final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
				"[REJECTREQUEST]system busy, start flow control for a while");
			response.setOpaque(opaque);
			ctx.writeAndFlush(response);
			return;
		}

		try {
			final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
			// 线程池提交
			pair.getObject2().submit(requestTask);
		} catch (RejectedExecutionException e) {
			if ((System.currentTimeMillis() % 10000) == 0) {
				log.warn(RemotingHelper.parseChannelRemoteAddr(ctx.channel())
					+ ", too many requests and system thread pool busy, RejectedExecutionException "
					+ pair.getObject2().toString()
					+ " request code: " + cmd.getCode());
			}

			if (!cmd.isOnewayRPC()) {
				final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
					"[OVERLOAD]system busy, start flow control for a while");
				response.setOpaque(opaque);
				ctx.writeAndFlush(response);
			}
		}
	} else {
		String error = " request type " + cmd.getCode() + " not supported";
		final RemotingCommand response =
			RemotingCommand.createResponseCommand(RemotingSysResponseCode.REQUEST_CODE_NOT_SUPPORTED, error);
		response.setOpaque(opaque);
		ctx.writeAndFlush(response);
		log.error(RemotingHelper.parseChannelRemoteAddr(ctx.channel()) + error);
	}
}
```
同步或异步都是到SendMessageProcessor
```java
// SendMessageProcessor
public void asyncProcessRequest(ChannelHandlerContext ctx, RemotingCommand request, RemotingResponseCallback responseCallback) throws Exception {
	asyncProcessRequest(ctx, request).thenAcceptAsync(responseCallback::callback, this.brokerController.getSendMessageExecutor());
}

public CompletableFuture<RemotingCommand> asyncProcessRequest(ChannelHandlerContext ctx,
															  RemotingCommand request) throws RemotingCommandException {
	final SendMessageContext mqtraceContext;
	switch (request.getCode()) {
		case RequestCode.CONSUMER_SEND_MSG_BACK:
			return this.asyncConsumerSendMsgBack(ctx, request);
		default:
			SendMessageRequestHeader requestHeader = parseRequestHeader(request);
			if (requestHeader == null) {
				return CompletableFuture.completedFuture(null);
			}
			mqtraceContext = buildMsgContext(ctx, requestHeader);
			this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
			if (requestHeader.isBatch()) {
				return this.asyncSendBatchMessage(ctx, request, mqtraceContext, requestHeader);
			} else {
				return this.asyncSendMessage(ctx, request, mqtraceContext, requestHeader);
			}
	}
}
```
通过`this.brokerController.getMessageStore().asyncPutMessage(msgInner)`MessageStore去处理
```java
public CompletableFuture<PutMessageResult> asyncPutMessage(MessageExtBrokerInner msg) {
	PutMessageStatus checkStoreStatus = this.checkStoreStatus();
	if (checkStoreStatus != PutMessageStatus.PUT_OK) {
		return CompletableFuture.completedFuture(new PutMessageResult(checkStoreStatus, null));
	}

	PutMessageStatus msgCheckStatus = this.checkMessage(msg);
	if (msgCheckStatus == PutMessageStatus.MESSAGE_ILLEGAL) {
		return CompletableFuture.completedFuture(new PutMessageResult(msgCheckStatus, null));
	}

	long beginTime = this.getSystemClock().now();
	CompletableFuture<PutMessageResult> putResultFuture = this.commitLog.asyncPutMessage(msg);

	putResultFuture.thenAccept((result) -> {
		long elapsedTime = this.getSystemClock().now() - beginTime;
		if (elapsedTime > 500) {
			log.warn("putMessage not in lock elapsed time(ms)={}, bodyLength={}", elapsedTime, msg.getBody().length);
		}
		this.storeStatsService.setPutMessageEntireTimeMax(elapsedTime);

		if (null == result || !result.isOk()) {
			this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
		}
	});

	return putResultFuture;
}
```
CommitLog去保存消息
```java
public CompletableFuture<PutMessageResult> asyncPutMessage(final MessageExtBrokerInner msg) {
	// Set the storage time
	msg.setStoreTimestamp(System.currentTimeMillis());
	// Set the message body BODY CRC (consider the most appropriate setting
	// on the client)
	msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
	// Back to Results
	AppendMessageResult result = null;

	StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

	String topic = msg.getTopic();
	int queueId = msg.getQueueId();

	final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
	if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
			|| tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
		// Delay Delivery
		if (msg.getDelayTimeLevel() > 0) {
			if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
				msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
			}

			topic = TopicValidator.RMQ_SYS_SCHEDULE_TOPIC;
			queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

			// Backup real topic, queueId
			MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
			MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
			msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

			msg.setTopic(topic);
			msg.setQueueId(queueId);
		}
	}

	long elapsedTimeInLock = 0;
	MappedFile unlockMappedFile = null;
	MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();

	putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
	try {
		long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
		this.beginTimeInLock = beginLockTimestamp;

		// Here settings are stored timestamp, in order to ensure an orderly
		// global
		msg.setStoreTimestamp(beginLockTimestamp);

		if (null == mappedFile || mappedFile.isFull()) {
			mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
		}
		if (null == mappedFile) {
			log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
			beginTimeInLock = 0;
			return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null));
		}

		result = mappedFile.appendMessage(msg, this.appendMessageCallback);
		switch (result.getStatus()) {
			case PUT_OK:
				break;
			case END_OF_FILE:
				unlockMappedFile = mappedFile;
				// Create a new file, re-write the message
				mappedFile = this.mappedFileQueue.getLastMappedFile(0);
				if (null == mappedFile) {
					// XXX: warn and notify me
					log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
					beginTimeInLock = 0;
					return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result));
				}
				result = mappedFile.appendMessage(msg, this.appendMessageCallback);
				break;
			case MESSAGE_SIZE_EXCEEDED:
			case PROPERTIES_SIZE_EXCEEDED:
				beginTimeInLock = 0;
				return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result));
			case UNKNOWN_ERROR:
				beginTimeInLock = 0;
				return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result));
			default:
				beginTimeInLock = 0;
				return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result));
		}

		elapsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
		beginTimeInLock = 0;
	} finally {
		putMessageLock.unlock();
	}

	if (elapsedTimeInLock > 500) {
		log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", elapsedTimeInLock, msg.getBody().length, result);
	}

	if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
		this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
	}

	PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

	// Statistics
	storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();
	storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());

	CompletableFuture<PutMessageStatus> flushResultFuture = submitFlushRequest(result, msg);
	CompletableFuture<PutMessageStatus> replicaResultFuture = submitReplicaRequest(result, msg);
	return flushResultFuture.thenCombine(replicaResultFuture, (flushStatus, replicaStatus) -> {
		if (flushStatus != PutMessageStatus.PUT_OK) {
			putMessageResult.setPutMessageStatus(flushStatus);
		}
		if (replicaStatus != PutMessageStatus.PUT_OK) {
			putMessageResult.setPutMessageStatus(replicaStatus);
			if (replicaStatus == PutMessageStatus.FLUSH_SLAVE_TIMEOUT) {
				log.error("do sync transfer other node, wait return, but failed, topic: {} tags: {} client address: {}",
						msg.getTopic(), msg.getTags(), msg.getBornHostNameString());
			}
		}
		return putMessageResult;
	});
}
```



https://cloud.tencent.com/developer/article/1645897


https://cloud.tencent.com/developer/column/80213



# Producer

MessageQueue是RocketMq的一种**数据分片**+**物理存储**机制

![msgqueue1](msgqueue1.png)

一般在创建Topic的时候会指定MessageQueue的数量。如上图中，一个Topic中有4个MessageQueue，每个Brokers上有2个MessageQueue，生产者通过算法(默认是均匀分配//负载均衡)来把消息写入不同的MessageQueue中。MessageQueue的数据可以持久化在磁盘上。这样就把消息分散到了多个Broker上，大大提升Broker的抗并发能力

Producer通过NameSever获取指定Topic的Broker路由信息，并在本地保存一份缓存数据，比如一个Topic有哪些MessageQueue，MessageQueue在哪几台Broker上，Broker的ip、port等等。Producer发送消息只发到Master Broker上，Slave通过主从同步获取数据































