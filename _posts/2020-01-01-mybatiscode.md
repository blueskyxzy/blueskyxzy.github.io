---
layout: post
title: mybatis源码研究
date: 2020-01-01
tags: 技术    
---

### 简介
持久层框架   
定制化的SQL，针对接口的映射，封装了JDBC代码并设置参数和获取结果集，存储过程支持等

### 简单原理
网上有很多这种手写mybatis的源码来模拟其大致流程

要点：   
1.读取xml配置如数据库配置   
2.解析dao层xml并封装bean   
3.动态代理   
4.jdbc连接执行sql并解析结果


Configuration定义了读取mapper.xml和database.xml的方法   
SqlSession定义了获取动态代理对象和执行Execute的方法。保存Execute和Configuration的引用   
Execute调用jdbc操作数据库，利用Configuration引用读取database.xml，遍历结果集   

SqlSession获取MapperProxy代理对象接口，动态代理对象MapperProxy，proxy读取Mapper.xml配置并调用execute对象执行查询数据库

用的是java自带的代理，必须是对接口代理，实现自己写在invoke方法。   
方法代理实现：解析读取xml并封装到方法bean容器,找到方法名字对应的方法bean，然后再jdbc连接，执行bean中的sql,返回并封装

模拟的只是大致流程，扩展性不强，很多功能不支持。

### 框架结构
   ![mybatis框架图](/images/posts/mybatis/mybatis001.png)  
   
    org.apache.ibatis.annotations  自定义的注解包。如@Param
    org.apache.ibatis.binding   动态代理包。如MapperProxy
    org.apache.ibatis.building  构造器包。如XMLConfigBuilder解析configuration.xml,XMLMapperBuilder解析Mapper.xml,XMLStatementBuider解析select,update等标签
    org.apache.ibatis.cache     缓存包。如缓存装饰器 二级缓存TransactionalCache
    org.apache.ibatis.cursor    游标包。DefaultCursor，mybatis3.4新特性，当查询大量（上百万）数据的时候，使用游标可以有效的减少内存使用，不需要一次性将所有数据得到，可以通过游标逐个或者分批（逐个获取一批后）处理，通常不适合一次性加载到内存中，类似使用SAX解析XML
    org.apache.ibatis.datasource  数据源。如jndi数据源和连接池 
    org.apache.ibatis.executor  执行器。sql执行器，缓存执行器，错误上下文跟踪所有执行流程
    org.apache.ibatis.io   io包。读取资源文件
    org.apache.ibatis.javassist   javassist包
    org.apache.ibatis.jdbc    jdbc包。包括JDBC,ScriptRunner脚本执行等
    org.apache.ibatis.lang    @UsesJava7 @UsesJava8注解，用于识别哪些可以使用JDK7或8的api
    org.apache.ibatis.logging  日志包。多种日志框架的对接，如实现debug模式下输出SQL
    org.apache.ibatis.mapping  映射包。配置文件和对象映射
    org.apache.ibatis.ognl   ognl包
    org.apache.ibatis.parsing   解析工具包。XPathParser通过XPath解析XML，PropertyParser解析properties,GenericTokenParser解析# &等占位符
    org.apache.ibatis.plugin    拦截器插件
    org.apache.ibatis.reflection  反射器。把Java对象转换成元数据MetaObject,数据库查询结果到元数据的映射就是通过元对象实现的
    org.apache.ibatis.scripting  动态SQL支持实现。如xml中的<if> <where>等
    org.apache.ibatis.session    核心包，sqlSession功能。获取Mapper,管理事务，执行sql等
    org.apache.ibatis.transaction  事务。包装数据库连接，处理连接生命周期
    org.apache.ibatis.type   类型处理器。包括数据库类型对应java类型，可自定义类型处理器
 
### 源码分析
mybatis源码   
版本：3.4.2   
作者：Clinton Begin  网上意外发现这名开发者现在已经加入了拳头游戏了
   
