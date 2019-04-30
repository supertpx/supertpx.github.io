---
title: 服务治理组件Eureka
date: 2019-04-29 09:28:28
categories:
- 框架
- 微服务-Spring Cloud
---

> Eurka是由Netflix开发的一套具有服务治理功能的组件，Spring Cloud在其基础上进行了二次封装，将其融入到了Spring Cloud微服务体系中。服务治理在任何一个微服务框架中都应是一个基础且重要的功能。

### Eureka的服务端和客户端

Eureka主要有两个重要的组成部分：Eureka Server和Eureka Client。顾名思义，Eureka Server，又称服务注册中心，其维护了一个ConcurrentHashMap对象registry，为一个双层map结构（ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>），registry外层的key为服务实例Instance注册时提供的AppName，内层的key为该Instance的Id。通过该对象，Eureka Server可以对外提供一个按服务名（AppName）分类组织的服务清单，该清单会通过心跳检测的机制去更新。Eureka Client提供了两个重要的功能：服务获取和服务注册（续约）。其中，服务获取可用于服务消费者，服务注册（续约）可用于服务提供者。

<!-- more -->

### Eureka Server

在一个基础的Spring Boot项目中，在pom.xml文件导入以下依赖：

{% codeblock pom.xml lang:xml %}
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
{% endcodeblock %}

然后在该应用的主类中加上@EnableEurekaServer注解，即可开启一个Eureka Server。如果需要对端口等行为进行自定义，可以在application.properties中配置如下：

{% codeblock application.properties lang:properties %}
spring.application.name=eureka-server
server.port=1001

eureka.instance.hostname=peer
#eureka.client.register-with-eureka=false
#eureka.client.fetch-registry=false
eureka.server.renewalPercentThreshold=0.49
eureka.client.serviceUrl.defaultZone=http://peer1:1002/eureka/
{% endcodeblock %}

其中eureka.client.serviceUrl.defaultZone一般为本机hostname+端口+/eureka/，本机已搭建了一个高可用的Eureka Server集群，分别有peer和peer1两个节点，两个节点的defaultZone互为对方地址。启动这两个节点，访问http://peer1:1002/eureka/ 或者 http://peer:1001/eureka/ ，可以见到如下界面：

{% asset_img server-web.png Eureka Server的web界面 %}

图中可以看出，本机已启动了两个Eureka Server节点，两个Eureka Client节点，以及一个服务消费者节点。先看Eureka Server节点。打开spring-cloud-netflix-eureka的jar包（以2.1.1版本为例），我们可以看到如下的目录结构：

{% asset_img server包结构.jpg Eureka Server的包结构 %}

可以看到Eureka Server定义了5个事件，其中三个与Eureka Client相关，分别是EurekaInstanceCanceledEvent（服务的取消）、EurekaInstanceRegisteredEvent（服务的注册）、EurekaInstanceRenewedEvent（服务的续约），EurekaRegistryAvailableEvent和EurekaServerStartedEvent则与Eureka Server自身启动和注册相关。

查看InstanceRegistry类，可以找到跟服务注册、取消、续约相关的代码片段如下：

{% codeblock InstanceRegistry.class lang:java %}
	@Override
	public void register(InstanceInfo info, int leaseDuration, boolean isReplication) {
		handleRegistration(info, leaseDuration, isReplication);
		super.register(info, leaseDuration, isReplication);
	}

	@Override
	public boolean cancel(String appName, String serverId, boolean isReplication) {
		handleCancelation(appName, serverId, isReplication);
		return super.cancel(appName, serverId, isReplication);
	}

	@Override
	public boolean renew(final String appName, final String serverId,
			boolean isReplication) {
		log("renew " + appName + " serverId " + serverId + ", isReplication {}"
				+ isReplication);
		List<Application> applications = getSortedApplications();
		for (Application input : applications) {
			if (input.getName().equals(appName)) {
				InstanceInfo instance = null;
				for (InstanceInfo info : input.getInstances()) {
					if (info.getId().equals(serverId)) {
						instance = info;
						break;
					}
				}
				publishEvent(new EurekaInstanceRenewedEvent(this, appName, serverId,
						instance, isReplication));
				break;
			}
		}
		return super.renew(appName, serverId, isReplication);
	}
{% endcodeblock %}

