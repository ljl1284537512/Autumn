## Dubbo 服务发布



>  发布过程 : 解析配置文件，IP地址和端口号暴露出去；

> Dubbo 的配置文件dubbo.xsd  是基于Spring进行扩展的，Spring提供了一种机制进行标签扩展 - NamesapceHandler，BeanDefinitionParse; 通过这两个接口去实现自定义扩展；

1. 加载/META-INF/spring.handles文件；

2. DubboNamespaceHandler文件内容如下：

   ```java
   public class DubboNamespaceHandler extends NamespaceHandlerSupport {
   
       static {
           Version.checkDuplicate(DubboNamespaceHandler.class);
       }
   
       @Override
       public void init() {
           registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
           registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
   				...
       }
   
   }
   ```

3. Spring在提供的NameHandler机制，（在什么时候调用？）在调用init方法时，配置解析对应标签的解析器；

> Spring将解析bean之后的数据保存到对应的对象中如ModuleConfig, ApplicationConfig...



#### ServiceBean的实现

首先看ServiceBean的定义：

```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean,ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware,ApplicationEventPublisherAware
```

- InitializingBean : bean初始化的时候调用afterPropertiesSet方法；
- DisposableBean : bean销毁时执行其destroy方法；

ServiceBean在初始化之后，调用其afterProperties方法：

```java
public void afterPropertiesSet() throws Exception {
  	...
    if (!supportedApplicationListener) {
      	export();
    }
}
```

export方法实现如下：

```java
    @Override
    public void export() {
      	// 发布服务
        super.export();
        // 发布服务暴露事件
        publishExportEvent();
    }
```

最终执行ServiceConfig的export方法(ServiceBean继承自ServiceConfig)

```java
    public synchronized void export() {
        checkAndUpdateSubConfigs();
        if (!shouldExport()) {
            return;
        }
        if (shouldDelay()) {// 需要延迟暴露服务的情况
            delayExportExecutor.schedule(this::doExport, delay, TimeUnit.MILLISECONDS);
        } else {
            doExport();
        }
    }
```

doExport方法实现如下：

```java
    protected synchronized void doExport() {
        if (unexported) {
            throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
        }
        if (exported) { // 已经暴露过了，就不需要重复暴露了
            return;
        }
        exported = true;

        if (StringUtils.isEmpty(path)) {
          	// 路径名 = 接口名
            path = interfaceName;
        }
        String uniqueServiceName = getUniqueServiceName();
        ProviderModel providerModel = new ProviderModel(uniqueServiceName, ref, interfaceClass);
        ApplicationModel.initProviderModel(uniqueServiceName, providerModel);
        doExportUrls();
    }
```

doExportUrls();实现如下：

```java
    private void doExportUrls() {
      	// dubbo的配置文件中可以配置多个注册中心
      	// <dubbo:registry/>
      	// <dubbo:registry/>
        List<URL> registryURLs = loadRegistries(true);
				// dubbo支持多协议发布
      	// <dubbo:protocol/>
        for (ProtocolConfig protocolConfig : protocols) {
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```

> 启动一个服务需要做什么？调用注册中心发布服务到zk,启动一个Netty服务

doExportUrlsFor1Protocol方法实现如下：