#### 1.测试demo

        // 获取Mybatis的配置文件，内部通过ClassLoader加载文件流
        InputStream resourceAsStream = Resources.getResourceAsStream("mybatis-config.xml");
        // 创建sessionFactory对象,通过JDK内部的w3c解析配置文件的内容，封装到Configration对象中，最后通过Configuration来创建DefaultSqlSessionFactory.
        SqlSessionFactory sf = new SqlSessionFactoryBuilder().
                build(resourceAsStream);
        // 通过SqlSessionFactory创建SqlSession对象
        SqlSession session = sf.openSession();
        //创建实体对象
        User user = new User();
        user.setName("boy");
        // 这里测试暂用明文
        user.setPassword("123456");
        user.setMobile("18656400001");
        //保存数据到数据库中
        session.insert("com.xzy.mapper.UserDao.insertSelective", user);
        UserDao mapper = session.getMapper(UserDao.class);
        User userDemo = mapper.selectByPrimaryKey(3L);
        System.out.println(JSONObject.toJSON(userDemo));
        User user2 = new User();
        user2.setName("boy2");
        // 这里测试暂用明文
        user2.setPassword("123456");
        user2.setMobile("18656400001");
        int i = mapper.insertSelective(user2);
        //提交事务,这个是必须要的,否则即使sql发了也保存不到数据库中
        session.commit();
        //关闭资源
        session.close();
#### 2.类加载器获取io
io包中封装的Resources，Resource中有静态变量 private static ClassLoaderWrapper classLoaderWrapper = new ClassLoaderWrapper();

    // 自定义的ClassLoaderWrapper，相当于类加载的装饰器，并完成其方法的封装，如getResourceAsStream方法，默认取的是系统类加载器
    ClassLoaderWrapper() {
      try {
        systemClassLoader = ClassLoader.getSystemClassLoader();
      } catch (SecurityException ignored) {
        // AccessControlException on Google App Engine   
      }
    }
    