其中，服务注册和取消的时候，都会先通过Spring Context发布一个事件，通知网络中的其他Eureka Server节点，然后再执行相应的注册或取消动作。服务续约则会从已存储的服务列表中找到appName与serverId一致的那个服务实例，发布续约事件，然后执行续约动作。以注册动作为例，根据super.register(info, leaseDuration, isReplication)找到eureka-core包中的AbstractInstanceRegistry类，可以看到register动作的函数体为：

{% codeblock AbstractInstanceRegistry.class lang:java %}
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock();
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // Retain the last dirty timestamp without overwriting it, if there is already a lease
            if (existingLease != null && (existingLease.getHolder() != null)) {

                 ...

                // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
                // InstanceInfo instead of the server local copy.
                 ...

            } else {
                // The lease does not exist and hence it is a new registration
                synchronized (lock) {
                    if (this.expectedNumberOfClientsSendingRenews > 0) {
                        // Since the client wants to register it, increase the number of clients sending renews
                        this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                        updateRenewsPerMinThreshold();
                    }
                }
                logger.debug("No previous lease information found; it is new registration");
            }
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            gMap.put(registrant.getId(), lease);
            synchronized (recentRegisteredQueue) {
                recentRegisteredQueue.add(new Pair<Long, String>(
                        System.currentTimeMillis(),
                        registrant.getAppName() + "(" + registrant.getId() + ")"));
            }
            // This is where the initial state transfer of overridden status happens
            
             ...

            // If the lease is registered with UP status, set lease service up timestamp
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
             
             ...

        } finally {
            read.unlock();
        }
    }
{% endcodeblock %}

该函数中，Eureka Server维护了一个上文中提到的ConcurrentHashMap对象registry，以及recentRegisteredQueue和recentlyChangedQueue两个序列。在register动作发生时，Eureka Server首先会去检查registry对象，查看需要注册的InstanceInfo是否存在，如果存在，则修改对应的状态、时间等相关信息；如果不存在，则增加，并在recentRegisteredQueue和recentlyChangedQueue中登记。

服务取消动作和续约动作分别对应该类中的internalCancel和renew函数。

### Eureka Client

在微服务体系中，除去Eureka Server之外的服务，都具有Eureka Client的属性，因为它们要么是服务提供者，要么是服务消费者，或者两者都是。

#### 服务提供者

在pom.xml中导入以下依赖：

{% codeblock pom.xml lang:xml %}
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
{% endcodeblock %}

并在应用主类中添加@EnableDiscoveryClient注解，即可开启一个Eureka Client。如果需要对端口等行为进行自定义，可以在application.properties中配置如下：

{% codeblock application.properties lang:properties %}
spring.application.name=eureka-client
server.port=2001
eureka.client.serviceUrl.defaultZone=http://peer:1001/eureka/,http://peer1:1002/eureka/
#eureka.client.healthcheck.enabled=true
#eureka.instance.statusPageUrlPath=http://localhost:1001/info
#eureka.instance.healthCheckUrlPath=http://localhost:1001/health
{% endcodeblock %}

作为服务提供者，则需要提供接口供服务消费者来消费。本例中以rest请求的方式定义一个最简单的接口，示例如下：

{% codeblock EcController.java lang:java %}
package com.tpx.ms.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class EcController {
	@Autowired
	private DiscoveryClient discoveryClient;

	@GetMapping("/dc")
	public String dc() {
		String services = "Services: " + discoveryClient.getServices();
		System.out.println(services);
		return services;
	}
}
{% endcodeblock %}

通过注入DiscoveryClient对象，可以得到这个Eureka Client的相关信息。打开spring-cloud-netflix-eureka-client的jar包，可以看到如下的目录结构：

{% asset_img client包结构.png Eureka Client的包结构 %}

图中可以看到spring-cloud-netflix-eureka-client只是Spring Cloud的封装，图下的位置里还有一个eureka-client的jar包。查看DiscoveryClient类的构造方法，代码如下：

