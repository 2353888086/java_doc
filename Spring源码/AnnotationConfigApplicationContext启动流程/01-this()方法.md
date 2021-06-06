> Spring的入口

~~~java
	public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
		this();
		register(componentClasses);
		refresh();
	}

~~~

> Spring的加载流程





> UML图

![ss](E:\Z-资料\总结文档\文档图片\1622551462558.png)



> this()方法

~~~java
	先调用调用父类的
	public DefaultResourceLoader() {
		//初始化文件类加载器
        this.classLoader = ClassUtils.getDefaultClassLoader();
    }

	public AbstractApplicationContext() {
		//创建资源模式处理器：用来解析当前系统运行的时候需要用到的一些资源，如配置文件
		this.resourcePatternResolver = getResourcePatternResolver();
	}
   
	public GenericApplicationContext() {
		//初始化beanFactory，类型为DefaultListableBeanFactory
		this.beanFactory = new DefaultListableBeanFactory();
	}
	
	public AnnotationConfigApplicationContext() {
        // BeanDefinition读取器. BeanDefinition是描述bean注册的信息
		this.reader = new AnnotatedBeanDefinitionReader(this);
        // 创建BeanDefinition扫描器
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
~~~



> this.reader = new AnnotatedBeanDefinitionReader(this);

~~~java
	//AnnotationConfigApplicationContext间接继承了BeanDefinitionRegistry
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
		this(registry, getOrCreateEnvironment(registry));
	}

	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		//registry就是AnnotationConfigApplicationContext这个类的实例
        this.registry = registry;
         //条件解析器,
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		//后置处理器，提前往容器中注册一些必要的后置处理器
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}

~~~



> this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);

~~~java
	public ConditionEvaluator(@Nullable BeanDefinitionRegistry registry,
			@Nullable Environment environment, @Nullable ResourceLoader resourceLoader) {
		//ConditionContextImpl是ConditionEvaluator的内部类
		this.context = new ConditionContextImpl(registry, environment, resourceLoader);
	}

	public ConditionContextImpl(@Nullable BeanDefinitionRegistry registry,
				@Nullable Environment environment, @Nullable ResourceLoader resourceLoader) {

			this.registry = registry;
             // 获取beanFactory，类型为DefaultListableBeanFactory
			this.beanFactory = deduceBeanFactory(registry);
             // 从容器中获取environment，前面介绍过，容器中的environment的封装类是 StandardEnvironment
			this.environment = (environment != null ? environment : deduceEnvironment(registry));
             // 资源加载器. resourceLoader就是容器, 因为容器间接继承了ResourceLoader
			this.resourceLoader = (resourceLoader != null ? resourceLoader : deduceResourceLoader(registry));
        	//类加载器. 实际上就是获取beanFactory的类加载器。因为，容器当中的类加载器肯定要一致才行
			this.classLoader = deduceClassLoader(resourceLoader, this.beanFactory);
		}
~~~



>  AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry); //注入后置处理器

~~~java
	public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
		registerAnnotationConfigProcessors(registry, null);
	}

	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {
		// 获取容器中的beanFactory,类型为DefaultListableBeanFactory
		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
		if (beanFactory != null) {
            // 此时beanFactory的属性dependencyComparator首次为null，并对dependencyComparator进行设置。
            // AnnotationAwareOrderComparator继承了OrderComparator,
            // 因此可以对实现了Ordered接口、打上@Order或者@Priority注解的类进行排序。
            // 也就是说，在这里设置beanFactory中的orderComparator，以支持解析bean的排序功能。
			if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
				beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
			}
              // beanFactory初始化时，默认为SimpleAutowireCandidateResolver
              // private AutowireCandidateResolver autowireCandidateResolver = new SimpleAutowireCandidateResolver();
              // SimpleAutowireCandidateResolver implements AutowireCandidateResolver
       		 // ContextAnnotationAutowireCandidateResolver最主要的作用就是支持@Lazy注解的类的处理。
			if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
				beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
			}
		}

         //创建BeanDefinition的持有者的容器,盛放接下来将解析出来的后置处理器
		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

        // 获取并注册ConfigurationClassPostProcessor后置处理器
		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

        // 获取并注册AutowiredAnnotationBeanPostProcessor后置处理器
		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		 // 获取并注册 CommonAnnotationBeanPostProcessor 后置处理器
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		 // 获取并注册 PersistenceAnnotationBeanPostProcessor 后置处理器
		if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition();
			try {
				def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
						AnnotationConfigUtils.class.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
			}
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

         // 获取并注册 EventListenerMethodProcessor 后置处理器
		if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
		}

         // 获取并注册 DefaultEventListenerFactory 后置处理器
		if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
		}

		return beanDefs;
	}
