## Spring IoC容器源码分析


![Spring继承体系](https://tva3.sinaimg.cn/large/005VwC5mly1gd5e9hl3m4j31300pwn0i.jpg)

入口为Spring-context 的 ApplicationContex

![1111](https://tvax1.sinaimg.cn/large/005VwC5mly1gd5eyjjozcj312i0ne0w7.jpg)

* ApplicationContex 继承了 ListableBeanFactory，这个 Listable 的意思就是，通过这个接口，我们可以获取多个Bean，大家看源码会发现，最顶层 BeanFactory 接口的方法都是获取单个 Bean 的。
* ApplicationContext 继承了 HierarchicalBeanFactory，Hierarchical 单词本身已经能说明问题了，也就是说我们可以在应用中起多个 BeanFactory，然后可以将各个 BeanFactory 设置为父子关系。
* AutowireCapableBeanFactory 这个名字中的 Autowire 大家都非常熟悉，它就是用来自动装配 Bean 用的，但是仔细看上图，ApplicationContext 并没有继承它，不过不用担心，不使用继承，不代表不可以使用组合，如果你看到 ApplicationContext 接口定义中的最后一个方法 getAutowireCapableBeanFactory() 就知道了。
* ConfigurableListableBeanFactory 也是一个特殊的接口，看图，特殊之处在于它继承了第二层所有的三个接口，而 ApplicationContext 没有。这点之后会用到。

源码分析
```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 前置准备，加同步锁，记录时间等 加载XML 校验等
			prepareRefresh();

      // 这步比较关键，这步完成后，配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，
      // 当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了，
      // 注册也只是将这些信息都保存到了注册中心(说到底核心是一个 beanName-> beanDefinition 的 map)
      //这步主要是对 xml文件中的bean定义进行解析，然后缓存到map中，但是没有初始换，仅仅是解析了标签
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		  // 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
			prepareBeanFactory(beanFactory);

			try {
        // 【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
         // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。】
         // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
         // 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
         //搜索哪些bean 实现了 BeanFactoryPostProcessor 接口
				postProcessBeanFactory(beanFactory);

				// 真正的执行了 实现接口的 bean 进行 模板方法调用
				invokeBeanFactoryPostProcessors(beanFactory);

        // 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
         // 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
         // 两个方法分别在 Bean 初始化之前和初始化之后得到执行。注意，到这里 Bean 还没初始化
         // 仅仅是注册 实现了 该类的接口，暂时不 调用方法
         //并且将
				registerBeanPostProcessors(beanFactory);

				   // 初始化当前 ApplicationContext 的 MessageSource，国际化
				initMessageSource();

				// 初始化当前 ApplicationContext 的事件广播器，这里也不展开了
				initApplicationEventMulticaster();

        // 从方法名就可以知道，典型的模板方法(钩子方法)，
         // 具体的子类可以在这里初始化一些特殊的 Bean（在初始化 singleton beans 之前）
         //classpathXml 类中没有具体实现，是一个空实现
				onRefresh();

					// 注册事件监听器，监听器需要实现 ApplicationListener 接口
				registerListeners();

        // 重点，重点，重点
         // 初始化所有的 singleton beans
         //（lazy-init 的除外）
         //在这里也会具体的调用 在 实现了 BeanPostProcessor的类的两个回调方法
				finishBeanFactoryInitialization(beanFactory);

				// 最后，广播事件，ApplicationContext 初始化完成
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

```


`prepareRefresh`前置准备代码
```java
protected void prepareRefresh() {
		// 记录时间
		this.startupDate = System.currentTimeMillis();
    //设置 上下文关闭表示为false
    this.closed.set(false);
    //设置激活状态
		this.active.set(true);

		if (logger.isDebugEnabled()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			}
			else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		//初始化 propertie文件 classpatXML 类中是空实现
		initPropertySources();

		//校验xml
		getEnvironment().validateRequiredProperties();

		// Store pre-refresh ApplicationListeners...
		if (this.earlyApplicationListeners == null) {
			this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
		}
		else {
			// Reset local application listeners to pre-refresh state.
			this.applicationListeners.clear();
			this.applicationListeners.addAll(this.earlyApplicationListeners);
		}

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}

```

#### 创建 Bean 容器，加载并注册 Bean

`obtainFreshBeanFactory`

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	  //关闭旧的 beanFactory，创建新的 BeanFactory，加载 Bean 定义、注册 Bean 等等
		//方法继承自 org.springframework.context.support.AbstractRefreshableApplicationContext
		refreshBeanFactory();
		//返回bean工厂
		return getBeanFactory();
}

```


```java
protected final void refreshBeanFactory() throws BeansException {
	//如果已经存在工厂
	if (hasBeanFactory()) {
   //进行对已经注册 bean  依赖信息等所有 都进行清空
	 // 如果 ApplicationContext 中已经加载过 BeanFactory 了，销毁所有 Bean，关闭 BeanFactory
	// 注意，应用中 BeanFactory 本来就是可以多个的，这里可不是说应用全局是否有 BeanFactory，而是当前
		destroyBeans();
    // 关闭工厂，实际代码为 设置了null 属性
		//就是下面的 设置为了null
		closeBeanFactory();
	}
	try {
		//创建新的工厂
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		//序列化 id
		beanFactory.setSerializationId(getId());
     // 设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用
		customizeBeanFactory(beanFactory);
    // 加载 Bean 到 BeanFactory 中
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

从ApplicationContext 的继承关系来看，是继承了BeanFactory，但是为啥。。又注入了一个 BeanFactory，而不是自己实现呢？？？


>看到这里的时候，我觉得读者就应该站在高处看 ApplicationContext 了，ApplicationContext 继承自 BeanFactory，但是它不应该被理解为 BeanFactory 的实现类，而是说其内部持有一个实例化的 BeanFactory（DefaultListableBeanFactory）。以后所有的 BeanFactory 相关的操作其实是委托给这个实例来处理的。

#### DefaultListableBeanFactory

![继承图](https://tva1.sinaimg.cn/large/005VwC5mly1gd8mv6b8g8j312g0o2n0o.jpg)

#### BeanDefinition

BeanDefinition就是我们所说的Spring 的 bean，我们在XML中定义的bean都会被转换为一个个BeanDefinition存在于SpringFactory中。

>BeanDefinition 中保存了我们的 Bean 信息，比如这个 Bean 指向的是哪个类、是否是单例的、是否懒加载、这个 Bean 依赖了哪些 Bean 等等。

每一个Bean配置，都是一个BeanDefinition配置

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

	// 我们可以看到，默认只提供 sington 和 prototype 两种，
	// 很多读者可能知道还有 request, session, globalSession, application, websocket 这几种，不过，它们属于基于 web 的扩展。
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;


	//暂时不知道
	int ROLE_APPLICATION = 0;
	int ROLE_SUPPORT = 1;
	int ROLE_INFRASTRUCTURE = 2;


	//设置父容器名称，应该是通过某Map，通过名称获取父容器的各种信息吧。
	// 设置父 Bean，这里涉及到 bean 继承，不是 java 继承。请参见附录的详细介绍
  // 一句话就是：继承父 Bean 的配置信息而已
	void setParentName(@Nullable String parentName);

	//获取
	@Nullable
	String getParentName();

	 // 设置 Bean 的类名称，将来是要通过反射来生成实例的
	void setBeanClassName(@Nullable String beanClassName);
	@Nullable
	String getBeanClassName();

	//Scop
	void setScope(@Nullable String scope);
	@Nullable
	String getScope();

	//设置是否懒加载
	void setLazyInit(boolean lazyInit);
	boolean isLazyInit();

	//设置依赖信息
	// 设置该 Bean 依赖的所有的 Bean，注意，这里的依赖不是指属性依赖(如 @Autowire 标记的)，
 // 是 depends-on="" 属性设置的值。
	void setDependsOn(@Nullable String... dependsOn);
	@Nullable
	String[] getDependsOn();

  //设置该bean是否可以被注入到其他bean中，只对根据类型注入有效，如果根据名称注入，即使是false，也是可以注入的
	void setAutowireCandidate(boolean autowireCandidate);
	boolean isAutowireCandidate();

	//设置是否是 主要的bean，根据类型注入时使用
	  // 主要的。同一接口的多个实现，如果不指定名字的话，Spring 会优先选择设置 primary 为 true 的 bean
	void setPrimary(boolean primary);
	boolean isPrimary();

	//xml 中设置的工厂方法
	// 如果该 Bean 采用工厂方法生成，指定工厂名称。对工厂不熟悉的读者，请参加附录
	// 一句话就是：有些实例不是用反射生成的，而是用工厂模式生成的
	void setFactoryBeanName(@Nullable String factoryBeanName);
	@Nullable
	String getFactoryBeanName();
	//设置指定工厂指定方法
	void setFactoryMethodName(@Nullable String factoryMethodName);
	@Nullable
	String getFactoryMethodName();

	// 获取指定构造器的参数
	ConstructorArgumentValues getConstructorArgumentValues();

	//是否有构造器参数
	default boolean hasConstructorArgumentValues() {
		return !getConstructorArgumentValues().isEmpty();
	}

	//获取 bean中的属性值，
	MutablePropertyValues getPropertyValues();
 bean中的属性值，	default boolean hasPropertyValues() {
		return !getPropertyValues().isEmpty();
	}

	//设置初始化的方法名
	void setInitMethodName(@Nullable String initMethodName);
	@Nullable
	String getInitMethodName();

	//设置销毁的方法名
	void setDestroyMethodName(@Nullable String destroyMethodName);
	@Nullable
	String getDestroyMethodName();

	/**
	 * Set the role hint for this {@code BeanDefinition}. The role hint
	 * provides the frameworks as well as tools with an indication of
	 * the role and importance of a particular {@code BeanDefinition}.
	 * @since 5.1
	 * @see #ROLE_APPLICATION
	 * @see #ROLE_SUPPORT
	 * @see #ROLE_INFRASTRUCTURE
	 */
	void setRole(int role);

	/**
	 * Get the role hint for this {@code BeanDefinition}. The role hint
	 * provides the frameworks as well as tools with an indication of
	 * the role and importance of a particular {@code BeanDefinition}.
	 * @see #ROLE_APPLICATION
	 * @see #ROLE_SUPPORT
	 * @see #ROLE_INFRASTRUCTURE
	 */
	int getRole();

	//描述
	void setDescription(@Nullable String description);
	@Nullable
	String getDescription();


	// Read-only attributes

	/**
	 * Return a resolvable type for this bean definition,
	 * based on the bean class or other specific metadata.
	 * <p>This is typically fully resolved on a runtime-merged bean definition
	 * but not necessarily on a configuration-time definition instance.
	 * @return the resolvable type (potentially {@link ResolvableType#NONE})
	 * @since 5.2
	 * @see ConfigurableBeanFactory#getMergedBeanDefinition
	 */
	ResolvableType getResolvableType();

	//是否是单例
	boolean isSingleton();

	//是否是 prototype类型
	boolean isPrototype();

  //是否是抽象类
	// 如果这个 Bean 是被设置为 abstract，那么不能实例化，
	// 常用于作为 父bean 用于继承，其实也很少用......
	boolean isAbstract();

	/**
	 * Return a description of the resource that this bean definition
	 * came from (for the purpose of showing context in case of errors).
	 */
	@Nullable
	String getResourceDescription();

	/**
	 * Return the originating BeanDefinition, or {@code null} if none.
	 * Allows for retrieving the decorated bean definition, if any.
	 * <p>Note that this method returns the immediate originator. Iterate through the
	 * originator chain to find the original BeanDefinition as defined by the user.
	 */
	@Nullable
	BeanDefinition getOriginatingBeanDefinition();

}
```

这个类总览看起来像是一个XML中bean定义中的所有东西了，这里只是记录了信息，真正的生成，不是在这里。






#### Spring 扩展点（实现接口）

1. `BeanPostProcessor` 实现该接口，主要有两个方法：`postProcessBeforeInitialization`和`postProcessAfterInitialization`，主要在bean初始化时候做一些操作。
2. `BeanFactoryPostProcessor` 实现该接口，主要是在容器初始化之后，进行一些操作
