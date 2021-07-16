# spring-cloud-openFeign源码解析
## 一、spring-cloud-openFeign的使用
### 1.启动类中增加@EnableFeignClients注解
```java
@SpringBootApplication
@EnableFeignClients
public class SpringcloudCApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringcloudCApplication.class, args);
    }
}
```
### 2.定义feign客户端
```java
@FeignClient(name = "test", url = "http://127.0.0.1:8080",path = "test")
public interface TestFeign {
    @RequestMapping(path = "/api/v1/getUser", method = RequestMethod.POST)
    Map<String,Object> getUser(@RequestBody Map<String,Object> params);
}
```
### 3.使用
```java
@RestController
@RequestMapping("/test")
public class TestController {
    @Autowired
    private TestFeign testFeign;
    
    @PostMapping("getUser")
    public Map<String, Object> checkUserOpenApp(){
        Map<String,Object> params = new HashMap<>() ;
        return testFeign.getUser(params);
    }
}
```
## 二、工作原理
### 1.客户端的声明
主要通过@EnableFeignClients及@FeignClient声明feign客户端
- @FeignClient
  用于描述客户端信息
- @EnableFeignClients
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {

	String[] value() default {};
	
	String[] basePackages() default {};

	Class<?>[] basePackageClasses() default {};

	Class<?>[] defaultConfiguration() default {};

	Class<?>[] clients() default {};

}
```
该注解核心是通过引入FeignClientsRegistrar.class进行依赖动态的注入
### 2.客户端的注册
通过@EnableFeignClients注解，引入FeignClientsRegistrar(feign客户端注册器)，通过扫描配置的包路径及@FeignCkient注解信息进行依赖注入。

2.1 registerBeanDefinitions方法 
  
通过实现ImportBeanDefinitionRegistrar#registerBeanDefinitions()方法，进行bean注册(包括feign客户端及feign配置类)，
```java
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
      //注册配置到容器 registry
      registerDefaultConfiguration(metadata, registry);
      //注册所发现的各个feign客户端到到容器registry
      registerFeignClients(metadata, registry);
	}
```
- 配置信息的注册
```java
    /**
	 * 处理，注册{@link EnableFeignClients#defaultConfiguration()}配置信息
	 * @param metadata
	 * @param registry
	 */
	private void registerDefaultConfiguration(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
		//获取@EnableFeignClients的注解信息
		Map<String, Object> defaultAttrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName(), true);
		if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
			String name;
			if (metadata.hasEnclosingClass()) {
				name = "default." + metadata.getEnclosingClassName();
			}
			else {
				name = "default." + metadata.getClassName();
			}
			//注册@EnableFeignClients中的defaultConfiguration配置类bean对象
			registerClientConfiguration(registry, name, defaultAttrs.get("defaultConfiguration"));
		}
	}

    /**
     * 注册{@link FeignClient}及{@link EnableFeignClients}中feign的配置类
     * @param registry
     * @param name
     * @param configuration
     */
    private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name, Object configuration) {
        //加载FeignClientSpecification bean,生成BeanDefinitionBuilder
        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(FeignClientSpecification.class);
        builder.addConstructorArgValue(name);
        builder.addConstructorArgValue(configuration);
        //进行配置bean注册
        registry.registerBeanDefinition(name + "." + FeignClientSpecification.class.getSimpleName(),
        builder.getBeanDefinition());
    }