{% codeblock DiscoveryClient.class lang:java %}
    @Inject
    DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider) {

         ...

        if (config.shouldFetchRegistry()) {
            this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        if (config.shouldRegisterWithEureka()) {
            this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        logger.info("Initializing Eureka in region {}", clientConfig.getRegion());

        if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
            logger.info("Client configured to neither register nor query for data.");
            scheduler = null;
            heartbeatExecutor = null;
            cacheRefreshExecutor = null;
            eurekaTransport = null;
            instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

            // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
            // to work with DI'd DiscoveryClient
            DiscoveryManager.getInstance().setDiscoveryClient(this);
            DiscoveryManager.getInstance().setEurekaClientConfig(config);

            initTimestampMs = System.currentTimeMillis();
            logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                    initTimestampMs, this.getApplications().size());

            return;  // no need to setup up an network tasks and we are done
        }

        try {
            // default size of 2 - 1 each for heartbeat and cacheRefresh
            scheduler = Executors.newScheduledThreadPool(2,
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-%d")
                            .setDaemon(true)
                            .build());

            heartbeatExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            cacheRefreshExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            eurekaTransport = new EurekaTransport();
            scheduleServerEndpointTask(eurekaTransport, args);

             ...

        } catch (Throwable e) {
            throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
        }

        if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
            fetchRegistryFromBackup();
        }

        // call and execute the pre registration handler before all background tasks (inc registration) is started
        if (this.preRegistrationHandler != null) {
            this.preRegistrationHandler.beforeRegistration();
        }

        if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
            try {
                if (!register() ) {
                    throw new IllegalStateException("Registration error at startup. Invalid server response.");
                }
            } catch (Throwable th) {
                logger.error("Registration error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        }

        // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
        initScheduledTasks();

         ...

    }
{% endcodeblock %}
 
代码有部分精简，从这个函数可以看出，在Eureka Client初始化的过程中，会先对一系列参数进行检查，其中最重要的两个参数为fetchRegistry和registerWithEureka，默认为ture。参数检查完毕之后，会执行scheduleServerEndpointTask函数，该方法会根据参数给EurekaTransport对象赋值，用于传输到Eureka Server端；然后执行initScheduledTasks函数，部分代码如下：

{% codeblock DiscoveryClient.class lang:java %}
    private void initScheduledTasks() {
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }

        if (clientConfig.shouldRegisterWithEureka()) {
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

            // Heartbeat timer
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "heartbeat",
                            scheduler,
                            heartbeatExecutor,
                            renewalIntervalInSecs,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new HeartbeatThread()
                    ),
                    renewalIntervalInSecs, TimeUnit.SECONDS);

            // InstanceInfo replicator
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize

            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    
                     ...

                }
            };

             ...

            } else {
            logger.info("Not registering with Eureka server per configuration");
        }

    }
{% endcodeblock %}

该函数中有两个重要的if分支，按顺序分别对应服务获取、服务注册（续约）功能，需要注意的是，服务获取只有cacheRefresh这个定时任务，而服务注册（续约）则拥有heartbeat定时任务、InstanceInfoReplicator、StatusChangeListener等功能。在InstanceInfoReplicator实例中，有一个定时任务会执行register函数，该函数的作用是将client自身的instanceinfo以rest请求的方式发送给Server端，从而完成注册。heartbeat定时任务则利用心跳机制来进行服务续约。

#### 服务消费者

新建服务提供者的步骤也可用于新建服务消费者，不同的是，我们需要消费服务提供者提供的dc接口。在Spring Boot项目中定义一个interface用于访问dc接口，代码如下：

{% codeblock Finterface.java lang:java %}
package com.tpx.ms.restinterface;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

@FeignClient("eureka-client")
public interface Finterface {
	@GetMapping("/dc")
    String consumer();
}
{% endcodeblock %}

然后再定义一个Controller用于调用该interface，代码如下:

{% codeblock FController.java lang:java %}
package com.tpx.ms.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class FController {

	@Autowired
	com.tpx.ms.restinterface.Finterface Finterface;
	
	@GetMapping("/consumer")
    public String dc() {
        return Finterface.consumer();
    }
}
{% endcodeblock %}

经过以上对服务提供者的分析，可以得知，严格意义上的服务消费者，其实只需要开启服务获取的功能即可。在initScheduledTasks函数中，第一个服务获取的if分支中提供了一个cacheRefresh的定时任务，该任务会定时获取Eureka Server端的服务提供者列表，存储在本地并定期刷新。通过这个机制，我们可以实现客户端负载均衡。

### 其他

- 更多关于Eureka Server和Eureka Client的配置相关信息，可以查看EurekaServerConfigBean和EurekaClientConfigBean的源码；

- Eureka Client配置时，还有关于Region和Zone的配置，本文不再展开；

- 各节点的/info和/heath信息需配合spring-boot-starter-actuator使用。