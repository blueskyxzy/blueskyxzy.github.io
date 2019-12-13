---
layout: post
title: 类加载器ClassLoader
date: 2019-12-20
tags: 技术    
---

### what is ClassLoader
加载类的抽象类

作用就是将class文件加载到jvm虚拟机中去

每个class对象都包含一个定义它的ClassLoader的引用

可以实现ClassLoader的子类，来扩展Java动态加载类的方式

### 类加载器分类
虚拟机不会一次加载全部的class文件，而是根据层级来动态加载

* 启动类加载器(Bootstrap ClassLoader):   
    主要加载核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等到虚拟机内存中，不能被java程序直接调用,代码是使用C++编写的.是虚拟机的代码

* 扩展类加载器(Extendsion ClassLoader):   
    扩展的类加载器，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件，开发者可以直接使用这个类加载器.
    
    也可以指定-D java.ext.dirs参数来添加和改变ExtClassLoader的加载路径

* 应用程序类加载器(Application ClassLoader):   
    加载用户类路径(CLASSPATH)下的类库,一般我们编写的java类都是由这个类加载器加载,这个类加载器是CLassLoader中的getSystemClassLoader()方法的返回值,所以也称为系统类加载器.


### 类加载器的双亲委派机制
    如果一个类加载器收到了类加载的请求,它不会自己去尝试加载这个类,而是把这个请求委派给父类加载器去完成,这样层层递进,
    最终所有的加载请求都被传到最顶层的启动类加载器中,只有当父类加载器无法完成这个加载请求(它的搜索范围内没有找到所需的类)时,才会交给子类加载器去尝试加载.


    作用：   
    类随着它的类加载器拥有了优先级的层次关系，
    如java.langObject,它存放在\jre\lib\rt.jar中,它是所有java类的父类,因此无论哪个类加载都要加载这个类,最终所有的加载请求都汇总到顶层的启动类加载器中,因此Object类会由启动类加载器来加载

### 源码研究
#### 一 Launcher
sun.misc.Launcher,它是一个java虚拟机的入口应用
源码如下图1所示：
 ![图1](/images/posts/classLoader/1.png)  
 源码可以看出：
 
 1.构造函数初始化了ExtClassLoader和AppClassLoader
 
 2.静态属性获取了 System.getProperty("sun.boot.class.path")，这些是/lib下的核心启动jar的路径字符串。再看System源码可以看出是native方法获取的配置属性
 
    private static Properties props;
    private static native Properties initProperties(Properties props);
 
 3.后面的代码获取的是SecurityManager，这东西现在不清楚具体干啥，应该和安全，权限等有关
 
#### 二 SystemClassLoader
 在很多源码中可以看到这句： systemClassLoader = ClassLoader.getSystemClassLoader();
 ![图2](/images/posts/classLoader/2.png)
 通过上面的分析，刚好知道getClassLoader的classLoader就是Launcher初始化中的AppClassLoader。
 
 同理
     static class AppClassLoader extends URLClassLoader {
  
          public static ClassLoader getAppClassLoader(final ClassLoader var0) throws IOException {
              final String var1 = System.getProperty("java.class.path");
              final File[] var2 = var1 == null ? new File[0] : Launcher.getClassPath(var1);
              return (ClassLoader)AccessController.doPrivileged(new PrivilegedAction<Launcher.AppClassLoader>() {
                  public Launcher.AppClassLoader run() {
                      URL[] var1x = var1 == null ? new URL[0] : Launcher.pathToURLs(var2);
                      return new Launcher.AppClassLoader(var1x, var0);
                  }
              });
          }
        ...  
      }
      
   System.getProperty("java.class.path");获取classPath地址并用文件系统加载资源   
   