```

- registerFeignClients方法
```java
    public void registerFeignClients(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {

		LinkedHashSet<BeanDefinition> candidateComponents = new LinkedHashSet<>();
		/**
		 * 获取注解{@link EnableFeignClients 的注解属性}
		 */
		Map<String, Object> attrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName());
		//根据注解获取feign客户端，根据客户端信息获生成BeanDefinition对象，用于注册到spring容器
		//判断有没有在@EnableFeignClients 定义clients客户端，
		final Class<?>[] clients = attrs == null ? null : (Class<?>[]) attrs.get("clients");
		if (clients == null || clients.length == 0) {
			ClassPathScanningCandidateComponentProvider scanner = getScanner();
			//通过basePackages 及FeignClient获取feign客户端
			scanner.setResourceLoader(this.resourceLoader);
			scanner.addIncludeFilter(new AnnotationTypeFilter(FeignClient.class));
			Set<String> basePackages = getBasePackages(metadata);
			for (String basePackage : basePackages) {
				candidateComponents.addAll(scanner.findCandidateComponents(basePackage));
			}
		}
		else {
			for (Class<?> clazz : clients) {
				candidateComponents.add(new AnnotatedGenericBeanDefinition(clazz));
			}
		}
		//循环BeanDefinition集合，进行注册
		for (BeanDefinition candidateComponent : candidateComponents) {
			if (candidateComponent instanceof AnnotatedBeanDefinition) {
				// verify annotated class is an interface
				AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
				AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
				//判断是不是@feignClient是否声明在接口上
				Assert.isTrue(annotationMetadata.isInterface(), "@FeignClient can only be specified on an interface");

				Map<String, Object> attributes = annotationMetadata
						.getAnnotationAttributes(FeignClient.class.getCanonicalName());
				//生成feign客户端bean名称
				String name = getClientName(attributes);
				//注册@feignClient中的configuration配置类bean对象
				registerClientConfiguration(registry, name, attributes.get("configuration"));
				//注册feign客户端bean
				registerFeignClient(registry, annotationMetadata, attributes);
			}
		}
	}

    /**
     * 注册feign客户端Bean（核心方法）
     * @param registry
     * @param annotationMetadata
     * @param attributes
     */
    private void registerFeignClient(BeanDefinitionRegistry registry, AnnotationMetadata annotationMetadata,
        Map<String, Object> attributes) {
        String className = annotationMetadata.getClassName();
        Class clazz = ClassUtils.resolveClassName(className, null);
        ConfigurableBeanFactory beanFactory = registry instanceof ConfigurableBeanFactory
        ? (ConfigurableBeanFactory) registry : null;
        String contextId = getContextId(beanFactory, attributes);
        String name = getName(attributes);
        //FeignClientFactoryBean对象，进行FeignClientFactoryBean构造
        FeignClientFactoryBean factoryBean = new FeignClientFactoryBean();
        factoryBean.setBeanFactory(beanFactory);
        factoryBean.setName(name);
        factoryBean.setContextId(contextId);
        factoryBean.setType(clazz);
        factoryBean.setRefreshableClient(isClientRefreshEnabled());
        BeanDefinitionBuilder definition = BeanDefinitionBuilder.genericBeanDefinition(clazz, () -> {
          factoryBean.setUrl(getUrl(beanFactory, attributes));
          factoryBean.setPath(getPath(beanFactory, attributes));
          factoryBean.setDecode404(Boolean.parseBoolean(String.valueOf(attributes.get("decode404"))));
          Object fallback = attributes.get("fallback");
          if (fallback != null) {
            factoryBean.setFallback(fallback instanceof Class ? (Class<?>) fallback
            : ClassUtils.resolveClassName(fallback.toString(), null));
          }
          Object fallbackFactory = attributes.get("fallbackFactory");
          if (fallbackFactory != null) {
            factoryBean.setFallbackFactory(fallbackFactory instanceof Class ? (Class<?>) fallbackFactory
            : ClassUtils.resolveClassName(fallbackFactory.toString(), null));
          }
          //factoryBean生成feign客户端对象（动态代理）
          return factoryBean.getObject();
        });
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
        definition.setLazyInit(true);
        validate(attributes);

        AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
        beanDefinition.setAttribute(FactoryBean.OBJECT_TYPE_ATTRIBUTE, className);
        beanDefinition.setAttribute("feignClientsRegistrarFactoryBean", factoryBean);

        // has a default, won't be null
        boolean primary = (Boolean) attributes.get("primary");

        beanDefinition.setPrimary(primary);

        String[] qualifiers = getQualifiers(attributes);
        if (ObjectUtils.isEmpty(qualifiers)) {
        qualifiers = new String[] { contextId + "FeignClient" };
        }

        BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className, qualifiers);
        BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
        registerOptionsBeanDefinition(registry, contextId);
    }
```
2.2 FeignClientFactoryBean创建Bean实例

FeignClientFactoryBean通过实现FactoryBean#getObject()方法进行Bean实例创建
```java
    <T> T getTarget() {
		/**
		 * FeignContext 通过自动装配，在{@link FeignAutoConfiguration#feignContext()}中进行配置
		 */
		FeignContext context = beanFactory != null ? beanFactory.getBean(FeignContext.class)
				: applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);
		/**
		 * 没有url信息，一般指通过注册中心进行调用，例如@FeignClient(name = "test",path="hello")，url->http://test/hello
		 */
		if (!StringUtils.hasText(url)) {
			if (url != null && LOG.isWarnEnabled()) {
				LOG.warn("The provided URL is empty. Will try picking an instance via load-balancing.");
			}
			else if (LOG.isDebugEnabled()) {
				LOG.debug("URL not provided. Will use LoadBalancer.");
			}
			if (!name.startsWith("http")) {
				url = "http://" + name;
			}
			else {
				url = name;
			}
			url += cleanPath();
			//生成负载对象
			return (T) loadBalance(builder, context, new HardCodedTarget<>(type, name, url));
		}
		/**
		 * url有值 例如@FeignClient(name = "test",url="127.0.0.1:8080",path="hello")，url->http://127.0.0.1:8080/hello
		 */
		if (StringUtils.hasText(url) && !url.startsWith("http")) {
			url = "http://" + url;
		}
		String url = this.url + cleanPath();
		Client client = getOptional(context, Client.class);
		if (client != null) {
			//负载均衡配置
			if (client instanceof FeignBlockingLoadBalancerClient) {
				client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
			}
			if (client instanceof RetryableFeignBlockingLoadBalancerClient) {
				client = ((RetryableFeignBlockingLoadBalancerClient) client).getDelegate();
			}
			builder.client(client);
		}
		Targeter targeter = get(context, Targeter.class);
		//生成对象
		return (T) targeter.target(this, builder, context, new HardCodedTarget<>(type, name, url));
	}