#### 3.初始化配置
    初始化mybatis-config.xml和UserDao.xml配置:
   
    public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
        try {
        // 新建XMLConfigBuilder构建器来初始化mybatis-config.xml
          XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
          return build(parser.parse());
        } catch (Exception e) {
          throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
          ErrorContext.instance().reset();
          try {
            inputStream.close();
          } catch (IOException e) {
            // Intentionally ignore. Prefer previous error.
          }
        }
      }
      
    。。。
    XMLConfigBuilder的方法： XMLConfigBuilder继承与BaseBuilder，BaseBuilder 保存 读取并解析的配置
    public class XMLConfigBuilder extends BaseBuilder 
    
    public abstract class BaseBuilder {
            protected final Configuration configuration;
            protected final TypeAliasRegistry typeAliasRegistry;
            protected final TypeHandlerRegistry typeHandlerRegistry;
            
            ......
          }
    
     public Configuration parse() {
        if (parsed) {
          throw new BuilderException("Each XMLConfigBuilder can only be used once.");
        }
        parsed = true;
        //   parser.evalNode解析成Xnode,然后读取到configuration等配置       
        parseConfiguration(parser.evalNode("/configuration"));
        return configuration;
     }
    
     private void parseConfiguration(XNode root) {
        try {
          //issue #117 read properties first
          // 解析properties标签
          propertiesElement(root.evalNode("properties"));
          // 解析settings标签
          Properties settings = settingsAsProperties(root.evalNode("settings"));
          loadCustomVfs(settings);
          // 解析typeAliases标签
          typeAliasesElement(root.evalNode("typeAliases"));
          pluginElement(root.evalNode("plugins"));
          objectFactoryElement(root.evalNode("objectFactory"));
          objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
          reflectorFactoryElement(root.evalNode("reflectorFactory"));
          settingsElement(settings);
          // read it after objectFactory and objectWrapperFactory issue #631
          environmentsElement(root.evalNode("environments"));
          databaseIdProviderElement(root.evalNode("databaseIdProvider"));
          typeHandlerElement(root.evalNode("typeHandlers"));
          // 解析UserDao.xml等xml
          mapperElement(root.evalNode("mappers"));
        } catch (Exception e) {
          throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
        }
     }
     
     // 已解析properties标签的方法为例
      private void propertiesElement(XNode context) throws Exception {
         if (context != null) {
           // 解析properties的子节点property,读name,vlue等属性并记录到Properties
           Properties defaults = context.getChildrenAsProperties();
           // 解析resource属性
           String resource = context.getStringAttribute("resource");
            // 解析url属性
           String url = context.getStringAttribute("url");
           // url和resource不能同时为空
           if (resource != null && url != null) {
             throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
           }
           if (resource != null) {
             defaults.putAll(Resources.getResourceAsProperties(resource));
           } else if (url != null) {
             defaults.putAll(Resources.getUrlAsProperties(url));
           }
           // 将configuration中的variable和配置中的信息合并
           Properties vars = configuration.getVariables();
           if (vars != null) {
             defaults.putAll(vars);
           }
           // set到parser和configuration的variables属性中
           parser.setVariables(defaults);
           configuration.setVariables(defaults);
         }
      }
      
      // 解析UserDao.xml等的sql配置xml
        private void mapperElement(XNode parent) throws Exception {
          if (parent != null) {
            for (XNode child : parent.getChildren()) {
              if ("package".equals(child.getName())) {
              // 读取包名那么并set到configuration的Mappers属性
                String mapperPackage = child.getStringAttribute("name");
                configuration.addMappers(mapperPackage);
              } else {
                //获取resource,url,class等属性
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                if (resource != null && url == null && mapperClass == null) {
                  ErrorContext.instance().resource(resource);
                  InputStream inputStream = Resources.getResourceAsStream(resource);
                  // 实例化XMLMapperBuilder并解析mapper映射xml文件
                  XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
                  mapperParser.parse();
                } else if (resource == null && url != null && mapperClass == null) {
                  ErrorContext.instance().resource(url);
                  InputStream inputStream = Resources.getUrlAsStream(url);
                  XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
                  mapperParser.parse();
                } else if (resource == null && url == null && mapperClass != null) {
                  Class<?> mapperInterface = Resources.classForName(mapperClass);
                  configuration.addMapper(mapperInterface);
                } else {
                  throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
                }
              }
            }
          }
        }
      
      
      XMLMapperBuilder读取UserDao.xml
        public void parse() {
          if (!configuration.isResourceLoaded(resource)) {
            configurationElement(parser.evalNode("/mapper"));
            configuration.addLoadedResource(resource);
            bindMapperForNamespace();
          }
      
          parsePendingResultMaps();
          parsePendingChacheRefs();
          parsePendingStatements();
        }
        
      private void configurationElement(XNode context) {
         try {
            // 获取xml配置的namespace属性
            String namespace = context.getStringAttribute("namespace");
            if (namespace == null || namespace.equals("")) {
              throw new BuilderException("Mapper's namespace cannot be empty");
            }
            builderAssistant.setCurrentNamespace(namespace);
            // 解析cache-ref
            cacheRefElement(context.evalNode("cache-ref"));
             // 解析cache
            cacheElement(context.evalNode("cache"));
            // 解析parameterMap
            parameterMapElement(context.evalNodes("/mapper/parameterMap"));
            // 解析resultMap
            resultMapElements(context.evalNodes("/mapper/resultMap"));
            sqlElement(context.evalNodes("/mapper/sql"));
            // 核心，解析select|insert|update|delete等标签
            buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
         } catch (Exception e) {
            throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
         }
      }
      
        private void buildStatementFromContext(List<XNode> list) {
          if (configuration.getDatabaseId() != null) {
            buildStatementFromContext(list, configuration.getDatabaseId());
          }
          buildStatementFromContext(list, null);
        }
      
        private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
          for (XNode context : list) {
            final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
            try {
              statementParser.parseStatementNode();
            } catch (IncompleteElementException e) {
              configuration.addIncompleteStatement(statementParser);
            }
          }
        }
        
        
    XMLStatementBuilder解析xml中的sql
    
    // 解析select|insert|update|delete等标签
    public void parseStatementNode() {
        String id = context.getStringAttribute("id");
        String databaseId = context.getStringAttribute("databaseId");
        // 验证id是否相等
        if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
          return;
        }
        // 获取各属性
        Integer fetchSize = context.getIntAttribute("fetchSize");
        Integer timeout = context.getIntAttribute("timeout");
        String parameterMap = context.getStringAttribute("parameterMap");
        String parameterType = context.getStringAttribute("parameterType");
        Class<?> parameterTypeClass = resolveClass(parameterType);
        String resultMap = context.getStringAttribute("resultMap");
        String resultType = context.getStringAttribute("resultType");
        String lang = context.getStringAttribute("lang");
        LanguageDriver langDriver = getLanguageDriver(lang);
    
        Class<?> resultTypeClass = resolveClass(resultType);
        String resultSetType = context.getStringAttribute("resultSetType");
        StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
        ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    
        String nodeName = context.getNode().getNodeName();
        SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
        boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
        boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
        boolean useCache = context.getBooleanAttribute("useCache", isSelect);
        boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);
    
        // Include Fragments before parsing
        XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
        includeParser.applyIncludes(context.getNode());
    
        // Parse selectKey after includes and remove them.
        processSelectKeyNodes(id, parameterTypeClass, langDriver);
        
        // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
        SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
        String resultSets = context.getStringAttribute("resultSets");
        String keyProperty = context.getStringAttribute("keyProperty");
        String keyColumn = context.getStringAttribute("keyColumn");
        KeyGenerator keyGenerator;
        String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
        keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
        if (configuration.hasKeyGenerator(keyStatementId)) {
          keyGenerator = configuration.getKeyGenerator(keyStatementId);
        } else {
          keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
              configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
              ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
        }
    
        // 将解析后的属性存入configuration的MappedStatement属性中
        builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
            fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
            resultSetTypeEnum, flushCache, useCache, resultOrdered, 
            keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
      }
      
  初始化后configuration的值：
    ![configuration1](/images/posts/mybatis/mybatis002.png)  
    ![configuration2](/images/posts/mybatis/mybatis003.png)  
  
  可以看到   
  environment属性已经存入了datasource等数据库配置信息   
  mappedStatement属性存了sql和执行方的方法名的map信息   
  sqlFragments存放sql查询的元素   
  resultMaps存放resultMap的map信息   
  。。。   
  这样我们的sqlSessionFactory就已经build了configration对象信息完成了初始化
  
