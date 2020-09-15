### Spring ioc 启动过程

#### 核心方法

基于 `ClassPathXMLApplicationContext`

refresh()--》进行上下文刷新，因为是可以重建上线文，因此是refresh

##### refresh过程
1. 锁（startupShutdownMonitor）
  - 防止刚刷新，就被销毁的尴尬。
2. prepareRefresh（） 准备工作
  - 主要为准备工作，记录启动时间，处理配置文件占位符
  详情：准备工作
    - 记录启动时间、设置closed = false active = true
    - 初始化properties，处理占位符
    - 校验XML文件
    - 创建Events set
3. ConfigurableListbleBeanFactory beanFactory = obtainFreshBeanFactory()
  - 这步主要是将XML中的bean定义解析，注册到BeanFactory中
    - `BeanDefinition`描述：
      - BeanDefinition是一个接口，保存了Bean定义信息（与Bean标签内容一致）
  - bean未初始化，仅仅是解析，记录信息
  - 注册到BeanFactory中，其实就是一个Map（beanName - beanDefinition）
  详情：注册bean
    - 关闭旧的BeanFactory，并创建新的BeanFactory，加载Bean定义，注册Bean
      - refreshBeanFactory()
        - 销毁beanFactory
        - createBeanFactory() 创建一个`DefaultListableBeanFactory`工厂
        - beanFactory.setSerializationId(getId()); 设置工厂序列化

        - customizeBeanFactory(beanFactory); 设置属性，是否允许Bean覆盖是否允许循环引用。
          - 详情：
          - 设置beanFactory是否允许定义覆盖，这里主要指定义bean的时候使用了相同的id或者name，默认情况下，`allowBeanDefinitionOverriding`属性为null，如果同一配置文件中重复了，会抛出错误，但是如果不在同一个配置文件中，会发生覆盖。
          - 设置Bean之间的循环依赖：循坏依赖 A->B  B->A
        - loadBeanDefinitions(beanFactory) 将bean加载到工厂里
          - 详情：
          - 初始化加载各个bean主要用的是一个`XmlBeanDefinitionReader`类
          - initBeanDefinitionReader(beanDefinitionReader) 初始化`Reader`，主要是为了让子类重写，但是没有重新
          - loadBeanDefinitions(beanDefinitionReader)
            - 重要的步骤：
            - 1. 设置ConfigResources()
            - 2. 加载BeanDefinitions()
              - loadBeanDefinitions(Resource... resources) 将所有Resource对象进行解析
              - loadBeanDefinitions(Resource resources) 逐个解析 将resource对象转换成encodeResource对象
              - loadBeanDefinitions(EncodedResource encodedResource),通过resource获取inputsource
              - doLoadBeanDefinitions(inputSource,encodedResource.getResource())
              - Document doc = doLoadDocument(inputSource, resource);转化成doc对象
              - registerBeanDefinitions(doc, resource) 将上步转化的doc对象进行注册bean
              - registerBeanDefinitions(doc, createReaderContext(resource))
              - doRegisterBeanDefinitions()
        - 设置当前bean工厂为引用新的beanFactory（Application.get）
    - 获取刚刚创建的BeanFactory
    - 返回给方法调用者
4. prepareBeanFactory(beanFactory)
  - 这步为 设置 beanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的bean
  -
5. postProcessBeanFactory(beanFactory)
  - 如果实现了BeanFactoryPostProcessor，在容器初始化之后，会调用 postProcessBeanFactory方法。 （方法名可知--在beanFactory工厂之后执行）
  - 在这里是给子类提供的扩展点，bean 值完成了定义与注册，并没有真正的初始化，
  - 具体的子类可以在这步添加一些特殊的 beanFactiryProcessor的实现做点事情。
  - （是实现beanFactiryProcessor 的这个类，在工厂初始化之后，提前做事）
6. invokeBeanFactoryPostProcessors(beanFactory);
  - 执行BeanFactoryPostProcessors方法
7. registerBeanPostProcessors(beanFactory)
  - 注册 实现 BeanPostProcessor的实现类。
8. initMessageSource()
  - i18n 国际化
9. initApplicationEventMulticaster()
  - 初始化事件广播器
10. onRefresh()
  - 模板方法，具体子类可以在这里初始化一些特殊的bean(在singleton beans 之前)
11. registerListeners()
  - 注册事件监听器，监听器需要实现ApplicationListenner接口（web）
12. finishBeanFactoryInitialization(beanFactory)
  - 初始化所有的bean（lazy-init 的除外）
13. finishRefresh()
  - 进行广播，初始化完成
