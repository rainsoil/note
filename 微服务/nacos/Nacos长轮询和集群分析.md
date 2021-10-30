#  长轮询的时间间隔

我们知道客户端会有一个长训轮的任务去检查服务器端的配置是否发生了变化, 如果发生了变更, 那么客户端会拿到变更的`groupKey` 再根据 `groupKey` 去获取配置项的最新值 跟新到本地的缓存以及文件中,那么这种每次都靠客户端去请求, 那请求的时间间隔设置多少合适呢? 

如果间隔时间设置的太长的话有可能无法及时获取服务端的变更, 如果间隔时间设置的太短的话, 那么频繁的请求对于服务端来说无疑也是一种负担,所以最好的方式是客户端每隔一段长度适中的时间去服务端请求, 而在这期间如果配置发生了变更, 服务端能够主动将变更后的结果推送给客户端, 这样既能够保证客户端能够实时感知到配置的变化, 也降低了服务端的压力, 我们来看看nacos 设置的间隔时间是多久? 

## 长轮询的概念

那么在讲解原理之前, 先给大家解释一下什么叫长轮询. 

客户端发起一个请求到服务端, 服务端收到客户端的请求后, 并不会立即响应给客户端, 而是先把请求hold住, 然后服务端会在hold 住的这段时间内检查数据是否有更新, 如果有, 则响应给客户端. 如果一直没有数据变更, 则达到一定时间(长轮询时间间隔)才返回. 

长轮询典型的场景有: 扫码登陆、扫码支付. 

