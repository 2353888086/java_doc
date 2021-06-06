> register(annotatedClasses);

~~~java
	public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		
         this();
        //传入的配置类annotatedClasses，生成BeanDefinition，然后将BeanDefinition注册到DefaultListableBeanFactory 类型的对象 beanFactory 中
		register(annotatedClasses);
		refresh();
	}


	public void register(Class<?>... annotatedClasses) {
		Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
		this.reader.register(annotatedClasses);
	}


	public void register(Class<?>... annotatedClasses) {
        //循环处理配置类，就算是没有配置注解是不会报错的
		for (Class<?> annotatedClass : annotatedClasses) {
			registerBean(annotatedClass);
		}
	}

	public void registerBean(Class<?> annotatedClass) {
		doRegisterBean(annotatedClass, null, null, null);
	}

	<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {

        //根据配置类的Class创建，AnnotatedGenericBeanDefinition（注解的BeanDefinition）
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
        //根据@Conditional这个直接判断是否需要跳过，（需要类实现Condition的matches(ConditionContext context, AnnotatedTypeMetadata metadata)方法）
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}

        //设置实例，默认是null
		abd.setInstanceSupplier(instanceSupplier);
        //解析类的作用域，默认是Singleton单例
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
         //将类的作用域添加到AnnotatedGenericBeanDefinition的数据结构中
		abd.setScope(scopeMetadata.getScopeName());
        
        //获取beanName，name为空，调用this.beanNameGenerator.generateBeanName(abd, this.registry)获取beanName
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
        
		// 处理配置类中的通用注解，即Lazy、DependsOn、Primary和Role等，将处理的结果放到AnnotatedGenericBeanDefinition的数据结构中
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
        //qualifiers是默认为null，
        //这个的作用就是在qualifiers不为空的时候，
        //依次判断了注解当中是否包含了Primary、Lazy、qualifier，如果包含就设置AnnotatedGenericBeanDefinition对应的属性
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				if (Primary.class == qualifier) {
					abd.setPrimary(true);
				}
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true);
				}
				else {
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}
        
        //定制的BeanDefinition不为空，将AnnotatedGenericBeanDefinition放入其中
		for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
			customizer.customize(abd);
		}

        //根据AnnotatedGenericBeanDefinition创建BeanDefinitionHolder，
		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
        //根据@Scope(proxyMode = ScopedProxyMode.INTERFACES)判断是否需要创建动态代理
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
         // 将BeanDefinition注册到BeanDefinitionMap中
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}

	public static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd) {
		processCommonDefinitionAnnotations(abd, abd.getMetadata());
	}

	// 检查通用的注解，将存在的注解添加到AnnotatedBeanDefinition中
	static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
		AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
		// 判断Lazy
        if (lazy != null) {
			abd.setLazyInit(lazy.getBoolean("value"));
		}// 判断metadata 和 Lazy
		else if (abd.getMetadata() != metadata) {
			lazy = attributesFor(abd.getMetadata(), Lazy.class);
			if (lazy != null) {
				abd.setLazyInit(lazy.getBoolean("value"));
			}
		}
		// 判断Primary
		if (metadata.isAnnotated(Primary.class.getName())) {
			abd.setPrimary(true);
		}
        // 判断DependsOn
		AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
		if (dependsOn != null) {
			abd.setDependsOn(dependsOn.getStringArray("value"));
		}
		// 判断Role
		AnnotationAttributes role = attributesFor(metadata, Role.class);
		if (role != null) {
			abd.setRole(role.getNumber("value").intValue());
		}
        // 判断Description
		AnnotationAttributes description = attributesFor(metadata, Description.class);
		if (description != null) {
			abd.setDescription(description.getString("value"));
		}
	}




     public static void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) throws 			BeanDefinitionStoreException {
            String beanName = definitionHolder.getBeanName();
         	// 获取到的是配置类的BeanDefinition
            registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
            String[] aliases = definitionHolder.getAliases();
            if (aliases != null) {
                String[] var4 = aliases;
                int var5 = aliases.length;

                for(int var6 = 0; var6 < var5; ++var6) {
                    String alias = var4[var6];
                    registry.registerAlias(beanName, alias);// 注册别名
                }
            }

        }

    public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException {
        Assert.hasText(beanName, "Bean name must not be empty");
        Assert.notNull(beanDefinition, "BeanDefinition must not be null");
        if (beanDefinition instanceof AbstractBeanDefinition) {
            try {
                /**
                 * 主要是对于AbstractBeanDefinition属性中的methodOverrides校验，
                 * 校验methodOverrides是否与工厂方法并存或者methodOverrides对应的方法根本不存在
                 */
                ((AbstractBeanDefinition)beanDefinition).validate();
            } catch (BeanDefinitionValidationException var8) {
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName, "Validation of bean definition failed", var8);
            }
        }

        //根据名字判断beanDefinitionMap是否存在BeanDefinition
        BeanDefinition existingDefinition = (BeanDefinition)this.beanDefinitionMap.get(beanName);
        if (existingDefinition != null) {
           // 如果对应的beanName已经注册且在配置中配置了bean不允许被覆盖，则抛出异常。
            if (!this.isAllowBeanDefinitionOverriding()) {
                throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
            }

            if (existingDefinition.getRole() < beanDefinition.getRole()) {
                if (this.logger.isInfoEnabled()) {
                    this.logger.info("Overriding user-defined bean definition for bean '" + beanName + "' with a framework-generated bean definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
                }
            } else if (!beanDefinition.equals(existingDefinition)) {
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Overriding bean definition for bean '" + beanName + "' with a different definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
                }
            } else if (this.logger.isTraceEnabled()) {
                this.logger.trace("Overriding bean definition for bean '" + beanName + "' with an equivalent definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
            }

            this.beanDefinitionMap.put(beanName, beanDefinition);
        } else {
            if (this.hasBeanCreationStarted()) {
                // 因为beanDefinitionMap是全局变量，这里定会存在并发访问的情况
                synchronized(this.beanDefinitionMap) {
                    this.beanDefinitionMap.put(beanName, beanDefinition);
                    List<String> updatedDefinitions = new ArrayList(this.beanDefinitionNames.size() + 1);
                    updatedDefinitions.addAll(this.beanDefinitionNames);
                    updatedDefinitions.add(beanName);
                    this.beanDefinitionNames = updatedDefinitions;
                    this.removeManualSingletonName(beanName);
                }
            } else {
                // 注册beanDefinition
                this.beanDefinitionMap.put(beanName, beanDefinition);
                 // 记录beanName
                this.beanDefinitionNames.add(beanName);
                this.removeManualSingletonName(beanName);
            }

            this.frozenBeanDefinitionNames = null;
        }

        if (existingDefinition != null || this.containsSingleton(beanName)) {
            // 重置所有beanName对应的缓存
            this.resetBeanDefinition(beanName);
        }

    }

~~~



> register(annotatedClasses)的作用

~~~txt
register(annotatedClasses)方法主要是将传入的配置类加入到BeanDefinitionMap中，但是配置类中配置的包扫描路径下的bean并没有添加到BeanDefinitionMap中
~~~