#### 4.代理
    上面获取的SqlSessionFactory默认是DefaultSqlSessionFactory，并已经完成了configuration的初始化
    public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
    }
    
    然后通过DefaultSqlSessionFactory来new DefaultSqlSession(configuration, executor, autoCommit);是个抽象工厂实现创建DefaultSqlSession的过程
    
    下面就是获取代理mapper
    
    DefaultSqlSession的getMapper方法，实际是调用onfiguration的getMapper方法
      @Override
      public <T> T getMapper(Class<T> type) {
        return configuration.<T>getMapper(type, this);
      }
    
    Configuration的mapper方法，调用的是MapperRegistry的getMapper方法
      public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        return mapperRegistry.getMapper(type, sqlSession);
      }
      
    MapperRegistry对象已经初始化了，并存了configuration配置 以及存了Dao接口和MapperProxyFactory的Map容器
      @SuppressWarnings("unchecked")
      public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
          throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        }
        try {
          return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
          throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
      }
    
    MapperProxyFactory代理工厂再创建对象
      @SuppressWarnings("unchecked")
      protected T newInstance(MapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
      }
    
      public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
      }
    
    这里实例化了MapperProxy并完成了对目标接口的动态代理
    当代理mapper执行方法，开始代理
    
    public class MapperProxy<T> implements InvocationHandler, Serializable {
    
      private static final long serialVersionUID = -6424540398559729838L;
      private final SqlSession sqlSession;
      private final Class<T> mapperInterface;
      private final Map<Method, MapperMethod> methodCache;
    
      public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
        this.methodCache = methodCache;
      }
    
      @Override
      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
          // 判断执行方法是否一致
          if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
          } else if (isDefaultMethod(method)) {
            return invokeDefaultMethod(proxy, method, args);
          }
        } catch (Throwable t) {
          throw ExceptionUtil.unwrapThrowable(t);
        }
        // 从缓存容器中获取MapperMethod，有则去缓存，没有则新建并存入缓存容器
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        // 执行方法对应的sql
        return mapperMethod.execute(sqlSession, args);
      }
    
    这里用的是java自带的动态代理，即对接口代理，实现InvocationHandler，并重写invoke方法。
    
     
    
    public class MapperMethod {
    
      private final SqlCommand command;
      private final MethodSignature method;
    
      public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
        this.command = new SqlCommand(config, mapperInterface, method);
        this.method = new MethodSignature(config, mapperInterface, method);
      }
    
      public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        // 根据类型调用不同的方法
        switch (command.getType()) {
          case INSERT: {
        	Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
          }
          case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
          }
          case DELETE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
          }
          case SELECT:
            // 判断返回类型，调用不同的执行方法
            if (method.returnsVoid() && method.hasResultHandler()) {
              executeWithResultHandler(sqlSession, args);
              result = null;
            } else if (method.returnsMany()) {
              result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
              result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) {
              result = executeForCursor(sqlSession, args);
            } else {
            // 处理返回单一对象的情况
            // 参数解析器解析参数
              Object param = method.convertArgsToSqlCommandParam(args);
              // 执行单一对象返回的查询
              result = sqlSession.selectOne(command.getName(), param);
            }
            break;
          case FLUSH:
            result = sqlSession.flushStatements();
            break;
          default:
            throw new BindingException("Unknown execution method for: " + command.getName());
        }
        if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
          throw new BindingException("Mapper method '" + command.getName() 
              + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
        }
        return result;
      }
      
      ......
    }
    
    
    // 验证是否返回一个，多个报错，底层调的还是selectList
      @Override
      public <T> T selectOne(String statement, Object parameter) {
        // Popular vote was to return null on 0 results and throw exception on too many.
        List<T> list = this.<T>selectList(statement, parameter);
        if (list.size() == 1) {
          return list.get(0);
        } else if (list.size() > 1) {
          throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
        } else {
          return null;
        }
      }
  
      @Override
      public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
        try {
        // 这里从mappedStatement里获取要执行的封装的SQL
          MappedStatement ms = configuration.getMappedStatement(statement);
          // 执行器执行
          return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
        } catch (Exception e) {
          throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
        } finally {
          ErrorContext.instance().reset();
        }
      }
      
 