~~~

> 注入后置处理器以后BeanFactory中的BeanDefinitionMpa中的 BeanDefinition

![1622899114593](E:\Z-资料\总结文档\文档图片\1622899114593.png)

> beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));

~~~java
	private static BeanDefinitionHolder registerPostProcessor(
			BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {

		definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
         //将BeanDefinition注册到registry里，主要是讲BeanDefinition放入到BeanFactory中的BeanDefinitionMpa的中
		registry.registerBeanDefinition(beanName, definition);
         //根据BeanDefinition创建BeanDefinition持有者并返回，放到上面创建的容器中
		return new BeanDefinitionHolder(definition, beanName);
	}
~~~



> reader的作用

~~~txt
1.创建并设置容器当中的Environment属性。即默认为StandardEnvironment类。
2.创建并设置容器当中的条件解析器,即ConditionEvaluator，其内部实际委托给内部类ConditionContextImpl
3.注册6个后置处理器到容器当中。注：仅是生成了后置处理器的BeanDefinition持有至 ，还并没有进行bean解析和后置处理的执行。
4.注册6个后置处理器到容器当中，但是BeanDefinitionMpa中只有5个BeanDefinition，PersistenceAnnotationBeanPostProcessor这个BeanDefinition并未放入Map中，因为jpaPresent是false（//TODO）
~~~





> this.scanner = new ClassPathBeanDefinitionScanner(this);



~~~java
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
		this(registry, true);
	}
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
		this(registry, useDefaultFilters, getOrCreateEnvironment(registry));
	}
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment) {

		this(registry, useDefaultFilters, environment,
				(registry instanceof ResourceLoader ? (ResourceLoader) registry : null));
	}

	//最终调用初始化方法
	public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) {

		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		this.registry = registry;

        //是否包括默认过滤器,useDefaultFilters默认是true
		if (useDefaultFilters) {
			registerDefaultFilters();
		}
		setEnvironment(environment);
		setResourceLoader(resourceLoader);
	}


~~~



> registerDefaultFilters() //为 @Component 注册默认筛选器，
>
> 并将隐式注册所有具有包括 @Component 元注释的原型注解，包括： @Repository  @Service  @Controller

~~~java
	protected void registerDefaultFilters() {
         // 扫描@Component注解的类
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
            // 扫描所有@ManageBean的类
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
            // 扫描所有@Named的类
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}

	注：@ManageBean和@Named的作用和@Component作用一致，没有添加默认扫描@Service、@Repository、@Controller，是因为这些注解都间接继承了@Component
~~~



> this()的作用

~~~txt
1.创建并设置容器当中的Environment属性。即默认为StandardEnvironment类。
2.创建并设置容器当中的条件解析器,即ConditionEvaluator，其内部实际委托给内部类ConditionContextImpl
3.注册6个后置处理器到容器当中。注：仅是生成了后置处理器的BeanDefinition持有至 ，还并没有进行bean解析和后置处理的执行。
4.BeanDefinitionMpa中只有5个BeanDefinition，PersistenceAnnotationBeanPostProcessor这个BeanDefinition并未放入Map中，因为jpaPresent是false（//TODO）
5.创建BeanDefinition扫描器，即：创建注册默认过滤器  @Component 注解，包含：@Repository  @Service  @Controller
~~~

