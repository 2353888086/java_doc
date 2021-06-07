~~~java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			//准备工作：设置启动时间、是否激活标识位 初始化早期事件
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			// 获取DefaultListableBeanFactory，创建容器是在上面this()方法中实现的，如果是xml配置的，会在这里解析bean定义
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
             // 准备bean工厂，注册了部分类
			// 准备bean和非bean
			// BeanFactory各种功能的填充，比如对@Qualifier和@Autowired注解的支持
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 扩展点，具体功能由子类实现
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				// 激活各种BeanFactory处理器
                 //注册bean工厂后置处理器，并解析java代码配置bean定义
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				// 注册拦截Bean创建的后处理器，这里只是注册，真正的调用在getBean的时候
                 //  注册bean后置处理器，并不会执行后置处理器，在后面实例化的时候执行
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				// 国际化处理
				// 为上下文初始化Message源，即不同语音的消息体
				initMessageSource();

				// Initialize event multicaster for this context.
				// 初始化应用消息广播器，并初始化"applicationEventMulticaster"bean
                 //  初始化事件监听多路广播器
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				// 模版方法，由子类扩展
                 // 待子类实现，springBoot在这里实现创建内置的tomact容器
				onRefresh();

				// Check for listener beans and register them.
				// 在所有注册的bean中查找Listener bean,注册到消息广播器中
                  // 注册监听器
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				// 对非惰性的单例进行初始化
				// 一般情况下单例都会在这里就初始化了，除非指定了惰性加载
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				// 完成刷新过程，通知生命周期处理器lifecycleProcessor刷新过程，
				// 同时发出ContextRefreshedEvent时间通知别人
                 // 广播事件，ApplicationContext初始化完成，springColud有实现该方法
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
~~~

