---
layout: post
title: 类加载器ClassLoader(java层源码级)
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

实际应用常用的代码：
    
    ClassLoader targetClassLoader = null;// 外部参数
    ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
    try {
        Thread.currentThread().setContextClassLoader(targetClassLoader);
        // TODO
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        Thread.currentThread().setContextClassLoader(contextClassLoader);
    }
    
问题来了，为什么用有contextClassLoader？解决什么问题的？   
  有些JVM核心代码必须动态加载由应用程序开发人员提供的资源时.   
  如JAX和rt.jar 因为是两个加载器加载的 那么BootStrap需要加载Ext的资源,怎么办? 这不是与委托机制相反了吗? 所以就不能只依赖委派双亲模式,那么怎么做   
  如如下是单向的，右边的ClassLoader所加载的代码需要反过来去找委派链靠左边的ClassLoader去加载东西怎么办？   
      ClassLoader A -> System class loader -> Extension class loader -> Bootstrap class loader    
  这种情况下就可以把某个位于委派链左边的ClassLoader设置为线程的context class loader，这样就给机会让代码不受parent delegation的委派方向的限制而加载到类了   
  
  如JNDI， Java 命名和目录接口（Java Naming and Directory Interface，JNDI）   
  它的核心内容在rt.jar中的引导类中实现了，但是这些JNDI核心类可能加载由独立厂商实现和部署在应用程序的classpath中的JNDI提供者。   
  这个场景要求一个父类加载器（这个例子中的原始类加载器，即加载rt.jar的加载器）去加载一个在它的子类加载器（系统类加载器）中可见的类。此时通常的J2SE委托机制不能工作，   
  解决办法是让JNDI核心类使用线程上下文加载器，从而有效建立一条与类加载器层次结构相反方向的“通道”达到正确的委托   
  
#### 八 Class
java中有两种对象：实例对象和Class对象   
Class对象是jvm生成用来保存对应类的信息的   

方法  生成实例：forName newInstance, 获取类信息：getDeclaredAnnotations,getDeclaredFields,getDeclaredMethods 类型转换：asSubClass,cast
1.forName 
静态方法，常用Class.forName(");实际调用的都是native方法。反射获取class对象

2.newInstance

    @CallerSensitive
        public T newInstance()
            throws InstantiationException, IllegalAccessException
        {
            if (System.getSecurityManager() != null) {
                checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), false);
            }
    
            // NOTE: the following code may not be strictly correct under
            // the current Java memory model.
    
            // Constructor lookup
            if (cachedConstructor == null) {
                if (this == Class.class) {
                    throw new IllegalAccessException(
                        "Can not call newInstance() on the Class for java.lang.Class"
                    );
                }
                try {
                    Class<?>[] empty = {};
                    final Constructor<T> c = getConstructor0(empty, Member.DECLARED);
                    // Disable accessibility checks on the constructor
                    // since we have to do the security check here anyway
                    // (the stack depth is wrong for the Constructor's
                    // security check to work)
                    java.security.AccessController.doPrivileged(
                        new java.security.PrivilegedAction<Void>() {
                            public Void run() {
                                    c.setAccessible(true);
                                    return null;
                                }
                            });
                    cachedConstructor = c;
                } catch (NoSuchMethodException e) {
                    throw (InstantiationException)
                        new InstantiationException(getName()).initCause(e);
                }
            }
            Constructor<T> tmpConstructor = cachedConstructor;
            // Security check (same as in java.lang.reflect.Constructor)
            int modifiers = tmpConstructor.getModifiers();
            if (!Reflection.quickCheckMemberAccess(this, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                if (newInstanceCallerCache != caller) {
                    Reflection.ensureMemberAccess(caller, this, null, modifiers);
                    newInstanceCallerCache = caller;
                }
            }
            // Run constructor
            try {
                return tmpConstructor.newInstance((Object[])null);
            } catch (InvocationTargetException e) {
                Unsafe.getUnsafe().throwException(e.getTargetException());
                // Not reached
                return null;
            }
        }
        private volatile transient Constructor<T> cachedConstructor;
        private volatile transient Class<?>       newInstanceCallerCache;
   
  看源码，本质上是获取的类的无参构造方法，然后执行无参构造方法来生成实例。
  
  1.检查访问权限
  
  2.检查是否有已经缓存过的构造器,没有则通过获取该类已经声明的的无参构造方法获取并保存缓存构造器中
  
  3.使用类的构造方法来生成实例
  
