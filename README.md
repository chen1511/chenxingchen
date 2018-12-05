# Log4j源码阅读分析报告
## Part1——功能分析与建模  
### 一、为什么要用log4j？  
&emsp;&emsp;首先可以确认的是，我们大多数人，至少看这篇文章的人，基本都写过一些代码。那么问个小问题，我们的代码出错的时候，我们是怎么发现错误的呢？你们有些人可能会说，我是通过系统反馈的debug信息来一步步进行调试的。或许还有一部分人会说，我是通过在代码每一行写一个print语句看看printf语句能否输出来判断这个语句是否出错的。的确，在代码量比较少的时候，这样的确是个找到错误句子的一个简单的方法，多耗费一点时间而已。但是在代码量非常大的时候，这种方法就不可取了。这个时候我们一般都是通过第一种人说的那种方法，我们根据debug信息来确定错误的来源。debug信息就是日志记录的一种具体表现，日志记录不止在编译器debug的时候有用。在其他方面，比如我们的一个应用出了问题，我们要定位问题的源头，或者我们的数据出现了一些问题，我们要分析离线数据来找到这些问题。这个时候日志记录就很有用了，其可以把程序运行时的各种数据日志信息打印出来，从而让我们更好的分析，这样也更加的节省时间。  
&emsp;&emsp;介绍完了日志记录的功能，我们该介绍为什么要用Log4j了。我们先看看Log4j是什么东西，Log4j是Apache的一个开放源代码的项目，通过使用log4j，我们可以控制日志信息输送的目的地：控制台、文件、GUI组件、套接口服务器、NT的事件记录器、UNIX Syslog守护进程等；我们也可以控制每一条日志的输出格式；通过定义每一条日志信息的级别，我们能够更加细致地控制日志的生成过程。最重要的是，我们可以直接通过配置文件来对这些部件进行灵活地配置，而不用修改相应的源代码。所以说Log4j就是一个日志记录的框架了？没错，Log4j就是一个日志记录的框架。并且现在正在被很多软件所使用。因此，我们有必要对Log4j的源码进行分析，搞清楚其工作原理。
### 二、在看Log4j时，我们应该重点关注哪些部分？  
&emsp;&emsp;Log4j在我们的平时生活中非常有用，其中日志优先级，日志输出目的地，日志输出格式，这几部分都是我们在使用Log4j时重点关注的部分。而本次课堂分析因为我们需要关注的功能是日志记录。因此，我们更应该关注的是Log4j这个程序是怎么将日志信息给记录下来的。我们更应该关注的功能是日志是什么时候产生的，日志产生的步骤，以及日志是以什么样的形式反映出来的，这些都是我们在看源码的时候需要重点注意的部分。
## Part2——设计流程与分析  
### 一、设计流程介绍  
&emsp;&emsp;log4j的设计流程不是很复杂，但是通过文字进行介绍的话会比较抽象，所以我先用一张图来描述Log4j源码中的层次划分，在这里，我参考了文章[Apache log4j-1.2.17源码学习笔记](https://blog.csdn.net/zilong_zilong/article/details/78715500)并引用了文章中的图片。  
     ![avatar](http://dl2.iteye.com/upload/attachment/0128/1093/51891f13-a4bd-3108-99fd-42a99632fce8.jpg)  
&emsp;&emsp;通过观看这张流程图，我们能够很清晰的理清源码中各部件的关系，比如Repository,Hierarchy,Logger,Rrootlogger等等。  
&emsp;&emsp;首先我们能看到这张图最外面的一层是LogManager，这说明我们在记录日志的时候，总是从LogManager开始的，然后比其第一级的是RepositorySelector，即仓库选择器，为我们选择一个仓库，它的实现类也用小字标明了，DefaultRepositorySelector，然后再往下一级就是仓库了，其中装载着日志的各种信息，Repository的类型为LoggerRepository，实现类为Hierarchy，Repository中每一个部件的作用都已经在其中看出，DefaultFactory是负责logger创建的日志工厂类，其默认为DefaultCatagoryFactory，Listeners是Vector类型的变量，其功能是维持jmx对于appender的增加或者删除时间的监听处理，ht是Hashtable类型的变量，记录着所有logger类型变量名字的Catagory或者Privosion，还有其他一些变量信息就不一一说明了，通过观看图片就能很清楚看出来他们的功能及类别。而Root类型为Logger，实现类型为Rootlogger，然后再对root进行层层剖析，进而找到Rootlogger，Logger，然后通过Catagory，我们能够找到这个日志的各种信息了，比如level，name，parent，aai等等信息。然后我们是通过调用Appender对信息进行输出的，我们先通过layout对输出信息进行格式化，再调用不同的Appender，比如Consoleappender，Fileappender等等。到这个时候我们差不多对整个代码的设计流程有个大概的了解了。  
### 二、设计流程分
&emsp;&emsp;前面对Log4j介绍了这么多，可是我们还没有看过Log4j代码，现在我们来看看代码，通过对代码进行分析来进一步对Log4j源码进行解析。首先我们找到一个程序例子，从程序入手来对日志记录过程进行分析。这里我选择的是源码中附带的一个sort.java程序，代码如下：
```
public class Sort {

  static Logger logger = Logger.getLogger(Sort.class.getName());
  
  public static void main(String[] args) {
    if(args.length != 2) {
      usage("Incorrect number of parameters.");
    }
    int arraySize = -1;
    try {
      arraySize = Integer.valueOf(args[1]).intValue();
      if(arraySize <= 0) 
	usage("Negative array size.");
    }
    catch(java.lang.NumberFormatException e) {
      usage("Could not number format ["+args[1]+"].");
    }

    PropertyConfigurator.configure(args[0]);

    int[] intArray = new int[arraySize];

    logger.info("Populating an array of " + arraySize + " elements in" +
	     " reverse order.");
    for(int i = arraySize -1 ; i >= 0; i--) {
      intArray[i] = arraySize - i - 1;
    }

    SortAlgo sa1 = new SortAlgo(intArray);
    sa1.bubbleSort();
    sa1.dump();

    // We intentionally initilize sa2 with null.
    SortAlgo sa2 = new SortAlgo(null);
    logger.info("The next log statement should be an error message.");
    sa2.dump();  
    logger.info("Exiting main method.");    
  }
  
  static
  void usage(String errMsg) {
    System.err.println(errMsg);
    System.err.println("\nUsage: java org.apache.examples.Sort " +
		       "configFile ARRAY_SIZE\n"+
      "where  configFile is a configuration file\n"+
      "      ARRAY_SIZE is a positive integer.\n");
    System.exit(1);
  }
}
```  
&emsp;&emsp;我们可以看到程序先声明了一个
`static Logger logger = Logger.getLogger(Sort.class.getName());`
Logger类型的变量，然后发现其调用了子函数getlogger，于是我们去找getlogger函数，查看其是怎么运行的。
```
static
  public
  Logger getLogger(String name) {
    return LogManager.getLogger(name);
  }
```  
&emsp;&emsp;然后我们发现其调用了Logmanager的代码，因此我们很自然的继续去分析Logmanager的代码，看看其是怎么返回的。我们先看看Logmanager前面的static段，代码如下：  
```
static {
    // By default we use a DefaultRepositorySelector which always returns 'h'.
    Hierarchy h = new Hierarchy(new RootLogger((Level) Level.DEBUG));
    repositorySelector = new DefaultRepositorySelector(h);

    /** Search for the properties file log4j.properties in the CLASSPATH.  */
    String override =OptionConverter.getSystemProperty(DEFAULT_INIT_OVERRIDE_KEY,
						       null);

    // if there is no default init override, then get the resource
    // specified by the user or the default config file.
    if(override == null || "false".equalsIgnoreCase(override)) {

      String configurationOptionStr = OptionConverter.getSystemProperty(
							  DEFAULT_CONFIGURATION_KEY, 
							  null);

      String configuratorClassName = OptionConverter.getSystemProperty(
                                                   CONFIGURATOR_CLASS_KEY, 
						   null);
      URL url = null;

      // if the user has not specified the log4j.configuration
      // property, we search first for the file "log4j.xml" and then
      // "log4j.properties"
      if(configurationOptionStr == null) {	
	url = Loader.getResource(DEFAULT_XML_CONFIGURATION_FILE);
	if(url == null) {
	  url = Loader.getResource(DEFAULT_CONFIGURATION_FILE);
	}
      } else {
	try {
	  url = new URL(configurationOptionStr);
	} catch (MalformedURLException ex) {
	  // so, resource is not a URL:
	  // attempt to get the resource from the class path
	  url = Loader.getResource(configurationOptionStr); 
	}	
      }
      
      // If we have a non-null url, then delegate the rest of the
      // configuration to the OptionConverter.selectAndConfigure
      // method.
      if(url != null) {
	    LogLog.debug("Using URL ["+url+"] for automatic log4j configuration.");
        try {
            OptionConverter.selectAndConfigure(url, configuratorClassName,
					   LogManager.getLoggerRepository());
        } catch (NoClassDefFoundError e) {
            LogLog.warn("Error during default initialization", e);
        }
      } else {
	    LogLog.debug("Could not find resource: ["+configurationOptionStr+"].");
      }
    } else {
        LogLog.debug("Default initialization of overridden by " + 
            DEFAULT_INIT_OVERRIDE_KEY + "property."); 
    }  
  } 
  ```  
  看完这个还是不太懂，我们先具体看看Hierarchy类的代码：
  ```
  public interface LoggerRepository {  
public void addHierarchyEventListener(HierarchyEventListener listener);  
 boolean isDisabled(int level);  
 public void setThreshold(Level level);  
 public void setThreshold(String val);  
public void emitNoAppenderWarning(Category cat);  
 public Level getThreshold();  
 public Logger getLogger(String name);  
public Logger getLogger(String name, LoggerFactory factory);  
 public Logger getRootLogger();  
public abstract Logger exists(String name);  
public abstract void shutdown();  
public Enumeration getCurrentLoggers();  
 public abstract void fireAddAppenderEvent(Category logger, Appender appender);  
 public abstract void resetConfiguration();  
}  
```  
&emsp;&emsp;Hierarchy是Log4j中默认对LoggerRepository的实现类，它用于表达其内部的Logger是以层次结构存储的。在对LoggerRepository接口的实现中，getLogger()方法是其最核心的实现，因而首先从这个方法开始。Hierarchy中用一个Hashtable来存储所有Logger实例，它以CategoryKey作为key，Logger作为value，其中CategoryKey是对Logger中Name字符串的封装，之所以要引入这个类是出于性能考虑，因为它会缓存Name字符串的hash code，这样在查找过程中计算hash code时就可以直接取得而不用每次都计算。  
  &emsp;&emsp;我们再来看Logmanager这段代码，代码是static类型的，即一段静态变量。意思是在Logmanager每次被运行的时候，这段代码每次都会被运行且只运行一次，不需要任何条件，目的是初始化整个过程。接下来我们来分析static中的内容。首先创建了一个默认的RepositorySelector对象，其主要在Logmanager中使用，它的实现类对特定的应用上下文提供了一个LoggerPrspository。其实现类负责追踪应用上下文。ResponsitorySelector提供了LoggerRepository getLoggerRepository();方法。LoggerRepository从字面上理解，它是一个Logger的容器，它会创建并缓存Logger实例，从而具有相同名字的Logger实例不会多次创建，以提高性能。这两个东西搞好了之后，后面的部分就是设置配置文件了。Log4j支持两种配置文件：properties文件和xml文件。先加载override配置文件，之后再根据前面的情况加载Configurator配置文件，然后Configurator解析配置文件，并将解析后的信息添加到LoggerRepository中。LogManager最终将LoggerRepository和Configurator整合在一起,构建一个基本的RootLogger。这样我们的static部分就讲的差不多了。用一张流程图介绍如下：
  ![avatar](http://dl2.iteye.com/upload/attachment/0128/0358/45329127-53e1-38d0-8141-f2d1ef0d5f02.jpg)  
  这张图同样是借用的上面这篇文章中的图片。其很清晰的解释了配置文件加载的层次关系。  
    &emsp;&emsp;接下来就开始讲Logger.getLogger（class classname）这个函数干了什么事。继续通过一张图片来说明实验过程
    ![avatar](http://dl2.iteye.com/upload/attachment/0128/0466/81af7de2-84ee-3400-a734-8274ade18493.jpg)  
    图片中是原文章调用的另外一个函数，这个不太重要，我们需要明白的是这个过程干了什么事。从图片中可以看出，我们调用开始这个rootlogger函数时，我们就创立一个新的日志，但是其没有任何appender，并且其父母指向了rootlogger，通过查看logmanager中的代码：
    ```
    
public static Logger getLogger(final String name) {
	return getLoggerRepository().getLogger(name);
}
/**
 * repositorySelector在LogManager的静态代码块已经创建，默认实现为DefaultRepositorySelector
 */
static public LoggerRepository getLoggerRepository() {
	if (repositorySelector == null) {
		repositorySelector = new DefaultRepositorySelector(new NOPLoggerRepository());
		guard = null;
		Exception ex = new IllegalStateException("Class invariant violation");
		String msg = "log4j called after unloading, see http://logging.apache.org/log4j/1.2/faq.html#unload.";
		if (isLikelySafeScenario(ex)) {
			LogLog.debug(msg, ex);
		} else {
			LogLog.error(msg, ex);
		}
	}
	return repositorySelector.getLoggerRepository();
}
```  
由于Hierarchy是其实现类，我们继续跟进代码，看看Hierarchy中getlogger是怎么实行的。
```

public Logger getLogger(String name) {
	return getLogger(name, defaultFactory);
}
 
/**
 * 传入Logger名字，获取真对该名字唯一的一个Logger，只会在第一次创建，第二次是直接返回第一次创建的Logger对象
 */
public Logger getLogger(String name, LoggerFactory factory) {
	//CategoryKey是一个对String的包装类，这里传入了name=com.log4jtest.LoggerTest
	CategoryKey key = new CategoryKey(name);
 
      //Logger只会创建一次，创建完毕立即放入类型为Hashtable的变量ht中存储，后续直接取出返回
	Logger logger;
 
	//这里防止多线程重复创建，先对类型为Hashtable的变量ht上锁，避免多线程相互竞争
	synchronized (ht) {
		//从类型为Hashtable的变量ht中查找之前是否已经创建过该Logger
		Object o = ht.get(key);
		if (o == null) {//如果从类型为Hashtable的变量ht中找不到名字为com.log4jtest.LoggerTest的Logger，这里会执行
			//调用工厂类DefaultCategoryFactory创建一个全新的Logger对象
			logger = factory.makeNewLoggerInstance(name);
			//设置新建的Logger对象的parent属性指向log4j-1.2.17名字为root的全局根节点RootLogger
			logger.setHierarchy(this);
			//创建完毕立即放入类型为Hashtable的变量ht中存储，后续直接取出返回
			ht.put(key, logger);
			//更新刚才的名字com.log4jtest.LoggerTest的Logger对应的报名，往类型为Hashtable的变量ht中丢入如下信息
			//com.log4jtest.LoggerTest  -->  Logger[name=com.log4jtest.LoggerTest]
			//com.log4jtest                  -->  ProvisionNode{Logger[name=com.log4jtest.LoggerTest]}
			//com                               -->  ProvisionNode{Logger[name=com.log4jtest.LoggerTest]}
			updateParents(logger);
			return logger;
		} else if (o instanceof Logger) {//如果从类型为Hashtable的变量ht中能找到对应名字的Logger，这里会执行，直接返回该Logger
			return (Logger) o;
		} else if (o instanceof ProvisionNode) {
			//如果从类型为Hashtable的变量ht中能找到对应名字ProvisionNode
              //说明之前是子类里创建了Logger，包名的父包名被指向了ProvisionNode，那么这里就创建一个新的Logger，然后把类型为Hashtable的变量ht的记录更新掉
			logger = factory.makeNewLoggerInstance(name);
			//设置新建的Logger对象的parent属性指向log4j-1.2.17名字为root的全局根节点RootLogger
			logger.setHierarchy(this);
			ht.put(key, logger);
              //说明之前是子类里创建了Logger，包名的父包名被指向了ProvisionNode，那么这里就创建一个新的Logger，然后把类型为Hashtable的变量ht的记录更新掉
			updateChildren((ProvisionNode) o, logger);
			updateParents(logger);
			return logger;
		} else {
			// It should be impossible to arrive here
			return null; // but let's keep the compiler happy.
		}
	}
}
```
这样，我们会发现这里是调用了factory，来构造一个新的日志，这样我们就分析清楚了这条日志产生的流程了。而日志内容，则是通过Logger.info、Logger.warn和Logger.error调用appender来打印出来的，因次我们还要分析这几个内容。这几个就在最后一节——高级设计意图中去分析了。