#### 5.执行器执行SQL并封装返回
     
      这里是构建sqlSession前，实例化执行器。
      public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
        executorType = executorType == null ? defaultExecutorType : executorType;
        executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
        Executor executor;
        // 根据不同执行类型创建不同的执行器
        if (ExecutorType.BATCH == executorType) {
          executor = new BatchExecutor(this, transaction);
        } else if (ExecutorType.REUSE == executorType) {
          executor = new ReuseExecutor(this, transaction);
        } else {
          // 默认本次就是Simple执行器
          executor = new SimpleExecutor(this, transaction);
        }
        // 执行器缓存，已有则取缓存
        if (cacheEnabled) {
          executor = new CachingExecutor(executor);
        }
        executor = (Executor) interceptorChain.pluginAll(executor);
        return executor;
      }
      
      // 这是封装Sql的实体bean
      public class BoundSql {
      
        private String sql;
        private List<ParameterMapping> parameterMappings;
        private Object parameterObject;
        private Map<String, Object> additionalParameters;
        private MetaObject metaParameters;
       。。。
      }
      CachingExecutor的query方法
     
        @Override
        public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
          // 封装并获取BoundSql
          BoundSql boundSql = ms.getBoundSql(parameterObject);
          // 拼接缓存的key
          CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
          // 执行sql
          return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        }
      
      
      CachingExecutor的query
      @Override
        public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
            throws SQLException {
          Cache cache = ms.getCache();
          if (cache != null) {
            flushCacheIfRequired(ms);
            if (ms.isUseCache() && resultHandler == null) {
              ensureNoOutParams(ms, parameterObject, boundSql);
              @SuppressWarnings("unchecked")
              // 先从缓存中拿
              List<E> list = (List<E>) tcm.getObject(cache, key);
              if (list == null) {
                list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                tcm.putObject(cache, key, list); // issue #578 and #116
              }
              return list;
            }
          }
          return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        }
      
      
      缓存为null,调用delegate即BaseExecutor的query方法
      @SuppressWarnings("unchecked")
        @Override
        public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
          ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
          // 检查Executor是否关闭
          if (closed) {
            throw new ExecutorException("Executor was closed.");
          }
          if (queryStack == 0 && ms.isFlushCacheRequired()) {
            clearLocalCache();
          }
          List<E> list;
          try {
          // 查询层级+1
            queryStack++;
            list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
            if (list != null) {
            // 处理调用存储过程的结果处理
              handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
            } else {
              // 没缓存，从数据库加载
              list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
            }
          } finally {
            queryStack--;
          }
          if (queryStack == 0) {
            for (DeferredLoad deferredLoad : deferredLoads) {
              deferredLoad.load();
            }
            // issue #601
            deferredLoads.clear();
            if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
              // issue #482
              clearLocalCache();
            }
          }
          return list;
      
      
      private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
          List<E> list;
          // 缓存中添加占位符
          localCache.putObject(key, EXECUTION_PLACEHOLDER);
          try {
          // SQL查询
            list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
          } finally {
            // 缓存中删除占位符
            localCache.removeObject(key);
          }
          // 缓存中放入查询结果
          localCache.putObject(key, list);
          if (ms.getStatementType() == StatementType.CALLABLE) {
            localOutputParameterCache.putObject(key, parameter);
          }
          return list;
        }
       
       
       SimpleExecutor执行doQuery
         @Override
         public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
           Statement stmt = null;
           try {
             Configuration configuration = ms.getConfiguration();
             StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
             // 获取connection连接 处理占位符等
             stmt = prepareStatement(handler, ms.getStatementLog());
             // 处理完毕，执行sql
             return handler.<E>query(stmt, resultHandler);
           } finally {
             closeStatement(stmt);
           }
         }
         
         private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
           Statement stmt;
           // 获取connection
           Connection connection = getConnection(statementLog);
           stmt = handler.prepare(connection, transaction.getTimeout());
           // 处理占位符
           handler.parameterize(stmt);
           return stmt;
         }
         
         获取connection
           protected Connection getConnection(Log statementLog) throws SQLException {
             Connection connection = transaction.getConnection();
             // 如果支持debug,则获取debug代理连接，添加日志等
             if (statementLog.isDebugEnabled()) {
               return ConnectionLogger.newInstance(connection, statementLog, queryStack);
             } else {
               return connection;
             }
           }
           
         DefaultParameterHandler处理占位符
           @Override
           public void setParameters(PreparedStatement ps) {
             ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
             List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
             if (parameterMappings != null) {
               // 遍历parameterMappings参数列表，并设置到PreparedStatement中
               for (int i = 0; i < parameterMappings.size(); i++) {
                 ParameterMapping parameterMapping = parameterMappings.get(i);
                 if (parameterMapping.getMode() != ParameterMode.OUT) {
                 // 不处理存储过程
                   Object value;
                   String propertyName = parameterMapping.getProperty();
                   if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
                     value = boundSql.getAdditionalParameter(propertyName);
                   } else if (parameterObject == null) {
                     value = null;
                   } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                     value = parameterObject;
                   } else {
                     MetaObject metaObject = configuration.newMetaObject(parameterObject);
                     value = metaObject.getValue(propertyName);
                   }
                   TypeHandler typeHandler = parameterMapping.getTypeHandler();
                   JdbcType jdbcType = parameterMapping.getJdbcType();
                   if (value == null && jdbcType == null) {
                     jdbcType = configuration.getJdbcTypeForNull();
                   }
                   try {
                   // 占位符绑定参数
                     typeHandler.setParameter(ps, i + 1, value, jdbcType);
                   } catch (TypeException e) {
                     throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
                   } catch (SQLException e) {
                     throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
                   }
                 }
               }
             }
           }
      
      执行SQL
        @Override
        public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
          PreparedStatement ps = (PreparedStatement) statement;
          // jdbc执行sql
          ps.execute();
          // 封装返回结果
          return resultSetHandler.<E> handleResultSets(ps);
        }
        
       
       返回结果封装 
       //
        // HANDLE RESULT SETS
        //
        @Override
        public List<Object> handleResultSets(Statement stmt) throws SQLException {
          ErrorContext.instance().activity("handling results").object(mappedStatement.getId());
      
          final List<Object> multipleResults = new ArrayList<Object>();
      
          int resultSetCount = 0;
          ResultSetWrapper rsw = getFirstResultSet(stmt);
      
          List<ResultMap> resultMaps = mappedStatement.getResultMaps();
          int resultMapCount = resultMaps.size();
          validateResultMapsCount(rsw, resultMapCount);
          while (rsw != null && resultMapCount > resultSetCount) {
            ResultMap resultMap = resultMaps.get(resultSetCount);
            handleResultSet(rsw, resultMap, multipleResults, null);
            rsw = getNextResultSet(stmt);
            cleanUpAfterHandlingResultSet();
            resultSetCount++;
          }
      
          String[] resultSets = mappedStatement.getResultSets();
          if (resultSets != null) {
            while (rsw != null && resultSetCount < resultSets.length) {
              ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
              if (parentMapping != null) {
                String nestedResultMapId = parentMapping.getNestedResultMapId();
                ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
                handleResultSet(rsw, resultMap, null, parentMapping);
              }
              rsw = getNextResultSet(stmt);
              cleanUpAfterHandlingResultSet();
              resultSetCount++;
            }
          }
      
          return collapseSingleResultList(multipleResults);
        }
 
