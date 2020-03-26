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

		// 初始化 propertie文件 classpatXML 类中是空实现
		initPropertySources();

		// Validate that all properties marked as required are resolvable:
		// see ConfigurablePropertyResolver#setRequiredProperties
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


#### Spring 扩展点（实现接口）

1. `BeanPostProcessor` 实现该接口，主要有两个方法：`postProcessBeforeInitialization`和`postProcessAfterInitialization`，主要在bean初始化时候做一些操作。
2. `BeanFactoryPostProcessor` 实现该接口，主要是在容器初始化之后，进行一些操作
3. `ApplicationListener` 实现该接口，在