#### 三 ExtClassLoader   
ExtClassLoader默认是System.getProperty("java.ext.dirs");获取路径地址然后getExtDirs获取file文件资源，并提供getExtURLs等扩展加载资源功能
 
      static class ExtClassLoader extends URLClassLoader {
            public static Launcher.ExtClassLoader getExtClassLoader() throws IOException {
                final File[] var0 = getExtDirs();
    
                try {
                    return (Launcher.ExtClassLoader)AccessController.doPrivileged(new PrivilegedExceptionAction<Launcher.ExtClassLoader>() {
                        public Launcher.ExtClassLoader run() throws IOException {
                            int var1 = var0.length;
    
                            for(int var2 = 0; var2 < var1; ++var2) {
                                MetaIndex.registerDirectory(var0[var2]);
                            }
    
                            return new Launcher.ExtClassLoader(var0);
                        }
                    });
                } catch (PrivilegedActionException var2) {
                    throw (IOException)var2.getException();
                }
            }
    
    
            private static File[] getExtDirs() {
                String var0 = System.getProperty("java.ext.dirs");
                File[] var1;
                if (var0 != null) {
                    StringTokenizer var2 = new StringTokenizer(var0, File.pathSeparator);
                    int var3 = var2.countTokens();
                    var1 = new File[var3];
    
                    for(int var4 = 0; var4 < var3; ++var4) {
                        var1[var4] = new File(var2.nextToken());
                    }
                } else {
                    var1 = new File[0];
                }
    
                return var1;
            }
            
            ...
      }
    
#### 四 ClassLoader的parent   
    自己编写的class是AppClassLoader加载的，int等是BootstrapClassLoader加载的。
   
    
    static class ExtClassLoader extends URLClassLoader {}
    static class AppClassLoader extends URLClassLoader {}
    
通过查看ClassLoader源码发现：

     private final ClassLoader parent;
     
parent是构造函数传入的，也就是可以通过构造指定parent

在Launcher初始化中发现，AppClassLoader创建时传入了ExtClassLoader，一步步发现就是将ExtClassLoader设置为AppClassLoader的父加载器。

    var1 = Launcher.ExtClassLoader.getExtClassLoader();
    this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);

     static class AppClassLoader extends URLClassLoader {
            final URLClassPath ucp = SharedSecrets.getJavaNetAccess().getURLClassPath(this);
    
            public static ClassLoader getAppClassLoader(final ClassLoader var0) throws IOException {
                final String var1 = System.getProperty("java.class.path");
                final File[] var2 = var1 == null ? new File[0] : Launcher.getClassPath(var1);
                return (ClassLoader)AccessController.doPrivileged(new PrivilegedAction<Launcher.AppClassLoader>() {
                    public Launcher.AppClassLoader run() {
                        URL[] var1x = var1 == null ? new URL[0] : Launcher.pathToURLs(var2);
                        return new Launcher.AppClassLoader(var1x, var0);
                    }
                });
            }
    
            AppClassLoader(URL[] var1, ClassLoader var2) {
                super(var1, var2, Launcher.factory);
                this.ucp.initLookupCache(this);
            }
            
          ...
          }
AppClassLoader(URL[] var1, ClassLoader var2) 的var2就是ExtClassLoader并且调用了ClassLoader的构造设置parent为ExtClassLoader


这里会有一个疑问：明明都只有一个parent，为什么叫双亲委派，不叫父亲委派啥的？   
国外叫parents delegate model也有parent delegate model   
1.翻译问题，但毕竟用这么久了，双亲应该有它的道理吧   
2.parent不是通过继承，而是组合来实现的   
3.jvm和classpath两种形式的   

#### 五 ClassLoader