到此看完了查询SQL的源码过程,insert,update等类似 
    
### 用到的设计模式 
单例模式，工厂模式，创建者模式，装饰器模式，代理模式，模板方法模式，适配器模式，组合模式，迭代器模式

设计模式的使用使代码更加规范，扩展性更强。   
学习其他框架源码也更加得心应手，以后也更容易看懂大佬写的代码，实现从 看懂 -> 自己会写 ->创新 这个成为大佬的过程

#### 单例模式
ErrorContext，LogFactory

ErrorContext 错误上下文，记录该线程的执行环境错误信息   
其单例模式是懒汉式写法，需要注意的是用了ThreadLocal，说明每个线程都有自己的一个ErrorContext

    public class ErrorContext {
    
      private static final String LINE_SEPARATOR = System.getProperty("line.separator","\n");
      private static final ThreadLocal<ErrorContext> LOCAL = new ThreadLocal<ErrorContext>();
    
      private ErrorContext stored;
      private String resource;
      private String activity;
      private String object;
      private String message;
      private String sql;
      private Throwable cause;
    
      private ErrorContext() {
      }
    
      public static ErrorContext instance() {
        ErrorContext context = LOCAL.get();
        if (context == null) {
          context = new ErrorContext();
          LOCAL.set(context);
        }
        return context;
      }
      ...
    }
    