3.getDeclaredMethods
这里看下getDeclaredMethods源码，getDeclaredFields同理

    @CallerSensitive
    public Method[] getDeclaredMethods() throws SecurityException {
        checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
        return copyMethods(privateGetDeclaredMethods(false));
    }
   
   
     private static Method[] copyMethods(Method[] arg) {
           Method[] out = new Method[arg.length];
           ReflectionFactory fact = getReflectionFactory();
           for (int i = 0; i < arg.length; i++) {
               out[i] = fact.copyMethod(arg[i]);
           }
           return out;
       }
       
      //
         //
         // java.lang.reflect.Method handling
         //
         //
     
         // Returns an array of "root" methods. These Method objects must NOT
         // be propagated to the outside world, but must instead be copied
         // via ReflectionFactory.copyMethod.
         private Method[] privateGetDeclaredMethods(boolean publicOnly) {
             checkInitted();
             Method[] res;
             ReflectionData<T> rd = reflectionData();
             if (rd != null) {
                 res = publicOnly ? rd.declaredPublicMethods : rd.declaredMethods;
                 if (res != null) return res;
             }
             // No cached value available; request value from VM
             res = Reflection.filterMethods(this, getDeclaredMethods0(publicOnly));
             if (rd != null) {
                 if (publicOnly) {
                     rd.declaredPublicMethods = res;
                 } else {
                     rd.declaredMethods = res;
                 }
             }
             return res;
         }
   
   1. 还在最先权限检查
   2. 核心是 private native Method[]      getDeclaredMethods0(boolean publicOnly);获取methods
   3. 然后是copyMethod，调的是反射的copyMethod，实质是调的实现Method.copyMethod() ，   
   Method res = new Method(clazz, name, parameterTypes, returnType,exceptionTypes, modifiers, slot, signature,annotations, parameterAnnotations, annotationDefault);    
   构造复制一个相同的method.   
   之所以需要拷贝可能是因为不想让获取的方法被随意修改

4.asSubclass,cast
    asSubclass转成子类
    
        @SuppressWarnings("unchecked")
        public <U> Class<? extends U> asSubclass(Class<U> clazz) {
            if (clazz.isAssignableFrom(this))
                return (Class<? extends U>) this;
            else
                throw new ClassCastException(this.toString());
        }
        
        
         @SuppressWarnings("unchecked")
         public T cast(Object obj) {
             if (obj != null && !isInstance(obj))
                 throw new ClassCastException(cannotCastMsg(obj));
             return (T) obj;
         }            
   
    
#### 九 SecurityManager
源码很多地方用到SecurityManager   

SecurityManager是jvm提供的在应用层进行安全检查的机制，应用程序可以根据策略文件被赋予一定的权限
  
例如是否可以读写文件，是否可以读写网络端口，是否可以读写内存，是否可以获取类加载器 


### 总结
1.BootstrapClassLoader、ExtClassLoader、AppClassLoader实际是查阅相应的环境属性sun.boot.class.path、java.ext.dirs和java.class.path来加载资源文件的。

2.自己编写的class是AppClassLoader加载的，int等是BootstrapClassLoader加载的。

3.JVM初始化sun.misc.Launcher并创建Extension ClassLoader和AppClassLoader实例。并将ExtClassLoader设置为AppClassLoader的父加载器。

4.双亲委派。委托是从下向上，然后具体查找过程却是自上至下，可以通过classLoad的loadClass方法来体现其向上委托的过程

5.自定义classLoader,重写findClass方法并运用defineClass()方法。不通过构造指定ClassLoader，默认parent是AppClassLoader