```
- 如果未指定url，则根据client的name来拼接url，并开启负载均衡 
- 如果指定了URL，没有指定client，那么就根据url来调用，相当于直连，没有负载均衡。
### 3.feign的工作原理
3.1 Feign接口的动态代理

调用Feign的target方法，生成代理对象,ReflectiveFeign构造函数有三个参数：
- ParseHandlersByName 将builder所有参数进行封装，并提供解析接口方法的逻辑
- InvocationHandlerFactory 默认值是InvocationHandlerFactory.Default,通过java动态代理的InvocationHandler实现
- QueryMapEncoder 接口参数注解@QueryMap时，参数的编码器，默认值QueryMapEncoder.DefaultReflectiveFeign 生成动态代理对象。
  
(1)ReflectiveFeign#newInstance
```java
    public <T> T newInstance(Target<T> target) {
        //为每个方法创建一个SynchronousMethodHandler对象，并放在 Map 里面。
        //targetToHandlersByName是构造器传入的ParseHandlersByName对象，根据target对象生成MethodHandler映射
        Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
        Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
        List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
        //遍历接口所有方法，构建Method->MethodHandler的映射
        for (Method method : target.type().getMethods()) {
            if (method.getDeclaringClass() == Object.class) {
            continue;
          } else if (Util.isDefault(method)) {
            //如果是 default 方法，说明已经有实现了，用 DefaultHandler接口default方法的Handler
            DefaultMethodHandler handler = new DefaultMethodHandler(method);
            defaultMethodHandlers.add(handler);
            methodToHandler.put(method, handler);
          } else {
            //否则就用上面的 SynchronousMethodHandler
            methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
            }
          }
          // 创建动态代理，factory 是 InvocationHandlerFactory.Default，创建出来的是 ReflectiveFeign.FeignInvocationHanlder，也就是说后续对方法的调用都会进入到该对象的 inovke 方法。
          InvocationHandler handler = factory.create(target, methodToHandler);
          // 创建动态代理对象
          T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
          new Class<?>[] {target.type()}, handler);
          //将default方法直接绑定到动态代理上
          for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
            defaultMethodHandler.bindTo(proxy);
          }
        return proxy;
      }
```
方法执行的逻辑，在MethodHandler具体的执行逻辑之中。SynchronousMethodHandler和DefaultMethodHandler实现了、
InvocationHandlerFactory.MethodHandler接口，动态代理对象调用方法时，如果是default方法，会直接调用接口方法，
因为这里将接口的default方法绑定到动态代理对象上了，其他方法根据方法签名找到SynchronousMethodHandler对象，调用其invoke方法。

(2)SynchronousMethodHandler#invoke()
```java
public Object invoke(Object[] argv) throws Throwable {
    //根据参数生成请求模板
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Options options = findOptions(argv);
    //重试器处理接口重试机制
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        //解码并执行请求
        return executeAndDecode(template, options);
      } catch (RetryableException e) {
        try {
          retryer.continueOrPropagate(e);
        } catch (RetryableException th) {
          Throwable cause = th.getCause();
          if (propagationPolicy == UNWRAP && cause != null) {
            throw cause;
          } else {
            throw th;
          }
        }
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
```
```java
/**
 * 解码，执行请求
 */
Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    //执行各个拦截器拦截器
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      response = client.execute(request, options);
      // ensure the request is set. TODO: remove in Feign 12
      response = response.toBuilder()
          .request(request)
          .requestTemplate(template)
          .build();
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);


    if (decoder != null)
      return decoder.decode(response, metadata.returnType());

    CompletableFuture<Object> resultFuture = new CompletableFuture<>();
    asyncResponseHandler.handleResponse(resultFuture, metadata.configKey(), response,
        metadata.returnType(),
        elapsedTime);

    try {
      if (!resultFuture.isDone())
        throw new IllegalStateException("Response handling not done");

      return resultFuture.join();
    } catch (CompletionException e) {
      Throwable cause = e.getCause();
      if (cause != null)
        throw cause;
      throw e;
    }
  }
```