```java
    private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
        String name = protocolConfig.getName();
        if (StringUtils.isEmpty(name)) {
            name = Constants.DUBBO;
        }

        Map<String, String> map = new HashMap<String, String>();
        map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
        appendRuntimeParameters(map);
        appendParameters(map, application);
        appendParameters(map, module);
        appendParameters(map, provider, Constants.DEFAULT_KEY);
        appendParameters(map, protocolConfig);
        appendParameters(map, this);
      	// 处理methods相关...
  			// end of methods for
        }

        if (ProtocolUtils.isGeneric(generic)) {
            map.put(Constants.GENERIC_KEY, generic);
            map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
        } else {
            String revision = Version.getVersion(interfaceClass, version);
            if (revision != null && revision.length() > 0) {
                map.put("revision", revision);
            }

            String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
            if (methods.length == 0) {
                logger.warn("No method found in service interface " + interfaceClass.getName());
                map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
            } else {
                map.put(Constants.METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
            }
        }
        if (!ConfigUtils.isEmpty(token)) {
            if (ConfigUtils.isDefault(token)) {
                map.put(Constants.TOKEN_KEY, UUID.randomUUID().toString());
            } else {
                map.put(Constants.TOKEN_KEY, token);
            }
        }
        // export service
        String contextPath = protocolConfig.getContextpath();
        if (StringUtils.isEmpty(contextPath) && provider != null) {
            contextPath = provider.getContextpath();
        }
				// 1. 获取host
        String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
			  // 2. 获取端口号
        Integer port = this.findConfigedPorts(protocolConfig, name, map);
				// 3. 将map中的数据转换成url, map中的数据就是interface,delay,application,side...等
        URL url = new URL(name, host, port, (StringUtils.isEmpty(contextPath) ? "" : contextPath + "/") + path, map);

        if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .hasExtension(url.getProtocol())) {
            url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
        }
				// 4. 从url中拿出scope
        String scope = url.getParameter(Constants.SCOPE_KEY);
        // don't export when none is configured
        if (!Constants.SCOPE_NONE.equalsIgnoreCase(scope)) {

            // 5.1 如果配置的scope不是remote，就直接发布服务到本地
            if (!Constants.SCOPE_REMOTE.equalsIgnoreCase(scope)) {
                exportLocal(url);
            }
            // 5.2 如果配置的scope不是local, 就直接发布到远程 
            if (!Constants.SCOPE_LOCAL.equalsIgnoreCase(scope)) {
                if (logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }
                if (CollectionUtils.isNotEmpty(registryURLs)) {
                    for (URL registryURL : registryURLs) { // 多注册中心发布
                        url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                        URL monitorUrl = loadMonitor(registryURL);
                        if (monitorUrl != null) {
                            url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                        }
                        
                        // For providers, this is used to enable custom proxy to generate invoker
                        String proxy = url.getParameter(Constants.PROXY_KEY);
                        if (StringUtils.isNotEmpty(proxy)) {
                            registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
                        }
												// 
                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
												// protocol = Protocol$Adaptive
                        // protocol.export内部实现为：
                      //ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("registry")
                        Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
                    }
                } else {
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
                /**
                 * @since 2.7.0
                 * ServiceData Store
                 */
                MetadataReportService metadataReportService = null;
                if ((metadataReportService = getMetadataReportService()) != null) {
                    metadataReportService.publishProvider(url);
                }
            }
        }
        this.urls.add(url);
    }
```

服务暴露的核心点是：

```java
// protocol = Protocol$Adaptive
// protocol.export内部实现为：                    //ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("registry")
Exporter<?> exporter = protocol.export(wrapperInvoker);
```

所以最终调用的是RegistryProtocol.export方法：

```java
    @Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        URL registryUrl = getRegistryUrl(originInvoker);
        // url to export locally
        URL providerUrl = getProviderUrl(originInvoker);

        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

        providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
        //本地暴露
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

        // url to registry
        final Registry registry = getRegistry(originInvoker);
        final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
        ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
                registryUrl, registeredProviderUrl);
        //to judge if we need to delay publish
        boolean register = registeredProviderUrl.getParameter("register", true);
        if (register) {
            register(registryUrl, registeredProviderUrl);
            providerInvokerWrapper.setReg(true);
        }

        // Deprecated! Subscribe to override rules in 2.6.x or before.
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

        exporter.setRegisterUrl(registeredProviderUrl);
        exporter.setSubscribeUrl(overrideSubscribeUrl);
        //Ensure that a new exporter instance is returned every time export
        return new DestroyableExporter<>(exporter);
    }
```

#### 本地暴露服务

doLocalExport方法内部实现如下：

```java
    private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
        String key = getCacheKey(originInvoker);

        return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
          	// 封装Invoker , 委托
            Invoker<?> invokerDelegete = new InvokerDelegate<>(originInvoker, providerUrl);
          	// 该protocol是通过set方法注入的 
          return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
        });
    }
```



#### 远程暴露服务