#### 工厂模式
SqlSessionFactory、ObjectFactory、MapperProxyFactory， DataSourceFactory， ExceptionFactory等

SqlSessionFactory和SqlSessionFactoryBuilder完成框架中常见的构建模式构建工厂   
SqlSessionFactory接口有DefaultSqlSessionFactory等实现来new DefaultSqlSession,是类似于抽象工厂的实现DefaultSqlSession是具体产品，DefaultSqlSessionFactory具体产品工厂

MapperProxyFactory通过newInstance创建代理

ExceptionFactory，可通过wrapException方法创建PersistenceException，并传入到错误上下文

    public class ExceptionFactory {
    
      private ExceptionFactory() {
        // Prevent Instantiation
      }
    
      public static RuntimeException wrapException(String message, Exception e) {
        return new PersistenceException(ErrorContext.instance().message(message).cause(e).toString(), e);
      }
    
    }

#### 创建者模式 
SqlSessionFactoryBuilder, BaseBuilder , XMLConfigBuilder, XMLMapperBuilder, XMLStatementBuilder, CacheBuilder等

Builder就太多了，很多类都包含着其他的类来完成初始化，因此可以创建多个Builder

如BaseBuilder , XMLConfigBuilder, XMLMapperBuilder, XMLStatementBuilder这些Builder会读取文件或者配置，然后做大量的XpathParser解析、配置或语法的解析、反射生成对象、存入结果缓存等步骤


#### 装饰器模式
cache包中的cache.decorators中的各个装饰者，TransactionalCache等

装饰器特征是包装原类，构造初始化，调用原方法进行方法增强   

如下LoggingCache，通过构造初始化获取Cache的实现，就可以调用其实现。

    public class LoggingCache implements Cache {
    
      private Log log;  
      private Cache delegate;
      protected int requests = 0;
      protected int hits = 0;
    
      public LoggingCache(Cache delegate) {
        this.delegate = delegate;
        this.log = LogFactory.getLog(getId());
      }
    
      @Override
      public String getId() {
        return delegate.getId();
      }
    
      @Override
      public int getSize() {
        return delegate.getSize();
      }
     。。。。。。
     
     }

mybatis缓存分为一级缓存和二级缓存

一级缓存，又叫本地缓存，是PerpetualCache类型的永久缓存，保存在执行器中（BaseExecutor），而执行器又在SqlSession（DefaultSqlSession）中，所以一级缓存的生命周期与SqlSession是相同的。

二级缓存，又叫自定义缓存，实现了Cache接口的类都可以作为二级缓存，所以可配置如encache等的第三方缓存。二级缓存以namespace名称空间为其唯一标识，被保存在Configuration核心配置对象中。

二级缓存对象的默认类型为PerpetualCache，如果配置的缓存是默认类型，则mybatis会根据配置自动追加一系列装饰器。

#### 代理模式
Mybatis实现的核心，完成了面向接口的编程，比如MapperProxy、ConnectionLogger，用的jdk的动态代理；   
还有executor.loader包使用了cglib或者javassist达到延迟加载的效果

Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy)利用了java的动态代理技术


    public class MapperProxyFactory<T> {
  
    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<Method, MapperMethod>();
  
    public MapperProxyFactory(Class<T> mapperInterface) {
      this.mapperInterface = mapperInterface;
    }
  
    public Class<T> getMapperInterface() {
      return mapperInterface;
    }
  
    public Map<Method, MapperMethod> getMethodCache() {
      return methodCache;
    }
  
    @SuppressWarnings("unchecked")
    protected T newInstance(MapperProxy<T> mapperProxy) {
      return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
    }
  
    public T newInstance(SqlSession sqlSession) {
      final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
      return newInstance(mapperProxy);
    }
  
  }
#### 模板方法模式
BaseExecutor和SimpleExecutor，BaseTypeHandler和所有的子类例如IntegerTypeHandler；

模板方法就像Servlet的service定义doGet和doPost一样，把一个方法中的一些功能抽象出来，交给子类来实现，

如下面BaseExecutor的代码，将doQuery分离出来交给SimpleExecutor等子类实现

    public abstract class BaseExecutor implements Executor
     
       protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
           throws SQLException;
     。。。。。。
     
     private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
       List<E> list;
       localCache.putObject(key, EXECUTION_PLACEHOLDER);
       try {
         list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
       } finally {
         localCache.removeObject(key);
       }
       localCache.putObject(key, list);
       if (ms.getStatementType() == StatementType.CALLABLE) {
         localOutputParameterCache.putObject(key, parameter);
       }
       return list;
     }
     
     。。。。。。


