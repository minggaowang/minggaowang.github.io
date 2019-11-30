---
layout:     post
title:      SpringBoot可执行JAR文件启动器 JarLauncher
subtitle:   可执行JAR文件启动器
date:       2019-11-10
author:     王明高
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - SpringBoot
---

## 1 JarLauncher 包来源

- 在浏览器中输入https://search.maven.org/
- 单击  Classic Search  链接，跳转到新页面
- 单击 Advanced Search 链接，跳转到新页面
- 在 Classname: 输入框输入 org.springframework.boot.loader.JarLauncher
- 单击SEARCH按钮
- 直接定位到Maven GA “ org.springframework.boot:spring-boot-loader:2.0.2.RELEASE”
  -  **GroupId:** org.springframework.boot
  -  **ArtifactId:** spring-boot-loader
  -  **Version:** 2.0.2.RELEASE

## 2 JarLauncher 如何调用执行

官方文档告知了开发人员构建可执行 JAR 的前提，即需要添加 spring-boot-plugin 到 pom.xm 文件中，而该插件默认地被追加到由 https://start.spring.io 构建的 Spring Boot 应用中，因此 first-app-by-gui 项目天然地存在该插件。

当 Spring Boot 应用可执行 JAR 文件被 java -jar 执行时，其命令本身对 JAR 文件是否来自 Spring Boot 插件打包并不感知。换言之，该命令引导的是标准可执行 JAR 文件，而按照 java 官方文档的规定， java -jar 命令引导的具体启动类必须配置 MANIFEST.MF 资源的 Main-Class 属性中。

根据“JAR文件规范”， MANIFEST.MF 资源必须存放在 /META-INF/ 目录下，因此 /META-INF/MANIFEST.MF 如下

>
>
>Manifest-Version: 1.0
>
>Implementation-Title: first-app-by-gui
>
>Implementation-Version: 0.0.1-SNAPSHOT
>
>Built-By: william
>
>Implementation-Vendor-Id:  thinking-in-spring-boot
>
>Spring-Boot-Version:  2.0.2.RELEASE
>
>Main-Class:   org.springframework.boot.loader.JarLauncher
>
>Strart-Class:  thinkinginspringboot.firstappbygui.FirstAppByGuiApplication
>
>Spring-Boot-Classes: BOOT-INF/classes/
>
>Spring-Boot-Lib: BOOT-INF/lib/
>
>Createed-by: Apache Maven 3.5.0
>
>Build-Jdk: 1.8.0_172
>
>Implementation-URL: https://projects.spring.io/spring-boot/#/spring-boot-start-parent/first-app-by-gui

发现 Main-Class 属性指向的 Class 为 org.springframework.boot.loader.JarLauncher，而该类存放在 org/springframework/boot/loader/ 目录下，而且项目的引导类定义在Start-Class属性中，该属性并非 java 平台标准META-INF/MANIFEST.MF属性。 

##  3 JarLauncher 的实现原理

### 3.1 准备工作

- 在正式展开讨论前，需将 first-app-by-gui 工程导入IDEA。
- 为了便于源码分析，需要将 org.springframework.boot:spring-boot-loader 添加到当前 pom.xml 文件中，并且因为 spring-boot-starter-parent 固化依赖的关系，所以不需要指定版本
- 由于运行时 spring-boot-loader 存在于 FAT JAR 中，所以 spring-boot-loader 的依赖 scope 为 provided。当IDEA 下载 spring-boot-loader 完毕，单击右侧 maven Projects 按钮，查看是否完成。

### 3.2 源码分析

- 在IDEA按下 ctr + shift + n 键，查找 org.springframework.boot.loader.JarLauncher 类。

  ```java
  public class JarLauncher extends ExecutableArchiveLauncher {
  
  	static final String BOOT_INF_CLASSES = "BOOT-INF/classes/";
  
  	static final String BOOT_INF_LIB = "BOOT-INF/lib/";
  
  	public JarLauncher() {
  	}
  
  	protected JarLauncher(Archive archive) {
  		super(archive);
  	}
  
  	@Override
  	protected boolean isNestedArchive(Archive.Entry entry) {
  		if (entry.isDirectory()) {
  			return entry.getName().equals(BOOT_INF_CLASSES);
  		}
  		return entry.getName().startsWith(BOOT_INF_LIB);
  	}
  
  	public static void main(String[] args) throws Exception {
  		new JarLauncher().launch(args);
  	}
  
  }
  ```

  

