# Nacos 原理分析以及实战

##  简介

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。

Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。                                                                

​                                                                                                                                     --> copy 自官网. 

##  实战演练

###  下载安装

#### 1.下载源码或者安装包

你可以通过源码和发行包两种方式来获取 Nacos。

##### 从 Github 上下载源码方式

```bash
git clone https://github.com/alibaba/nacos.git
cd nacos/
mvn -Prelease-nacos clean install -U  
ls -al distribution/target/

// change the $version to your actual path
cd distribution/target/nacos-server-$version/nacos/bin
```

##### 下载编译后压缩包方式

您可以从 [最新稳定版本](https://github.com/alibaba/nacos/releases) 下载 `nacos-server-$version.zip` 包。

```bash
  unzip nacos-server-$version.zip 或者 tar -xvf nacos-server-$version.tar.gz
  cd nacos/bin
```

#### 2.启动服务器

##### Linux/Unix/Mac

启动命令(standalone代表着单机模式运行，非集群模式):

```
sh startup.sh -m standalone
```

如果您使用的是ubuntu系统，或者运行脚本报错提示[[符号找不到，可尝试如下运行：

```
bash startup.sh -m standalone
```

##### Windows

启动命令：

```
cmd startup.cmd
```

或者双击startup.cmd运行文件。



###  配置中心试用

####  服务端
我们先登录http://192.168.31.22:8848/nacos/index.html ,账号密码默认为nacos/nacos.
我们在 

![](http://files.luyanan.com//img/20191204152256.png)


####  客户端
我们先新建一个springboot 项目, 然后在pom.xml 中加入config 的依赖
```java
 <dependency>
            <groupId>com.alibaba.boot</groupId>
            <artifactId>nacos-config-spring-boot-starter</artifactId>
            <version>0.2.3</version>
        </dependency>
```

新建一个 `NacosConfigController` , 然后做如下配置
```java
@NacosPropertySource(dataId = "example", autoRefreshed = true)
@RestController
@RequestMapping("config")
public class NacosConfigController {

    @NacosValue(value = "{info:默认}", autoRefreshed = true)
    private String info;


    @GetMapping("info")
    public String info() {
        return this.info;
    }
}
```

### 测试

我们在浏览器访问 `http://localhost:8080/config/info`, 可以看到返回的结果是在nacos中配置的信息



###  使用SDK 进行访问

我们在项目中加入

```java
<dependency>
<groupId>com.alibaba.nacos</groupId>
<artifactId>nacos-client</artifactId>
<version>${version}</version>
</dependency>
```

```java
 public static void main(String[] args) {
        try {
            String serverAddr = "127.0.0.1:8848";
            String dataId = "example";
            String group = "DEFAULT_GROUP";
            Properties properties = new Properties();
            properties.put("serverAddr", serverAddr);
            ConfigService configService = NacosFactory.createConfigService(properties);
            String content = configService.getConfig(dataId, group, 5000);
            System.out.println(content);
        } catch (NacosException e) {
            e.printStackTrace();
        }
    }
```







## 源码分析

###  ConfigService

我们通过 `NacosFactory.createConfigService(properties)`  知道,通过ConfigFactory 工厂构建了ConfigService

```java
	public static ConfigService createConfigService(Properties properties) throws NacosException {
		return ConfigFactory.createConfigService(properties);
	}
```

```java
	public static ConfigService createConfigService(Properties properties) throws NacosException {
		try {
			Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
			Constructor constructor = driverImplClass.getConstructor(Properties.class);
			ConfigService vendorImpl = (ConfigService) constructor.newInstance(properties);
			return vendorImpl;
		} catch (Throwable e) {
			throw new NacosException(-400, e.getMessage());
		}
	}
```

我们看到这里通过反射初始化了一个NacosConfigService. 

###  configService.getConfig

```java
private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) throws NacosException {
		group = null2defaultGroup(group);
		ParamUtils.checkKeyParam(dataId, group);
		ConfigResponse cr = new ConfigResponse();

		cr.setDataId(dataId);
		cr.setTenant(tenant);
		cr.setGroup(group);

		// 优先使用本地配置
		String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
		if (content != null) {
			log.warn(agent.getName(), "[get-config] get failover ok, dataId={}, group={}, tenant={}, config={}", dataId,
					group, tenant, ContentUtils.truncateContent(content));
			cr.setContent(content);
			configFilterChainManager.doFilter(null, cr);
			content = cr.getContent();
			return content;
		}

		try {
            // 从服务端获取
			content = worker.getServerConfig(dataId, group, tenant, timeoutMs);
			cr.setContent(content);
			configFilterChainManager.doFilter(null, cr);
			content = cr.getContent();
			return content;
		} catch (NacosException ioe) {
			if (NacosException.NO_RIGHT == ioe.getErrCode()) {
				throw ioe;
			}
			log.warn("NACOS-0003",
					LoggerHelper.getErrorCodeStr("NACOS", "NACOS-0003", "环境问题", "get from server error"));
			log.warn(agent.getName(), "[get-config] get from server error, dataId={}, group={}, tenant={}, msg={}",
					dataId, group, tenant, ioe.toString());
		}

		log.warn(agent.getName(), "[get-config] get snapshot ok, dataId={}, group={}, tenant={}, config={}", dataId,
				group, tenant, ContentUtils.truncateContent(content));
		content = LocalConfigInfoProcessor.getSnapshot(agent.getName(), dataId, group, tenant);
		cr.setContent(content);
		configFilterChainManager.doFilter(null, cr);
		content = cr.getContent();
		return content;
	}
```

那么 agent 在哪里被初始化的呢? 



我们看到 NacosConfigService 的构造方法里面 

```java 

public NacosConfigService(Properties properties) throws NacosException {
		String encodeTmp = properties.getProperty(PropertyKeyConst.ENCODE);
		if (StringUtils.isBlank(encodeTmp)) {
			encode = Constants.ENCODE;
		} else {
			encode = encodeTmp.trim();
		}
		String namespaceTmp = properties.getProperty(PropertyKeyConst.NAMESPACE);
		if (StringUtils.isBlank(namespaceTmp)) {
			namespace = TenantUtil.getUserTenant();
			properties.put(PropertyKeyConst.NAMESPACE, namespace);
		} else {
			namespace = namespaceTmp;
			properties.put(PropertyKeyConst.NAMESPACE, namespace);
		}
		agent = new ServerHttpAgent(properties);
		agent.start();
		worker = new ClientWorker(agent, configFilterChainManager);
	}


```



###  worker.getServerConfig(dataId, group, tenant, timeoutMs)

```java
public String getServerConfig(String dataId, String group, String tenant, long readTimeout)
			throws NacosException {
		if (StringUtils.isBlank(group)) {
			group = Constants.DEFAULT_GROUP;
		}

		HttpResult result = null;
		try {
			List<String> params = null;
			if (StringUtils.isBlank(tenant)) {
				params = Arrays.asList("dataId", dataId, "group", group);
			} else {
				params = Arrays.asList("dataId", dataId, "group", group, "tenant", tenant);
			}
			result = agent.httpGet(Constants.CONFIG_CONTROLLER_PATH, null, params, agent.getEncode(), readTimeout);
		} catch (IOException e) {
			log.error(agent.getName(), "NACOS-XXXX",
					"[sub-server] get server config exception, dataId={}, group={}, tenant={}, msg={}", dataId, group,
					tenant, e.toString());
			throw new NacosException(NacosException.SERVER_ERROR, e.getMessage());
		}

		switch (result.code) {
		case HttpURLConnection.HTTP_OK:
			LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, result.content);
			return result.content;
		case HttpURLConnection.HTTP_NOT_FOUND:
			LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, null);
			return null;
		case HttpURLConnection.HTTP_CONFLICT: {
			log.error(agent.getName(), "NACOS-XXXX",
					"[sub-server-error] get server config being modified concurrently, dataId={}, group={}, tenant={}",
					dataId, group, tenant);
			throw new NacosException(NacosException.CONFLICT,
					"data being modified, dataId=" + dataId + ",group=" + group + ",tenant=" + tenant);
		}
		case HttpURLConnection.HTTP_FORBIDDEN: {
			log.error(agent.getName(), "NACOS-XXXX", "[sub-server-error] no right, dataId={}, group={}, tenant={}",
					dataId, group, tenant);
			throw new NacosException(result.code, result.content);
		}
		default: {
			log.error(agent.getName(), "NACOS-XXXX", "[sub-server-error]  dataId={}, group={}, tenant={}, code={}",
					dataId, group, tenant, result.code);
			throw new NacosException(result.code,
					"http error, code=" + result.code + ",dataId=" + dataId + ",group=" + group + ",tenant=" + tenant);
		}
		}
	}

```



我们看到这里通过http请求从服务端获取配置信息