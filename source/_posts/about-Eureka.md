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
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

                // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
                // InstanceInfo instead of the server local copy.
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    registrant = existingLease.getHolder();
                }
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
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                                + "overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            // Set the status based on the overridden status rules
            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);

            // If the lease is registered with UP status, set lease service up timestamp
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
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