- 明显地发现，BOOT-INF/classes/ 和 BOOT-INF/lib/ 路径分别用 BOOT_INF_CLASSES 和 BOOT_INF_LIB 表示，并且用于 isNestedArchive(Archive.Entry entry) 方法判断，从该方法的实现分析，方法参数 Archive.Entry 对象看似为JAR文件中的资源，比如 application.properties ，不过该对象与 java 标准的  java.util.jar.JarEntry 对象类似，其 name 属性 getName() 方法为 JAR 资源的相对路径。当 application.properties 资源位于 FAT JAR 时，实际的  Archive.Entry#getName() 为 /BOOT-INF/classes/application.properties ，故符合 entry.getName().startWith(BOOT_INF_LIB) 的判断，即 isNestedArchive(Archive.Entry entry) 方法返回 true 。说明 FAT JAR 被解压至文件目录，因此从侧面说明了 Spring Boot 应用能直接通过 java org.springframework.boot.loader.JarLauncher 命令启动的原因。换言之，Archive.Entry 与 java.util.jar.JarEntry 也存在差异，它也可表示文件或目录。实际上，Archive.Entry 存在两种实现，其中一种为 JarFileArchive.JarFileEntry，基于 java.util.jar.JarEntry 实现，表示 FAT JAR 嵌入资源：

  ```java
  public class JarFileArchive implements Archive {
      ...
  	/**
  	 * {@link Archive.Entry} implementation backed by a {@link JarEntry}.
  	 */
  	private static class JarFileEntry implements Entry {
  
  		private final JarEntry jarEntry;
  
  		JarFileEntry(JarEntry jarEntry) {
  			this.jarEntry = jarEntry;
  		}
  
  		public JarEntry getJarEntry() {
  			return this.jarEntry;
  		}
  
  		@Override
  		public boolean isDirectory() {
  			return this.jarEntry.isDirectory();
  		}
  
  		@Override
  		public String getName() {
  			return this.jarEntry.getName();
  		}
  
  	}
  }        
  ```

  另一种实现则是 ExplodedArchive.FileEntry，基于文件系统：

  ```java
  public class ExplodedArchive implements Archive {
      ...
  	/**
  	 * {@link Entry} backed by a File.
  	 */
  	private static class FileEntry implements Entry {
  
  		private final String name;
  
  		private final File file;
  
  		FileEntry(String name, File file) {
  			this.name = name;
  			this.file = file;
  		}
  
  		public File getFile() {
  			return this.file;
  		}
  
  		@Override
  		public boolean isDirectory() {
  			return this.file.isDirectory();
  		}
  
  		@Override
  		public String getName() {
  			return this.name;
  		}
  
  	}
  }
  ```

  所以，这也从实现层面证明了 JarLauncher 支持 JAR 和 文件系统两种启动方式。

  同时，JarLauncher 同样作为引导类，当执行 java -jar 命令时，/META-INF/ 资源的 Main-Class 属性将调用其 main(String[]) 方法，实际上调用的是 JarLauncher#launch(args) 方法，而该方法继承于基类 org.springframework.boot.loader.Launcher ，他们之间的继承关系如下图所示：

  ![](https://raw.githubusercontent.com/minggaowang/minggaowang.github.io/master/img/JarLauncher_20191110181732.png)

前面也提到，WarLauncher 是可执行 WAR 文件的启动器。

- 分析 org.springframework.boot.loader.Launcher#launch(java.lang.String[]) 方法实现：

  ```java
  public abstract class Launcher {
  	/**
  	 * Launch the application. This method is the initial entry point that should be
  	 * called by a subclass {@code public static void main(String[] args)} method.
  	 * @param args the incoming arguments
  	 * @throws Exception if the application fails to launch
  	 */
  	protected void launch(String[] args) throws Exception {
  		JarFile.registerUrlProtocolHandler();
  		ClassLoader classLoader = createClassLoader(getClassPathArchives());
  		launch(args, getMainClass(), classLoader);
  	}
      ...
  }
  ```

  **首先执行 JarFile.registerUrlProtocolHandler(String) 方法：**

  ```java
  public class JarFile extends java.util.jar.JarFile {
      ...
  	private static final String PROTOCOL_HANDLER = "java.protocol.handler.pkgs";
  
  	private static final String HANDLERS_PACKAGE = "org.springframework.boot.loader";
      ...
  	/**
  	 * Register a {@literal 'java.protocol.handler.pkgs'} property so that a
  	 * {@link URLStreamHandler} will be located to deal with jar URLs.
  	 */
  	public static void registerUrlProtocolHandler() {
  		String handlers = System.getProperty(PROTOCOL_HANDLER, "");
  		System.setProperty(PROTOCOL_HANDLER, ("".equals(handlers) ? HANDLERS_PACKAGE
  				: handlers + "|" + HANDLERS_PACKAGE));
  		resetCachedUrlHandlers();
  	}
  
  	/**
  	 * Reset any cached handlers just in case a jar protocol has already been used. We
  	 * reset the handler by trying to set a null {@link URLStreamHandlerFactory} which
  	 * should have no effect other than clearing the handlers cache.
  	 */
  	private static void resetCachedUrlHandlers() {
  		try {
  			URL.setURLStreamHandlerFactory(null);
  		}
  		catch (Error ex) {
  			// Ignore
  		}
  	}        
  }
  ```

  JarFile.registerUrlProtocolHandler()  方法利用了 java.net.URLStreamHandler 扩展机制，其实现由 java.net.URL#getURLStreamHandler 提供：

  ```java
  public final class URL implements java.io.Serializable {
      ...
      public static void setURLStreamHandlerFactory(URLStreamHandlerFactory fac) {
          synchronized (streamHandlerLock) {
              if (factory != null) {
                  throw new Error("factory already defined");
              }
              SecurityManager security = System.getSecurityManager();
              if (security != null) {
                  security.checkSetFactory();
              }
              handlers.clear();
              factory = fac;
          }
      }
      ...
  	/**
       * Returns the Stream Handler.
       * @param protocol the protocol to use
       */
      static URLStreamHandler getURLStreamHandler(String protocol) {
  
          URLStreamHandler handler = handlers.get(protocol);
          if (handler == null) {
  
              boolean checkedWithFactory = false;
  
              // Use the factory (if any)
              if (factory != null) {
                  handler = factory.createURLStreamHandler(protocol);
                  checkedWithFactory = true;
              }
  
              // Try java protocol handler
              if (handler == null) {
                  String packagePrefixList = null;
  
                  packagePrefixList
                      = java.security.AccessController.doPrivileged(
                      new sun.security.action.GetPropertyAction(
                          protocolPathProp,""));
                  if (packagePrefixList != "") {
                      packagePrefixList += "|";
                  }
  
                  // REMIND: decide whether to allow the "null" class prefix
                  // or not.
                  packagePrefixList += "sun.net.www.protocol";
  
                  StringTokenizer packagePrefixIter =
                      new StringTokenizer(packagePrefixList, "|");
  
                  while (handler == null &&
                         packagePrefixIter.hasMoreTokens()) {
  
                      String packagePrefix =
                        packagePrefixIter.nextToken().trim();
                      try {
                          String clsName = packagePrefix + "." + protocol +
                            ".Handler";
                          Class<?> cls = null;
                          try {
                              cls = Class.forName(clsName);
                          } catch (ClassNotFoundException e) {
                              ClassLoader cl = ClassLoader.getSystemClassLoader();
                              if (cl != null) {
                                  cls = cl.loadClass(clsName);
                              }
                          }
                          if (cls != null) {
                              handler  =
                                (URLStreamHandler)cls.newInstance();
                          }
                      } catch (Exception e) {
                          // any number of exceptions can get thrown here
                      }
                  }
              }
  
              synchronized (streamHandlerLock) {
  
                  URLStreamHandler handler2 = null;
  
                  // Check again with hashtable just in case another
                  // thread created a handler since we last checked
                  handler2 = handlers.get(protocol);
  
                  if (handler2 != null) {
                      return handler2;
                  }
  
                  // Check with factory if another thread set a
                  // factory since our last check
                  if (!checkedWithFactory && factory != null) {
                      handler2 = factory.createURLStreamHandler(protocol);
                  }
  
                  if (handler2 != null) {
                      // The handler from the factory must be given more
                      // importance. Discard the default handler that
                      // this thread created.
                      handler = handler2;
                  }
  
                  // Insert this handler into the hashtable
                  if (handler != null) {
                      handlers.put(protocol, handler);
                  }
  
              }
          }
  
          return handler;
  
      }
      ...
  }
  ```

  以上方法实现较为复杂，简言之，URL 的关联协议（Protocol）对应一种 URLStreamHandler 实现类，JDK 默认支持文件（file）、HTTP、JAR 的协议，故JDK内建了对应协议的实现。这些实现类均存放 sun.net.www.protocol 包下，并且类名必须为 Handler，其类全名模式为 sun.net.www.protocol.${protocol}.Handler,其中${protocol}表示协议名，常见如下：

  - File:  sun.net.www.protocol.file.Handler
  - JAR:   sun.net.www.protocol.jar.Handler
  - HTTP:  sun.net.www.protocol.http.Handler
  - HTTPS: sun.net.www.protocol.https.Handler
  - FTP:  sun.net.www.protocol.ftp.Handler

  以上均为 java.net.URLStreamHandler 实现类，换言之，如果需要扩展，则继承 URLStreamHandler 类为必选项，通常配置（java.lang.System#getProperties）java.protocol.handler.pkgs，追加 URLStreamHandler 实现类 package，多个 package 以竖线分割。因此 JarFile.registerUrlProtocolHandler()方法将 package org.springframework.boot.loader 追加到 java 系统属性 java.protocol.handler.pkgs 中。也就是说 org.springframework.boot.loader 包下存在协议对应的 Handler 类，即 org.springframework.boot.loader.jar.Handler，按照类名模式，其实现协议为 JAR 。令人疑惑的是，JAR 协议不是内建实现了吗？它是如何覆盖的呢？请注意 java.net.URL#getURLStreamHandler(String)方法的实现，其处理器包名先读取 Java 系统属性 java.protocol.handler.pkgs，无论是否存在，再追加 sun.net.www.protocol 包，所以 JDK 内建实现是默认实现，故org.springframework.boot.loader.jar.Handler 可覆盖 sun.net.www.protocol.jar.Handler。当Spring Boot FAT JAR 被 java -jar 命令引导时，其内部的 JAR 文件无法被内建实现  sun.net.www.protocol.jar.Handler 当作 Class Path，故需要替换实现。

  **下一步执行 createClassLoader(List) 方法**

  创建 ClassLoader，其中 getClassPathArchives() 方法返回值作为参数:

  ```java
  public abstract class Launcher {
      ...
  	protected abstract List<Archive> getClassPathArchives() throws Exception;
      ...
  }
  ```

  该方法为抽象方法，具体实现由子类 ExecutableArchiveLauncher 提供：

  ```java
  public abstract class ExecutableArchiveLauncher extends Launcher {
      ...
      @Override
  	protected List<Archive> getClassPathArchives() throws Exception {
  		List<Archive> archives = new ArrayList<>(
  				this.archive.getNestedArchives(this::isNestedArchive));
  		postProcessClassPathArchives(archives);
  		return archives;
  	}
     ...
     protected abstract boolean isNestedArchive(Archive.Entry entry);
     ...
  }
  ```

  同样，isNestedArchive(Archive.Entry) 方法需要由子类 JarLauncher 或 WarLauncher 实现，以 JarLauncher  为例：

  ```java
  public class JarLauncher extends ExecutableArchiveLauncher {
      ...
      @Override
  	protected boolean isNestedArchive(Archive.Entry entry) {
  		if (entry.isDirectory()) {
  			return entry.getName().equals(BOOT_INF_CLASSES);
  		}
  		return entry.getName().startsWith(BOOT_INF_LIB);
  	}
      ...
  }
  ```

  该方法又回到了前文的讨论，即过滤 Archive.Entry 实例是否匹配 BOOT/INF/classes/ 的名称或 BOOT-INF/lib/ 的名称前缀。换句话说，无论Archive.Entry 的实现类是 JarFileArchive.JarFileEntry 还是ExplodedArchive.FileEntry，只要它们的名称符合以上路径即可，故 getClassPathArchives() 返回值取决于 archive 属性的内容： 

  ```java
  public abstract class ExecutableArchiveLauncher extends Launcher {
      
      private final Archive archive;
      
      public ExecutableArchiveLauncher() {
  		try {
  			this.archive = createArchive();
  		}
  		catch (Exception ex) {
  			throw new IllegalStateException(ex);
  		}
  	}
      ...
  }
  ```

  该 archive 属性来源于父类的 Launcher 的 createArchive() 方法：

  ```java
  public abstract class Launcher {
     ...
     protected final Archive createArchive() throws Exception {
  		ProtectionDomain protectionDomain = getClass().getProtectionDomain();
  		CodeSource codeSource = protectionDomain.getCodeSource();
  		URI location = (codeSource != null ? codeSource.getLocation().toURI() : null);
  		String path = (location != null ? location.getSchemeSpecificPart() : null);
  		if (path == null) {
  			throw new IllegalStateException("Unable to determine code source archive");
  		}
  		File root = new File(path);
  		if (!root.exists()) {
  			throw new IllegalStateException(
  					"Unable to determine code source archive from " + root);
  		}
  		return (root.isDirectory() ? new ExplodedArchive(root)
  				: new JarFileArchive(root));
  	}
  }
  ```

  此处主要通过当前 Launcher 所在的介质，判断是否为 JAR 归档文件实现（JarFileArchive）或解压目录实现（ExplodeArchive）,不过该方法为final实现，因此其子类 JarLauncher 或 WarLauncher 均继承该实现。获取当前 Spring Boot FAT JAR 或解压其内容后。

  **下一步调用实际的引导类 launch(String[], String, ClassLoader) 方法**

  ```java
  public abstract class Launcher {
      ...
      protected void launch(String[] args, String mainClass, ClassLoader classLoader)
  			throws Exception {
  		Thread.currentThread().setContextClassLoader(classLoader);
  		createMainMethodRunner(mainClass, args, classLoader).run();
  	}
      ...
      protected MainMethodRunner createMainMethodRunner(String mainClass, String[] args,
  			ClassLoader classLoader) {
  		return new MainMethodRunner(mainClass, args);
  	}
      ...
  }
  ```

  该方法的实际执行者为 MainMethodRunner#run() 方法：

  ```java
  public class MainMethodRunner {
      
      private final String mainClassName;
  
  	private final String[] args;
  
  	/**
  	 * Create a new {@link MainMethodRunner} instance.
  	 * @param mainClass the main class
  	 * @param args incoming arguments
  	 */
  	public MainMethodRunner(String mainClass, String[] args) {
  		this.mainClassName = mainClass;
  		this.args = (args != null ? args.clone() : null);
  	}
  
  	public void run() throws Exception {
  		Class<?> mainClass = Thread.currentThread().getContextClassLoader()
  				.loadClass(this.mainClassName);
  		Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
  		mainMethod.invoke(null, new Object[] { this.args });
  	}
  }
  ```

  MainMethodRunner 对象需要关联 mainClass 及 main 方法参数 args，而 mainClass 来自 ExecutableArchiveLauncher#getMainClass() 方法：

  ```java
  public abstract class ExecutableArchiveLauncher extends Launcher {
      ...
      @Override
  	protected String getMainClass() throws Exception {
  		Manifest manifest = this.archive.getManifest();
  		String mainClass = null;
  		if (manifest != null) {
  			mainClass = manifest.getMainAttributes().getValue("Start-Class");
  		}
  		if (mainClass == null) {
  			throw new IllegalStateException(
  					"No 'Start-Class' manifest entry specified in " + this);
  		}
  		return mainClass;
  	}
  }
  ```

  类名称来自 /META-INF/MANIFEST.MF 资源中的 Start-Class 属性，其子类并未覆盖该方法的实现，故无论 JAR 是 WAR，读取  Spring Boot 启动类均来自此属性。获取 mainClass 之后，MainMethodRunner#run() 方法将读取 mainClass 类中的 main(String[]) 方法，并且作为静态方法调用。因此，JarLauncher 实际上是同进程内调用 Start-Class类的 main(String[]) 方法，并且在启动前准备好 Class Path。

  同理，WarLauncher 大部分实现逻辑与 JarLauncher 类似。

  **WarLauncher  和 JarLauncher  差异性讨论**

  二者均继承于 ExecutableArchiveLauncher，并使用 JarFileArchive 和 ExplodedArchive 分别管理归档和解压目录两种资源，其主要区别在于项目类文件和 JAR Class Path 路径的不同

  ```java
  public class WarLauncher extends ExecutableArchiveLauncher {
  
  	private static final String WEB_INF = "WEB-INF/";
  
  	private static final String WEB_INF_CLASSES = WEB_INF + "classes/";
  
  	private static final String WEB_INF_LIB = WEB_INF + "lib/";
  
  	private static final String WEB_INF_LIB_PROVIDED = WEB_INF + "lib-provided/";
  
  	public WarLauncher() {
  	}
  
  	protected WarLauncher(Archive archive) {
  		super(archive);
  	}
  
  	@Override
  	public boolean isNestedArchive(Archive.Entry entry) {
  		if (entry.isDirectory()) {
  			return entry.getName().equals(WEB_INF_CLASSES);
  		}
  		else {
  			return entry.getName().startsWith(WEB_INF_LIB)
  					|| entry.getName().startsWith(WEB_INF_LIB_PROVIDED);
  		}
  	}
  
  	public static void main(String[] args) throws Exception {
  		new WarLauncher().launch(args);
  	}
  
  }
  ```

  根据前面讨论，可以确定的是， WEB-INF/classes/ 、WEB-INF/lib 和 WEB-IN/lib-provided/ 均为 WarLauncherClassLoader 的 Class Path。其中 WEB-INF/classes/ 和 WEB-INF/lib 是传统的Servlet应用的Class Path 路径，而 WEB-INF/lib-provided/ 属于 Spring Boot WarLauncher 定制实现，那么是否意味着其路径存放Maven依赖 scope:provided 的 JAR 呢？回顾 first-app-by-gui 项目，其中 scope 是 provided 的直接依赖为 org.springframework.boot:spring-boot-loader，因此，可将当前 pom.xml 文件的  packaging 元素从 jar 调整为 war，改变当前 Spring Boot 项目打包归档类型：

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
      <modelVersion>4.0.0</modelVersion>
      
      <groupId>thinking-in-spring-boot</groupId>
      <artifactId>first-app-by-gui</artifactId>
      <version>0.0.1-SNAPSHOT</version>
      <name>first-app-by-gui</name>
      <description>第一个 Spring Boot App 项目</description>
      <packaging>war</packaging>
      ...
  </project>
  ```

  相比 FAT JAR 的解压目录，WAR 增加了 WEB-INF/lib-provided，并且该目录仅由一个 JAR 文件，即spring-boot-loader-2.0.2.RELEASE.jar，由此证明 WEB-INF/lib-provided 目录存放的是 scope:provided 的 JAR 文件。不过 spring-boot-loader 如此以上目录结构意图是什么呢？前面提到，传统的 Servlet 应用的 Class Path 路径仅关注 WEB-INF/classes/ 和 WEB-INF/lib 目录，因此，WEB-IN/lib-provided/ 中的 JAR 将被 Servlet 容器忽略，如 servlet API，该 API 由 Servlet 容器提供。因此设计的好处在于，打包后的 WAR 文件能在 Servlet 容器中兼容运行。总而言之，打包 WAR 文件是一种兼容措施，即被 WarLauncher 启动，又能兼容 Servlet 容器环境。换言之，WarLauncher 与 JarLauncher 并无本质差别，所以建议 Spring Boot 应用使用非传统 Web 部署时，尽可能地使用 JAR 归档方式。因此，以上 JAR 和 WAR 归档应用均为 Spring Boot 独立运行。

  ## 4 参考文献

  [1]  小马哥. Spring Boot 编程思想（核心篇）