![](http://files.luyanan.com//img/20191210211140.png)



##  客户端长轮询

我们在 `ClientWorker`  这个类中, 找到`checkUpdateConfigStr` 这个方法, 这里面就是去服务端查询发生变化的  groupKey

```java
/**
	 * 从Server获取值变化了的DataID列表。返回的对象里只有dataId和group是有效的。 保证不返回NULL。
	 */
	List<String> checkUpdateConfigStr(String probeUpdateString, boolean isInitializingCacheList) {

		List<String> params = Arrays.asList(Constants.PROBE_MODIFY_REQUEST, probeUpdateString);
		long timeout = TimeUnit.SECONDS.toMillis(30L);

		List<String> headers = new ArrayList<String>(2);
		headers.add("Long-Pulling-Timeout");
		headers.add("" + timeout);

		// told server do not hang me up if new initializing cacheData added in
		if (isInitializingCacheList) {
			headers.add("Long-Pulling-Timeout-No-Hangup");
			headers.add("true");
		}

		if (StringUtils.isBlank(probeUpdateString)) {
			return Collections.emptyList();
		}

		try {
			HttpResult result = agent.httpPost(Constants.CONFIG_CONTROLLER_PATH + "/listener", headers, params,
					agent.getEncode(), timeout);

			if (HttpURLConnection.HTTP_OK == result.code) {
				setHealthServer(true);
				return parseUpdateDataIdResponse(result.content);
			} else {
				setHealthServer(false);
				if (result.code == HttpURLConnection.HTTP_INTERNAL_ERROR) {
					log.error("NACOS-0007", LoggerHelper.getErrorCodeStr("Nacos", "Nacos-0007", "环境问题",
							"[check-update] get changed dataId error"));
				}
				log.error(agent.getName(), "NACOS-XXXX", "[check-update] get changed dataId error, code={}",
						result.code);
			}
		} catch (IOException e) {
			setHealthServer(false);
			log.error(agent.getName(), "NACOS-XXXX", "[check-update] get changed dataId exception, msg={}",
					e.toString());
		}
		return Collections.emptyList();
	}

```

这个方法最终会发起http请求, 注意这里面有个 `timeout` 属性. 

```java
HttpResult result = agent.httpPost(Constants.CONFIG_CONTROLLER_PATH + "/listener", headers, params,
					agent.getEncode(), timeout);
```



 timeout 是在init这个方法中赋值的, 默认情况下是30秒, 可以通过 `configLongPollTimeout` 进行修改, 

```java
 private void init(Properties properties) {

        timeout = Math.max(NumberUtils.toInt(properties.getProperty(PropertyKeyConst.CONFIG_LONG_POLL_TIMEOUT),
            Constants.CONFIG_LONG_POLL_TIMEOUT), Constants.MIN_CONFIG_LONG_POLL_TIMEOUT);

        taskPenaltyTime = NumberUtils.toInt(properties.getProperty(PropertyKeyConst.CONFIG_RETRY_TIME), Constants.CONFIG_RETRY_TIME);

        enableRemoteSyncConfig = Boolean.parseBoolean(properties.getProperty(PropertyKeyConst.ENABLE_REMOTE_SYNC_CONFIG));
    }
```

所有从这里得出的一个基本结论是:

> 客户端发起一个轮询请求,超时时间是 30s, 那么客户端为什么要等待30s 才超时呢？不是越快越好吗?

## 客户端长轮询的时间间隔

我们可以在nacos 的日志目录下 `$NACOS_HOME/nacos/logs/config-client-request.log` 文件中: 
```
2019-08-04 13:22:19,736|0|nohangup|127.0.0.1|polling|1|55|0
2019-08-04 13:22:49,443|29504|timeout|127.0.0.1|polling|1|55
2019-08-04 13:23:18,983|29535|timeout|127.0.0.1|polling|1|55
2019-08-04 13:23:48,493|29501|timeout|127.0.0.1|polling|1|55
2019-08-04 13:24:18,003|29500|timeout|127.0.0.1|polling|1|55
2019-08-04 13:24:47,509|29501|timeout|127.0.0.1|polling|1|55

```
可以看到一个现象, 在配置没有发生变化的情况下, 客户端会等待29.5s以上, 才请求到服务端的结果. 然后客户端拿到服务器端的结果后, 在做后续的操作. 

如果在配置发生变更的情况下, 由于客户端基于长轮询的连接保持, 所以返回的时间会非常短. 我摁可以做个小实验, 在`nacos console` 中频繁修改数据在观察一下:

```
2019-08-04 13:30:17,016|0|inadvance|127.0.0.1|polling|1|55|example+DEFAULT_GROUP
2019-08-04
13:30:17,022|3|null|127.0.0.1|get|example|DEFAULT_GROUP||e10e4d5973c497e490a8d7a
9e4e9be64|unknown
2019-08-04
13:30:20,807|10|true|0:0:0:0:0:0:0:1|publish|example|DEFAULT_GROUP||81360b7e732a
5dbb37d62d81cebb85d2|null
2019-08-04 13:30:20,843|0|inadvance|127.0.0.1|polling|1|55|example+DEFAULT_GROUP
2019-08-04
13:30:20,848|1|null|127.0.0.1|get|example|DEFAULT_GROUP||81360b7e732a5dbb37d62d8
1cebb85d2|unknown

```

![](http://files.luyanan.com//img/20191212102604.png)

##  服务端的处理

分析完客户端之后, 随着好奇心的驱使, 服务端是如何处理客户端的请求呢? 那么同样, 我们需要思考几个问题:

- 客户端的长轮询响应时间是受到哪些因素的影响
- 客户端的超时时间为什么要设置30s

客户端发送请求的地址是 `/v1/cs/configs/listener`, 找到服务端对应的方法. 

###  ConfigController

nacos 是使用 spring mvc 提供的rest api, 这里会调用`inner.doPollingConfig` 进行处理. 

```java
 /**
     * 比较MD5
     */
    @RequestMapping(value = "/listener", method = RequestMethod.POST)
    public void listener(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {
        request.setAttribute("org.apache.catalina.ASYNC_SUPPORTED", true);
        String probeModify = request.getParameter("Listening-Configs");
        if (StringUtils.isBlank(probeModify)) {
            throw new IllegalArgumentException("invalid probeModify");
        }

        probeModify = URLDecoder.decode(probeModify, Constants.ENCODE);

        Map<String, String> clientMd5Map;
        try {
            clientMd5Map = MD5Util.getClientMd5Map(probeModify);
        } catch (Throwable e) {
            throw new IllegalArgumentException("invalid probeModify");
        }

        // do long-polling
        inner.doPollingConfig(request, response, clientMd5Map, probeModify.length());
    }
```



### doPollingConfig

这个方法中,兼容了长轮询和短轮询的逻辑, 我们只需要关注长轮询的部分, 再次进入到 `longPollingService.addLongPollingClient` 

```java
  /**
     * 轮询接口
     */
    public String doPollingConfig(HttpServletRequest request, HttpServletResponse response,
                                  Map<String, String> clientMd5Map, int probeRequestSize)
        throws IOException, ServletException {

        // 长轮询
        if (LongPollingService.isSupportLongPolling(request)) {
            longPollingService.addLongPollingClient(request, response, clientMd5Map, probeRequestSize);
            return HttpServletResponse.SC_OK + "";
        }

        // else 兼容短轮询逻辑
        List<String> changedGroups = MD5Util.compareMd5(request, response, clientMd5Map);

        // 兼容短轮询result
        String oldResult = MD5Util.compareMd5OldResult(changedGroups);
        String newResult = MD5Util.compareMd5ResultString(changedGroups);

        String version = request.getHeader(Constants.CLIENT_VERSION_HEADER);
        if (version == null) {
            version = "2.0.0";
        }
        int versionNum = Protocol.getVersionNumber(version);

        /**
         * 2.0.4版本以前, 返回值放入header中
         */
        if (versionNum < START_LONGPOLLING_VERSION_NUM) {
            response.addHeader(Constants.PROBE_MODIFY_RESPONSE, oldResult);
            response.addHeader(Constants.PROBE_MODIFY_RESPONSE_NEW, newResult);
        } else {
            request.setAttribute("content", newResult);
        }

        // 禁用缓存
        response.setHeader("Pragma", "no-cache");
        response.setDateHeader("Expires", 0);
        response.setHeader("Cache-Control", "no-cache,no-store");
        response.setStatus(HttpServletResponse.SC_OK);
        return HttpServletResponse.SC_OK + "";
    }
```

### longPollingService.addLongPollingClient

从方法的名字上可以推出, 这个方法应该是把客户端的长轮询请求添加到某个任务中. 

- 获得客户端传递过来的超时时间, 并且进行本地计算, 提前500s 返回响应, 这就能解释为什么客户端响应超时时间是29.5+了, 当然如果 `isFixedPolling=true` 的情况下, 不会提前返回响应. 
- 根据客户端请求过来的md5和服务端对应的group 下对应内容的md5 进行比较, 如果不一致, 则通过`generateResponse`  将结果返回. 
- 如果配置文件没有发生变化, 则通过 `scheduler.execute`   启动了一个定时任务, 将客户端的长轮询请求封装成一个叫` ClientLongPolling` 的任务, 交给 ` scheduler` 去执行. 

```java
  public void addLongPollingClient(HttpServletRequest req, HttpServletResponse rsp, Map<String, String> clientMd5Map,
                                     int probeRequestSize) {

        String str = req.getHeader(LongPollingService.LONG_POLLING_HEADER);
        String noHangUpFlag = req.getHeader(LongPollingService.LONG_POLLING_NO_HANG_UP_HEADER);
        String appName = req.getHeader(RequestUtil.CLIENT_APPNAME_HEADER);
        String tag = req.getHeader("Vipserver-Tag");
        int delayTime = SwitchService.getSwitchInteger(SwitchService.FIXED_DELAY_TIME, 500);
        /**
         * 提前500ms返回响应，为避免客户端超时 @qiaoyi.dingqy 2013.10.22改动  add delay time for LoadBalance
         */
        long timeout = Math.max(10000, Long.parseLong(str) - delayTime);
        if (isFixedPolling()) {
            timeout = Math.max(10000, getFixedPollingInterval());
            // do nothing but set fix polling timeout
        } else {
            long start = System.currentTimeMillis();
            List<String> changedGroups = MD5Util.compareMd5(req, rsp, clientMd5Map);
            if (changedGroups.size() > 0) {
                generateResponse(req, rsp, changedGroups);
                LogUtil.clientLog.info("{}|{}|{}|{}|{}|{}|{}",
                    System.currentTimeMillis() - start, "instant", RequestUtil.getRemoteIp(req), "polling",
                    clientMd5Map.size(), probeRequestSize, changedGroups.size());
                return;
            } else if (noHangUpFlag != null && noHangUpFlag.equalsIgnoreCase(TRUE_STR)) {
                LogUtil.clientLog.info("{}|{}|{}|{}|{}|{}|{}", System.currentTimeMillis() - start, "nohangup",
                    RequestUtil.getRemoteIp(req), "polling", clientMd5Map.size(), probeRequestSize,
                    changedGroups.size());
                return;
            }
        }
        String ip = RequestUtil.getRemoteIp(req);
        // 一定要由HTTP线程调用，否则离开后容器会立即发送响应
        final AsyncContext asyncContext = req.startAsync();
        // AsyncContext.setTimeout()的超时时间不准，所以只能自己控制
        asyncContext.setTimeout(0L);

        scheduler.execute(
            new ClientLongPolling(asyncContext, clientMd5Map, ip, probeRequestSize, timeout, appName, tag));
    }
```



###  ClientLongPolling

我们来分析一下, ClientLongPolling 到底做了什么操作? 或者说我们可以先猜测一下应该会做什么事情? 

- 这个任务是要阻塞 29.5s 才能执行, 以为立马执行没有任何意思, 毕竟前面已经执行过一次了
- 如果在29.5s+ 之内, 数据发生变化, 需要提前通知, 需要有一种监控机制. 

基于这些猜想, 我们来看看他的实现过程. 

从代码粗粒度来看 ，它的实现似乎跟我们的猜想一致, 在run方法中, 通过`scheduler.schedule` 实现了一个定时任务, 它的delay 时间正好是前面计算的29.5s. 在这个任务中, 会通过`MD5Util.compareMd5` 来进行计算. 

那另外一个, 当数据发生变化以后, 肯定不能得到 29.5s 之后才通知呀, 那怎么办呢? 我们发现有一个 `allsubs` 的东西, 它似乎和发布订阅有关系,那是不是有可能当前的`clientLongPolling` 订阅了数据变化的时间呢? 

```java
  @Override
        public void run() {
            asyncTimeoutFuture = scheduler.schedule(new Runnable() {
                @Override
                public void run() {
                    try {
                        getRetainIps().put(ClientLongPolling.this.ip, System.currentTimeMillis());
                        /**
                         * 删除订阅关系
                         */
                        allSubs.remove(ClientLongPolling.this);

                        if (isFixedPolling()) {
                            LogUtil.clientLog.info("{}|{}|{}|{}|{}|{}",
                                (System.currentTimeMillis() - createTime),
                                "fix", RequestUtil.getRemoteIp((HttpServletRequest)asyncContext.getRequest()),
                                "polling",
                                clientMd5Map.size(), probeRequestSize);
                            List<String> changedGroups = MD5Util.compareMd5(
                                (HttpServletRequest)asyncContext.getRequest(),
                                (HttpServletResponse)asyncContext.getResponse(), clientMd5Map);
                            if (changedGroups.size() > 0) {
                                sendResponse(changedGroups);
                            } else {
                                sendResponse(null);
                            }
                        } else {
                            LogUtil.clientLog.info("{}|{}|{}|{}|{}|{}",
                                (System.currentTimeMillis() - createTime),
                                "timeout", RequestUtil.getRemoteIp((HttpServletRequest)asyncContext.getRequest()),
                                "polling",
                                clientMd5Map.size(), probeRequestSize);
                            sendResponse(null);
                        }
                    } catch (Throwable t) {
                        LogUtil.defaultLog.error("long polling error:" + t.getMessage(), t.getCause());
                    }

                }

            }, timeoutTime, TimeUnit.MILLISECONDS);

            allSubs.add(this);
        }
```



### allSubs

allSubs 是一个队列, 队列里面存放了 `ClientLongPolling` 这个对象, 这个队列似乎和配置变更有某种关联关系. 

```java
    /**
     * 长轮询订阅关系
     */
    final Queue<ClientLongPolling> allSubs;	
```



那这个时候, 我的第一想法是, 先去看一下当前类的类图, 发现 `LongPollingService` 继承了`AbstractEventListener`,事件监听. 



![](http://files.luyanan.com//img/20191212111937.png)

### AbstractEventListener

这里面有个抽象的onEvent方法, 明显是用来处理事件的方法, 而抽象方法必须由子类实现, 所以意味着`LongPollingService` 里面肯定实现了onEvent 方法. 

```java
  static public abstract class AbstractEventListener {

        public AbstractEventListener() {
            /**
             * automatic register
             */
            EventDispatcher.addEventListener(this);
        }

        /**
         * 感兴趣的事件列表
         *
         * @return event list
         */
        abstract public List<Class<? extends Event>> interest();

        /**
         * 处理事件
         *
         * @param event event
         */
        abstract public void onEvent(Event event);
    }
```



### LongPollingService.onEvent

这个事件的实现方法中: 

- 判断事件类型是否为`LocalDataChangeEvent`
- 通过`scheduler.execute` 执行 `DataChangeTask` 这个任务. 

```java
   @Override
    public void onEvent(Event event) {
        if (isFixedPolling()) {
            // ignore
        } else {
            if (event instanceof LocalDataChangeEvent) {
                LocalDataChangeEvent evt = (LocalDataChangeEvent)event;
                scheduler.execute(new DataChangeTask(evt.groupKey, evt.isBeta, evt.betaIps));
            }
        }
    }
```



### DataChangeTask.run

从名字来看, 这个是数据变化的任务, 最让人兴奋的是, 这里面有一个循环迭代器, 从 `allsubs` 里面获取`ClientLongPolling`

最后通过`clientSub.sendResponse`把数据返回到客户端, 所以这也就能理解为啥数据变化能够实时触发更新了. 

```java
    @Override
        public void run() {
            try {
                ConfigService.getContentBetaMd5(groupKey);
                for (Iterator<ClientLongPolling> iter = allSubs.iterator(); iter.hasNext(); ) {
                    ClientLongPolling clientSub = iter.next();
                    if (clientSub.clientMd5Map.containsKey(groupKey)) {
                        // 如果beta发布且不在beta列表直接跳过
                        if (isBeta && !betaIps.contains(clientSub.ip)) {
                            continue;
                        }

                        // 如果tag发布且不在tag列表直接跳过
                        if (StringUtils.isNotBlank(tag) && !tag.equals(clientSub.tag)) {
                            continue;
                        }

                        getRetainIps().put(clientSub.ip, System.currentTimeMillis());
                        iter.remove(); // 删除订阅关系
                        LogUtil.clientLog.info("{}|{}|{}|{}|{}|{}|{}",
                            (System.currentTimeMillis() - changeTime),
                            "in-advance",
                            RequestUtil.getRemoteIp((HttpServletRequest)clientSub.asyncContext.getRequest()),
                            "polling",
                            clientSub.clientMd5Map.size(), clientSub.probeRequestSize, groupKey);
                        clientSub.sendResponse(Arrays.asList(groupKey));
                    }
                }
            } catch (Throwable t) {
                LogUtil.defaultLog.error("data change error:" + t.getMessage(), t.getCause());
            }
        }
```



那么接下来的一个疑问是, 数据变化之后是如何触发事件的呢? 所以我们定位到数据变化的请求类中, 在 `configController` 这个类中, 找到POST 请求的方法. 

找到配置变更的位置, 发现数据持久化之后 , 会通过`EventDispatcher` 进行事件发布, `EventDispatcher.fireEvent`. 但这个事件似乎不是我们所关心的时间, 原因是这里发布的事件是 `ConfigDataChangeEvent`,而`LongPollingService` 最感兴趣的事件是 `LocalDataChangeEvent`. 

```java
  /**
     * 增加或更新非聚合数据。
     *
     * @throws NacosException
     */
    @RequestMapping(method = RequestMethod.POST)
    @ResponseBody
    public Boolean publishConfig(HttpServletRequest request, HttpServletResponse response,
                                 @RequestParam("dataId") String dataId, @RequestParam("group") String group,
                                 @RequestParam(value = "tenant", required = false, defaultValue = StringUtils.EMPTY)
                                     String tenant,
                                 @RequestParam("content") String content,
                                 @RequestParam(value = "tag", required = false) String tag,
                                 @RequestParam(value = "appName", required = false) String appName,
                                 @RequestParam(value = "src_user", required = false) String srcUser,
                                 @RequestParam(value = "config_tags", required = false) String configTags,
                                 @RequestParam(value = "desc", required = false) String desc,
                                 @RequestParam(value = "use", required = false) String use,
                                 @RequestParam(value = "effect", required = false) String effect,
                                 @RequestParam(value = "type", required = false) String type,
                                 @RequestParam(value = "schema", required = false) String schema)
        throws NacosException {
        final String srcIp = RequestUtil.getRemoteIp(request);
        String requestIpApp = RequestUtil.getAppName(request);
        ParamUtils.checkParam(dataId, group, "datumId", content);
        ParamUtils.checkParam(tag);

        Map<String, Object> configAdvanceInfo = new HashMap<String, Object>(10);
        if (configTags != null) {
            configAdvanceInfo.put("config_tags", configTags);
        }
        if (desc != null) {
            configAdvanceInfo.put("desc", desc);
        }
        if (use != null) {
            configAdvanceInfo.put("use", use);
        }
        if (effect != null) {
            configAdvanceInfo.put("effect", effect);
        }
        if (type != null) {
            configAdvanceInfo.put("type", type);
        }
        if (schema != null) {
            configAdvanceInfo.put("schema", schema);
        }
        ParamUtils.checkParam(configAdvanceInfo);

        if (AggrWhitelist.isAggrDataId(dataId)) {
            log.warn("[aggr-conflict] {} attemp to publish single data, {}, {}",
                RequestUtil.getRemoteIp(request), dataId, group);
            throw new NacosException(NacosException.NO_RIGHT, "dataId:" + dataId + " is aggr");
        }

        final Timestamp time = TimeUtils.getCurrentTime();
        String betaIps = request.getHeader("betaIps");
        ConfigInfo configInfo = new ConfigInfo(dataId, group, tenant, appName, content);
        if (StringUtils.isBlank(betaIps)) {
            if (StringUtils.isBlank(tag)) {
                persistService.insertOrUpdate(srcIp, srcUser, configInfo, time, configAdvanceInfo, false);
                EventDispatcher.fireEvent(new ConfigDataChangeEvent(false, dataId, group, tenant, time.getTime()));
            } else {
                persistService.insertOrUpdateTag(configInfo, tag, srcIp, srcUser, time, false);
                EventDispatcher.fireEvent(new ConfigDataChangeEvent(false, dataId, group, tenant, tag, time.getTime()));
            }
        } else { // beta publish
            persistService.insertOrUpdateBeta(configInfo, betaIps, srcIp, srcUser, time, false);
            EventDispatcher.fireEvent(new ConfigDataChangeEvent(true, dataId, group, tenant, time.getTime()));
        }
        ConfigTraceService.logPersistenceEvent(dataId, group, tenant, requestIpApp, time.getTime(),
            LOCAL_IP, ConfigTraceService.PERSISTENCE_EVENT_PUB, content);

        return true;
    }
```



后来我发现, 在 Nacos中有一个 `DumpService`, 它会定时把变更后的数据 dump 到磁盘上, `DumpService` 在spring 启动后, 会调用init 方法 启动几个 dump任务, 然后在任务结束后, 会触发一个 `LocalDataChangeEvent` 的事件. 

```java
 @PostConstruct
    public void init() {
        LogUtil.defaultLog.warn("DumpService start");
        DumpProcessor processor = new DumpProcessor(this);
        DumpAllProcessor dumpAllProcessor = new DumpAllProcessor(this);
        DumpAllBetaProcessor dumpAllBetaProcessor = new DumpAllBetaProcessor(this);
        DumpAllTagProcessor dumpAllTagProcessor = new DumpAllTagProcessor(this);

        dumpTaskMgr = new TaskManager(
            "com.alibaba.nacos.server.DumpTaskManager");
        dumpTaskMgr.setDefaultTaskProcessor(processor);

        dumpAllTaskMgr = new TaskManager(
            "com.alibaba.nacos.server.DumpAllTaskManager");
        dumpAllTaskMgr.setDefaultTaskProcessor(dumpAllProcessor);

        Runnable dumpAll = new Runnable() {
            @Override
            public void run() {
                dumpAllTaskMgr.addTask(DumpAllTask.TASK_ID, new DumpAllTask());
            }
        };

        Runnable dumpAllBeta = new Runnable() {
            @Override
            public void run() {
                dumpAllTaskMgr.addTask(DumpAllBetaTask.TASK_ID, new DumpAllBetaTask());
            }
        };

        Runnable clearConfigHistory = new Runnable() {
            @Override
            public void run() {
                log.warn("clearConfigHistory start");
                if (ServerListService.isFirstIp()) {
                    try {
                        Timestamp startTime = getBeforeStamp(TimeUtils.getCurrentTime(), 24 * getRetentionDays());
                        int totalCount = persistService.findConfigHistoryCountByTime(startTime);
                        if (totalCount > 0) {
                            int pageSize = 1000;
                            int removeTime = (totalCount + pageSize - 1) / pageSize;
                            log.warn("clearConfigHistory, getBeforeStamp:{}, totalCount:{}, pageSize:{}, removeTime:{}",
                                new Object[] {startTime, totalCount, pageSize, removeTime});
                            while (removeTime > 0) {
                                // 分页删除，以免批量太大报错
                                persistService.removeConfigHistory(startTime, pageSize);
                                removeTime--;
                            }
                        }
                    } catch (Throwable e) {
                        log.error("clearConfigHistory error", e);
                    }
                }
            }
        };

        try {
            dumpConfigInfo(dumpAllProcessor);

            // 更新beta缓存
            LogUtil.defaultLog.info("start clear all config-info-beta.");
            DiskUtil.clearAllBeta();
            if (persistService.isExistTable(BETA_TABLE_NAME)) {
                dumpAllBetaProcessor.process(DumpAllBetaTask.TASK_ID, new DumpAllBetaTask());
            }
            // 更新Tag缓存
            LogUtil.defaultLog.info("start clear all config-info-tag.");
            DiskUtil.clearAllTag();
            if (persistService.isExistTable(TAG_TABLE_NAME)) {
                dumpAllTagProcessor.process(DumpAllTagTask.TASK_ID, new DumpAllTagTask());
            }

            // add to dump aggr
            List<ConfigInfoChanged> configList = persistService.findAllAggrGroup();
            if (configList != null && !configList.isEmpty()) {
                total = configList.size();
                List<List<ConfigInfoChanged>> splitList = splitList(configList, INIT_THREAD_COUNT);
                for (List<ConfigInfoChanged> list : splitList) {
                    MergeAllDataWorker work = new MergeAllDataWorker(list);
                    work.start();
                }
                log.info("server start, schedule merge end.");
            }
        } catch (Exception e) {
            LogUtil.fatalLog.error(
                "Nacos Server did not start because dumpservice bean construction failure :\n" + e.getMessage(),
                e.getCause());
            throw new RuntimeException(
                "Nacos Server did not start because dumpservice bean construction failure :\n" + e.getMessage());
        }
        if (!STANDALONE_MODE) {
            Runnable heartbeat = new Runnable() {
                @Override
                public void run() {
                    String heartBeatTime = TimeUtils.getCurrentTime().toString();
                    // write disk
                    try {
                        DiskUtil.saveHeartBeatToDisk(heartBeatTime);
                    } catch (IOException e) {
                        LogUtil.fatalLog.error("save heartbeat fail" + e.getMessage());
                    }
                }
            };

            TimerTaskService.scheduleWithFixedDelay(heartbeat, 0, 10, TimeUnit.SECONDS);

            long initialDelay = new Random().nextInt(INITIAL_DELAY_IN_MINUTE) + 10;
            LogUtil.defaultLog.warn("initialDelay:{}", initialDelay);

            TimerTaskService.scheduleWithFixedDelay(dumpAll, initialDelay, DUMP_ALL_INTERVAL_IN_MINUTE,
                TimeUnit.MINUTES);

            TimerTaskService.scheduleWithFixedDelay(dumpAllBeta, initialDelay, DUMP_ALL_INTERVAL_IN_MINUTE,
                TimeUnit.MINUTES);
        }

        TimerTaskService.scheduleWithFixedDelay(clearConfigHistory, 10, 10, TimeUnit.MINUTES);

    }
```





## 简单总结

简单总结一i下刚才分析的过程: 

- 客户端发起长轮询请求. 

- 服务端收到i请求后, 先比较服务端缓存中的数据是否相同,如果不同, 则直接返回. 

- 如果相同, 则通过 `schedule` 延迟29.5s 之后再执行比较. 

- 为了保证当服务端在 29.5s之内数据发生变化能够及时的通知给客户端, 服务端采用事件订阅的方式来监听服务端本地数据变化的事件, 一旦收到事件, 则触发`ClientLongPolling` , 把结果写回到客户端, 就完成了一次数据的推送. 

- 如果 `ClientLongPolling` 任务完成了数据的推送之后, ClientLongPolling 中的调度任务又开始执行了怎么办？

    很见到那, 只要在进行推送操作之前, 先将原来等待执行的调度任务取消就行了,这样就方式了推送操作写完响应数据之后， 调用任务又去写响应数据, 这时肯定报错. 所以, 在`ClientLongPolling` 方法中, 最开始的一个步骤就是删除订阅事件. 

所以总的来说, Nacos 采用推+拉的方式, 来解决最开始关于长轮询事件间隔的问题, 当然, 30s 这个时间是可以设置的, 之所以设置成30s, 应该是一个经验值, 



# 集群选举问题

Nacos 支持集群模式,很显然, . 

而一旦涉及到集群, 就涉及到主从, 那么nacos 是一种什么样的机制来实现集群的呢? 

nacos 的集群模式类似于zookeeper, 它分为leader 角色和 follower角色, 那么从这个角色的名字可以看出, 这个集群存在选举的机制, 因为如果自己不具备选举功能, 角色的命名可能就是master/slave了. 

##  选举算法

nacos集群采用`raft` 算法来实现, 它是相对于zookeeper 的选举算法来说比较简单的一种. 

选举算法的核心在 `RaftCore`中, 包括数据的处理和数据的同步. 



[http://thesecretlivesofdata.com/raft/]:raft算法演示地址

在Raft 算法中, 节点有三种角色: 

- Leader : 负责接受客户端的请求. 
- Candidate: 用于选举Leader 的一种角色
- Follower: 负责响应来自leader  或者Candidate 的请求. 

选举分为两个节点:

- 服务启动的时候
- leader 挂了的时候

所有节点启动的时候, 都是 folower 状态, 如果在一段时间内如果没有收到leader 的心跳(可能是没有leader 或者leader 挂了), 那么folower 会变成Candidate . 然后发起选举, 在选举之前, 会增加term, 这个term 和zookeeper 中的epoch 的道理是一样的. 

- folower 会投自己一票, 并且给其他节点发送票据`vote`, 等待其他节点回复. 
- 在这个过程中, 可能出现几种情况: 
  - 收到过半的票数通过, 则成为leader
  - 被告知其他节点已经成为了leader, 则自己切换为 folower
  - 一段时间内没有收到过半的投票, 则重新发起选举. 
- 约束条件在任一term中, 单个节点最多只能投一票. 

选举的几种情况: 

- 第一种情况:  赢得选举后, leader 会给所有节点发送消息, 避免其他节点触发新的选举. 
- 第二种情况: 比如有三个节点A、B、C。 A、B 同时发起选举, 而A的选举消息先达到C,C给A 投了一票, 当B 的消息达到C时, 已经不能满足上面提到的第一个约束, 而C不会给B投票, 而A和B显然都不会给对方投票. A 胜出之后, 会给B、C 发送心跳消息, 节点B发现节点A的term 不低于自己的term, 知道已经有了leader, 于是转换为folower. 
- 第三种情况: 没有任何节点获得投票, 可能是平票的情况. 加入总共有四个节点, (A/B/C/D). NodeC、NodeD 同时成为了 candidate.但NodeA  投了NodeD 一票,NodeB 投了NodeC 一票. 这就出现了凭票 spilt vote 的情况. 这个时候大家都在等呀等呀. 知道超时后重新发起选举, 如果出现平票的情况, 那么就延长了系统不可用的时间, 于是raft 就引入了`randomized election timeouts`  来尽量避免平票的情况. 

## 数据的处理

对于事务操作, 请求会转发给leader. 

非事务操作, 可以任意一个节点来处理. 



下面这段代码摘自 RaftCore,.在发布内容的时候, 做了两个事情. 

- 如果当前的节点不是leader, 则转发给leader 节点处理. 
- 如果是, 则向所有节点发送 onPublish



```java
 public void signalPublish(String key, Record value) throws Exception {

        if (!isLeader()) {
            JSONObject params = new JSONObject();
            params.put("key", key);
            params.put("value", value);
            Map<String, String> parameters = new HashMap<>(1);
            parameters.put("key", key);

            raftProxy.proxyPostLarge(getLeader().ip, API_PUB, params.toJSONString(), parameters);
            return;
        }

        try {
            OPERATE_LOCK.lock();
            long start = System.currentTimeMillis();
            final Datum datum = new Datum();
            datum.key = key;
            datum.value = value;
            if (getDatum(key) == null) {
                datum.timestamp.set(1L);
            } else {
                datum.timestamp.set(getDatum(key).timestamp.incrementAndGet());
            }

            JSONObject json = new JSONObject();
            json.put("datum", datum);
            json.put("source", peers.local());

            onPublish(datum, peers.local());

            final String content = JSON.toJSONString(json);

            final CountDownLatch latch = new CountDownLatch(peers.majorityCount());
            for (final String server : peers.allServersIncludeMyself()) {
                if (isLeader(server)) {
                    latch.countDown();
                    continue;
                }
                final String url = buildURL(server, API_ON_PUB);
                HttpClient.asyncHttpPostLarge(url, Arrays.asList("key=" + key), content, new AsyncCompletionHandler<Integer>() {
                    @Override
                    public Integer onCompleted(Response response) throws Exception {
                        if (response.getStatusCode() != HttpURLConnection.HTTP_OK) {
                            Loggers.RAFT.warn("[RAFT] failed to publish data to peer, datumId={}, peer={}, http code={}",
                                datum.key, server, response.getStatusCode());
                            return 1;
                        }
                        latch.countDown();
                        return 0;
                    }

                    @Override
                    public STATE onContentWriteCompleted() {
                        return STATE.CONTINUE;
                    }
                });

            }

            if (!latch.await(UtilsAndCommons.RAFT_PUBLISH_TIMEOUT, TimeUnit.MILLISECONDS)) {
                // only majority servers return success can we consider this update success
                Loggers.RAFT.error("data publish failed, caused failed to notify majority, key={}", key);
                throw new IllegalStateException("data publish failed, caused failed to notify majority, key=" + key);
            }

            long end = System.currentTimeMillis();
            Loggers.RAFT.info("signalPublish cost {} ms, key: {}", (end - start), key);
        } finally {
            OPERATE_LOCK.unlock();
        }
    }

```