#### 适配器模式
Log的Mybatis接口和它对jdbc、log4j等各种日志框架的适配实现

适配器模式的目的是不修改原类的代码，封装一个可以调用老类完成新功能的新适配器类   
所以第三方包无法修改，又需要使用时可以通过该适配器来封装

如Log4jImpl用了org.apache.ibatis.logging.Log，通过构造组合的方式来初始化，并封装其log等方法，扩展为error,debug等方法


#### 组合模式
SqlNode和各个子类ChooseSqlNode， IfSqlNode，TextSqlNode

组合模式可统一处理层级结构，树形结构等。如目录和各类文件夹

mybatis的xml有自己的语法，如if, choose, trim等结构，根据条件来生成不同情况下的SQL

SqlNode接口，子类是ChooseSqlNode，IfSqlNode，TextSqlNode等类

    public interface SqlNode {
      boolean apply(DynamicContext context);
    }
    
如下IfSqlNode代码，就需要先做判断，如果判断通过，仍然会调用子元素的SqlNode，即contents.apply方法，实现递归的解析

    public class IfSqlNode implements SqlNode {
      private ExpressionEvaluator evaluator;
      private String test;
      private SqlNode contents;
    
      public IfSqlNode(SqlNode contents, String test) {
        this.test = test;
        this.contents = contents;
        this.evaluator = new ExpressionEvaluator();
      }
    
      @Override
      public boolean apply(DynamicContext context) {
        if (evaluator.evaluateBoolean(test, context.getBindings())) {
          contents.apply(context);
          return true;
        }
        return false;
      }
    
    }

ChooseSqlNode，

    public class ChooseSqlNode implements SqlNode {
      private SqlNode defaultSqlNode;
      private List<SqlNode> ifSqlNodes;
    
      public ChooseSqlNode(List<SqlNode> ifSqlNodes, SqlNode defaultSqlNode) {
        this.ifSqlNodes = ifSqlNodes;
        this.defaultSqlNode = defaultSqlNode;
      }
    
      @Override
      public boolean apply(DynamicContext context) {
        for (SqlNode sqlNode : ifSqlNodes) {
          if (sqlNode.apply(context)) {
            return true;
          }
        }
        if (defaultSqlNode != null) {
          defaultSqlNode.apply(context);
          return true;
        }
        return false;
      }
    }
 
TextSqlNode

    public class TextSqlNode implements SqlNode {
      private String text;
      private Pattern injectionFilter;
    
      public TextSqlNode(String text) {
        this(text, null);
      }
      
      public TextSqlNode(String text, Pattern injectionFilter) {
        this.text = text;
        this.injectionFilter = injectionFilter;
      }
      
      public boolean isDynamic() {
        DynamicCheckerTokenParser checker = new DynamicCheckerTokenParser();
        GenericTokenParser parser = createParser(checker);
        parser.parse(text);
        return checker.isDynamic();
      }
    
      @Override
      public boolean apply(DynamicContext context) {
        GenericTokenParser parser = createParser(new BindingTokenParser(context, injectionFilter));
        context.appendSql(parser.parse(text));
        return true;
      }
      ......
    }   
#### 迭代器模式
PropertyTokenizer等

迭代器用于容器的遍历，同时不暴露内部细节

    public class PropertyTokenizer implements Iterator<PropertyTokenizer> {
      private String name;
      private String indexedName;
      private String index;
      private String children;
    
      public PropertyTokenizer(String fullname) {
        int delim = fullname.indexOf('.');
        if (delim > -1) {
          name = fullname.substring(0, delim);
          children = fullname.substring(delim + 1);
        } else {
          name = fullname;
          children = null;
        }
        indexedName = name;
        delim = name.indexOf('[');
        if (delim > -1) {
          index = name.substring(delim + 1, name.length() - 1);
          name = name.substring(0, delim);
        }
      }
    
      public String getName() {
        return name;
      }
    
      public String getIndex() {
        return index;
      }
    
      public String getIndexedName() {
        return indexedName;
      }
    
      public String getChildren() {
        return children;
      }
    
      @Override
      public boolean hasNext() {
        return children != null;
      }
    
      @Override
      public PropertyTokenizer next() {
        return new PropertyTokenizer(children);
      }
    
      @Override
      public void remove() {
        throw new UnsupportedOperationException("Remove is not supported, as it has no meaning in the context of properties.");
      }
    }