重要方法：
1.Class<?> loadClass(String name)

    protected Class<?> loadClass(String name, boolean resolve)
            throws ClassNotFoundException
        {
            synchronized (getClassLoadingLock(name)) {
                // First, check if the class has already been loaded
                Class<?> c = findLoadedClass(name);
                if (c == null) {
                    long t0 = System.nanoTime();
                    try {
                        if (parent != null) {
                            c = parent.loadClass(name, false);
                        } else {
                            c = findBootstrapClassOrNull(name);
                        }
                    } catch (ClassNotFoundException e) {
                        // ClassNotFoundException thrown if class not found
                        // from the non-null parent class loader
                    }
    
                    if (c == null) {
                        // If still not found, then invoke findClass in order
                        // to find the class.
                        long t1 = System.nanoTime();
                        c = findClass(name);
    
                        // this is the defining class loader; record the stats
                        sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                        sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                        sun.misc.PerfCounter.getFindClasses().increment();
                    }
                }
                if (resolve) {
                    resolveClass(c);
                }
                return c;
            }
        }
        ...
        private final ConcurrentHashMap<String, Object> parallelLockMap;
        
        ...
        
         protected Object getClassLoadingLock(String className) {
                Object lock = this;
                if (parallelLockMap != null) {
                    Object newLock = new Object();
                    lock = parallelLockMap.putIfAbsent(className, newLock);
                    if (lock == null) {
                        lock = newLock;
                    }
                }
                return lock;
            }
         }
         ...
         protected final Class<?> findLoadedClass(String name) {
             if (!checkName(name))
                return null;
             return findLoadedClass0(name);
         }
         
         private native final Class<?> findLoadedClass0(String name);  
         ...
        
         
  步骤： 
   1.加对象锁，锁存在ConcurrentHashMap中，className为key,object为value
   
   2.findLoadedClass(name)来判断是否已经加装过，checkName做字符串参数验证，findLoadedClass0是native的方法处理
   
   3.parent不为null,则调用父加载器加装c = parent.loadClass(name, false);如AppClassLoader的父为ExtClassLoader，以及自定义的classLoader
   
   4.parent为null,如ExtClassLoader，则调用c = findBootstrapClassOrNull(name); 方法也是先验证参数然后调native方法处理
   
   5.还找不到则调findClass方法，这是子类实现，如果加载器没有重写findClass方法则直接抛错throw new ClassNotFoundException(name);
   
   6.之后是 resolveClass(c);，默认是false不调处理方法的
   
   该方法代码也解释了双亲委托！原来就是在这实现的
   
2.defineClass

     protected final Class<?> defineClass(String name, byte[] b, int off, int len,
                                             ProtectionDomain protectionDomain)
            throws ClassFormatError
        {
            protectionDomain = preDefineClass(name, protectionDomain);
            String source = defineClassSourceLocation(protectionDomain);
            Class<?> c = defineClass1(name, b, off, len, protectionDomain, source);
            postDefineClass(c, protectionDomain);
            return c;
        }
     }
     
它能将class二进制内容转换成Class对象,其中核心方法是native,也就是JVM来实现二进制转成class

#### 六 自定义ClassLoader   
源码有很多自己重写的ClassLoader,而且ClassLoader又局限性，只能加载指定目录的class文件或者jar包，规则和格式受限等。可以编写自定义加载器加载指定目录的文件或者文件格式可以不是class，自己写加密解密等

1.继承ClassLoader抽象类

2.一般重写findClass方法，而不是重写loadClass调用defineClass()；并在findClass看loadClass源码就解释了。
defineClass()需要先io读取file文件转成byte[].

也可以用已有的classLoader，如systemClassLoader组合为成员
   
    protected ClassLoader() {
        this(checkCreateClassLoader(), getSystemClassLoader());
    }  
    
一个ClassLoader创建时如果没有指定parent，那么它的parent默认就是AppClassLoader。看源码构造默认给parent为SystemClassLoader。
 
#### 七 ContextClassLoader

如Thread类中就有成员变量    private ClassLoader contextClassLoader;
可以通过setContextClassLoader()设置

#### 八 Class


### 总结
1.BootstrapClassLoader、ExtClassLoader、AppClassLoader实际是查阅相应的环境属性sun.boot.class.path、java.ext.dirs和java.class.path来加载资源文件的。

2.自己编写的class是AppClassLoader加载的，int等是BootstrapClassLoader加载的。

3.JVM初始化sun.misc.Launcher并创建Extension ClassLoader和AppClassLoader实例。并将ExtClassLoader设置为AppClassLoader的父加载器。

4.双亲委派。委托是从下向上，然后具体查找过程却是自上至下，可以通过classLoad的loadClass方法了解其过程

5.自定义classLoader,重写findClass方法并运用defineClass()方法。不通过构造指定ClassLoader，默认parent是AppClassLoader