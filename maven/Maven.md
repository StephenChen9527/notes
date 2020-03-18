## Maven


#### Maven 构建中的各个环节
1. 清理 ： mvn clean 将以前编译得到的旧的class字节码文件删除，为下一次编译做准备
2. 编译 ： mvn compile 将Java文件编译成class文件
3. 测试 ： mvn test-compile/test 自动测测试，调用Juint程序
4. 报告 ： 测试程序执行的结果
5. 打包 ： mvn package 动态Web工程为War包，普通工程为jar包
6. 安装 ： mvn install Maven特定的概念---将打包得到的文件复制到“仓库”中的指定位置
7. 部署 ： mvn deploy 将动态Web工程生成的war包复制到Servlet容器指定的目录下去，使其可以运行

#### Maven 核心概念
1. 约定的目录结构
目录结构
		Hello
		|---src
		|---|---main   业务代码
		|---|---|---java   代码文件夹
		|---|---|---resources  资源文件夹
		|---|---test   测试代码
		|---|---|---java
		|---|---|---resources
		|---pom.xml
2. POM ☆☆☆☆☆
3. 坐标
4. 依赖
5. 仓库
6. 生命周期/插件/目标
7. 继承
8. 聚合


#### Maven 常用命令

Maven 构建中各个环节。

#### POM

Project Object Model 项目对象模型

构建过程相关一切设置在这个文件中进行配置。

#### 坐标

1. groupid：公司或者组织域名倒叙+项目名
2. artifactid：模块名称
3. version：版本

#### 仓库

1. 本地仓库：本地文件夹
2. 远程仓库：
		1. 局域网（私服）：Nexus 搭建在局域网，为局域网内用户服务。
		2. 中央仓库：为全世界服务。
		3. 中央仓库镜像：仓库镜像，就近的仓库。

仓库保存的内容：
	1. Maven自身所需的插件
	2. 第三方框架或jar包
	3. 我们自己开发的Maven工程

#### 依赖
1. Maven解析依赖信息时会到本地仓库中查找被依赖的jar包
		对于我们自己开发的Maven工程，使用install命令部署到本地仓库中



1. compile范围依赖
		* 是否对主程序有效：有效
		* 是否对测试程序有效：有效
		* 是否参与打包：参与
2. test范围依赖
		* 是否对主程序有效：无效
		* 是否对测试程序有效：有效
		* 是否参与打包：不参与
3. provided
		* 是否对主程序有效：有效
		* 是否对测试程序有效：有效
		* 是否参与打包：不参与
		* 仅仅在开发时候提供依赖，部署时候被忽略。

![compile](https://tva1.sinaimg.cn/large/005VwC5mly1gcx0d2xc9vj30cb07jjrp.jpg)

![provided](https://tvax1.sinaimg.cn/large/005VwC5mly1gcx16qp0wxj30fa06w3z5.jpg)

	Hello
	|---src
	|---|---main   compile范围的依赖
	|---|---|---java
	|---|---|---resources
	|---|---test   test范围依赖
	|---|---|---java
	|---|---|---resources
	|---pom.xml


#### 生命周期
1. 各个构建环节执行的顺序：不能打乱顺序，必须按照正确的顺序来执行
2. Maven的核心程序中定义了抽象的生命周期，生命周期中各个阶段的具体任务是由插件来完成的。
3. Maven核心程序为了更好的实现自动化构建，按照这一特点执行生命周期中的各个阶段：不论现在要执行生命周期中的哪一个阶段，都是要从这个生命周期的最初始的位置开始执行。


#### Mavne的依赖性
A依赖于B，B依赖于C，那么当A引入B时，也会引入C。

好处：可以传递依赖，不比在每个模块工程中都重复声明，在最原始的工程中依赖一次即可。

注意：非compile范围的依赖是不能传递。所以在各个工程模块中，如果有需要，就得重复声明依赖。

依赖的排除：需要放入`dependency`

```xml
<exclusions>
	  <exclusion>
			  <groupId></groupId>
				<artifactid></artifactid>
		</exclusion>
</exclusions>
```

依赖原则：

作用：解决模块工程质检的jar包冲突问题

![依赖原则](https://tva2.sinaimg.cn/large/005VwC5mly1gcxc2dcbtsj30ee06vq3o.jpg)

同一版本管理：

maven中使用变量，进行管理版本号：

```xml
<properties>
		<spring.version>4.2.2.RELEASE</spring.version>
</properties>

<version>${spring.version}</version>

```
任何都可以设定为变量，视情况而使用。

#### 继承
Maven也有继承体系，如果有公共依赖的jar包可以放到父工程的pom文件中进行统一管理，这样子可以更方便，这里就不再多说。

但是会写出relativePath 指定父工程的位置。
```xml
<relativePath></relativePath>
```

还有一点，在父工程中引入依赖，使用了
```xml
<dependencyManager>
		<dependencies>

		</dependencies>
</dependencyManager>
```
这种标签，不知道有什么区别...


#### 聚合

作用：一件安装各个模块工程
配置：

```xml
<modules>
	 <module></module>
</modules>
```

#### build 配置
```xml
<build>
 <finalName>项目最终名称</finalName>

	<plugins>
			 <plugin>
				 	 <groupId></groupId>
					 <artifactId></artifactId>
					 <version></version>

					 <configuration>


					 </configuration>

					 <executions>

					 </executions>
			 </plugin>
	</plugins>

</build>
```

maven 没有什么难度，主要是使用插件进行profile。。。

详细的pom文件解析如下：

`https://www.cnblogs.com/tanghaoran-blog/p/10429329.html`

还是很多东西的，用的时候自己再查吧。

以下为插件`properties-maven-plugin`，可以先准备几个文件，然后在编译阶段，替换占位符，进而实现多环境切换。

`https://blog.csdn.net/wangliwei4321/article/details/54089826`
