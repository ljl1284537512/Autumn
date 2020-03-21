# MyBatis 懒加载实现原理分析

>   这篇文章是通过Mapper方式查询来引入懒加载分析。
>
>   前提知识 : 了解Mapper方式查询的流程；了解Javassist动态代理

我们的入口是DefaultSqlSession的SelectOne方法：

```java
@Override
public <T> T selectOne(String statement, Object parameter) {
  // Popular vote was to return null on 0 results and throw exception on too many.
  List<T> list = this.selectList(statement, parameter);
  if (list.size() == 1) {
    return list.get(0);
  } else if (list.size() > 1) {
    throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
  } else {
    return null;
  }
}
```

因为我们主要是看查询的**流程**，所以关注点是在**selectList**上，我们继续深入;

```java
@Override
public <E> List<E> selectList(String statement, Object parameter) {
  return this.selectList(statement, parameter, RowBounds.DEFAULT);
}

@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

首先从configuration中查询MappedStatement，这个MappedStatement相当于封装了某条SQL语句在Mapper.xml中的配置。

注意，rowBounds默认是使用RowBounds.DEFAULT，同样默认使用Executor.NO_RESULT_HANDLER作为结果处理器(其实就是null)；

因为Executor有可能是由于开启了二级缓存导致被CachingExecutor所装饰，我们可以直接忽略二级缓存，因为首次查询都会被委派到BaseExecutor中查询；

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameter);
  CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
  return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  if (queryStack == 0 && ms.isFlushCacheRequired()) {
    clearLocalCache();
  }
  List<E> list;
  try {
    queryStack++;
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
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
}
```

上面的代码看起来比较复杂，因为涉及到了一级缓存，在首次查询的情况下，入口点是在queryFromDatabase，也就是直接查询数据库；

```java
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

@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
  Statement stmt = null;
  try {
    Configuration configuration = ms.getConfiguration();
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    stmt = prepareStatement(handler, ms.getStatementLog());
    return handler.query(stmt, resultHandler);
  } finally {
    closeStatement(stmt);
  }
}
```

以上第一个方法逻辑主要涉及缓存，我们直接看调用到的第二个方法:

1.  先从mappedStatement中获取Configuration;
2.  生成一个StatementHandler，顾名思义就是处理Statement用的。
3.  接着调用prepareStatement，其实就是要获取连接，从而生成Statement,和我们jdbc操作类似, 如下：

```java
// 打开连接
conn = DriverManager.getConnection("jdbc:mysql://", "root", "123456");
// 执行查询
stmt = conn.createStatement();
```

4.  调用RoutingStatementHandler委派PreparedStatementHandler执行的query方法；

```java
@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  ps.execute();
  return resultSetHandler.handleResultSets(ps);
}
```

在PreparedStatementHandler中，首先恢复类型，执行完execute()方法之后，调用DefaultResultSetHandler的resultSetHandler去处理；

>   为什么是DefaultResultSetHandler？ 因为resultSetHandler是PreparedStatementHandler的成员属性，所以需要看PreparedStatementHandler的构造:
>
>   ```java
>   this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
>   ```
>
>   最终的resultSetHandler其实就是DefaultResultSetHandler；
>
>   ```java
>   ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
>   ```

我们接着看handleResultSets的处理;

```java
@Override
public List<Object> handleResultSets(Statement stmt) throws SQLException {
  ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

  final List<Object> multipleResults = new ArrayList<>();

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
  ...
}
```

1.  从Statement中获取ResultSetWrapper;
2.  取出MappedStatement中配置的ResultMap；
3.  根据查询的结果，以及配置中的ResultMap进行解析；

我们需要深入进去看看解析的结果;

```java
private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
  try {
    if (parentMapping != null) {
      handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
    } else {
      if (resultHandler == null) {
        DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
        handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
        multipleResults.add(defaultResultHandler.getResultList());
      } else {
        handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
      }
    }
  } finally {
    // issue #228 (close resultsets)
    closeResultSet(rsw.getResultSet());
  }
}
```

因为最早我们提过ResultSet其实是Executor.NO_RESULT_HANDLER = null, 所以构造了DefaultResultHandler，紧接着处理每一行的数据，我们继续看handleRowValues的内部实现；

```java
public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
  if (resultMap.hasNestedResultMaps()) {
    ensureNoRowBounds();
    checkResultHandler();
    handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
  } else {
    handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
  }
}
```

这里的`if (resultMap.hasNestedResultMaps())`其实是MappedStatement中配置内嵌resultMap，因为我的实例中并没有，所以，执行handleRowValuesForSimpleResultMap；

```java
private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
    throws SQLException {
  DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
  ResultSet resultSet = rsw.getResultSet();
  skipRows(resultSet, rowBounds);
  while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
    ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
    Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
    storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
  }
}

private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
  final ResultLoaderMap lazyLoader = new ResultLoaderMap();
  Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
  if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
    final MetaObject metaObject = configuration.newMetaObject(rowValue);
    boolean foundValues = this.useConstructorMappings;
    if (shouldApplyAutomaticMappings(resultMap, false)) {
      foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
    }
    foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
    foundValues = lazyLoader.size() > 0 || foundValues;
    rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
  }
  return rowValue;
}
```

从以上方法中直接看调用getRowValue的内部实现；

1.  首先构造出一个ResultLoaderMap，我们暂时不需要管它；
2.  将rowValue封装到MetaObject中；

3.  createResultObject(rsw, resultMap, lazyLoader, columnPrefix);返回了一个rowValue；我们猜测这是sql执行的一行结果，解析之后的数据要放入的对象(其实就是我们配置的ResultMap实体类)；

4.  接着核心的地方就是applyPropertyMappings方法，顾名思义是要去将属性的映射应用到对象中，lazyLoader也被作为参数传递进去；

```java
private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
    throws SQLException {
  // 获取所有要解析的数据库字段名称
  final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
  // 目前还没有找到值，打标
  boolean foundValues = false;
  // 获取所有的属性映射
  final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
  for (ResultMapping propertyMapping : propertyMappings) {
    // 遍历要查询的属性映射
    String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
    if (propertyMapping.getNestedResultMapId() != null) {
      // the user added a column attribute to a nested result map, ignore it
      column = null;
    }
    if (propertyMapping.isCompositeResult()
        || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
        || propertyMapping.getResultSet() != null) {
      // 查询属性的值
      Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
      // issue #541 make property optional
      final String property = propertyMapping.getProperty();
      if (property == null) {
        continue;
      } else if (value == DEFERRED) {
        foundValues = true;
        continue;
      }
      if (value != null) {
        foundValues = true;
      }
      // 如果值不为空，则设置到结果实体里
      if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
        // gcode issue #377, call setter on nulls (value is not 'found')
        metaObject.setValue(property, value);
      }
    }
  }
  return foundValues;
}
```

applyPropertyMappings方法的核心就是将resultSet中的数据一对一的设置到结果实体中。但是我们发现这个方法里面有一处部分比较可疑：

```java
} else if (value == DEFERRED) {
  foundValues = true;
  continue;
}
```

如果property == null的话, 执行continue还没问题，但是这个DEFERRED常量是从哪儿来的？我们直接取看getPropertyMappingValue方法，虽然我们知道这个方法大致就是获取属性的值；

```java
private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
    throws SQLException {
  // 这个地方被标记出来了，正好是符合我们的内嵌查询的情况
  if (propertyMapping.getNestedQueryId() != null) {
    return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
  } else if (propertyMapping.getResultSet() != null) {
    addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
    return DEFERRED;
  } else {
    final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
    final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
    return typeHandler.getResult(rs, column);
  }
}
```

getNestedQueryMappingValue像是要获取内嵌查询的对象值；

```java
private Object getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
    throws SQLException {
  // 内嵌查询的StatementId
  final String nestedQueryId = propertyMapping.getNestedQueryId();
  // 获取属性名
  final String property = propertyMapping.getProperty();
  // 从configuration中获取对应的MappedStatement
  final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
  // 内嵌查询所需的参数类型
  final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
  // 内嵌查询所需的参数值
  final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, propertyMapping, nestedQueryParameterType, columnPrefix);
  Object value = null;
  if (nestedQueryParameterObject != null) {
    // 获取内嵌查询的boundSQL
    final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
    // 构造查询的Key
    final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
    // 内嵌查询的结果类型
    final Class<?> targetType = propertyMapping.getJavaType();
    if (executor.isCached(nestedQuery, key)) {
      // 如果有缓存，直接返回缓存
      executor.deferLoad(nestedQuery, metaResultObject, property, key, targetType);
      value = DEFERRED;
    } else {
      // 第一次查询，首先构造ResultLoader
      final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
      if (propertyMapping.isLazy()) {
        // 是否符合懒加载，如果是，则将resultLoader放入lazyLoader中。
        lazyLoader.addLoader(property, metaResultObject, resultLoader);
        value = DEFERRED;
      } else {
        value = resultLoader.loadResult();
      }
    }
  }
  return value;
}
```

我们发现，在懒加载的情况下，构造出ResultLoader出，将其放入lazyLoader，便返回DEFERRED了。并没有发现进一步的获取内嵌查询的数据，倒是在非懒加载的情况下，会执行resultLoader.loadResult();这里我们猜测是去加载内嵌查询的结果了。我们先不看这块，跟着懒加载直接返回；

再次回到刚刚这个地方:

```java
private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
  final ResultLoaderMap lazyLoader = new ResultLoaderMap();
  Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
  if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
    final MetaObject metaObject = configuration.newMetaObject(rowValue);
    boolean foundValues = this.useConstructorMappings;
    if (shouldApplyAutomaticMappings(resultMap, false)) {
      foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
    }
    foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
    foundValues = lazyLoader.size() > 0 || foundValues;
    rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
  }
  return rowValue;
}
```

因为上一步我们再lazyLoader中添加了ResultLoader，所以lazyLoader.size() > 0是成立的。返回rowValues;

到这里，就算了结束了，但是非常神奇啊，假设这个返回的rowValues是我们要的结果实体，那么调用其get内嵌对象的方法，只会返回null ，它是如何做二次查询的呢？

我们回到之前分析的代码, createResultObject真的只是返回查询结果对象吗？

```java
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
  this.useConstructorMappings = false; // reset previous mapping result
  final List<Class<?>> constructorArgTypes = new ArrayList<>();
  final List<Object> constructorArgs = new ArrayList<>();
  // resultObject真的只是结果集对象吗？
  Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
  if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
      // issue gcode #109 && issue #149
      if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
        resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
        break;
      }
    }
  }
  this.useConstructorMappings = resultObject != null && !constructorArgTypes.isEmpty(); // set current mapping result
  return resultObject;
}
```

我们看到

```java
if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
  resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
  break;
}
```

以上代码分析， 如果内嵌查询不为空，并且内嵌查询是懒加载的，就会通过configuration.getProxyFactory().createProxy重新创建一个对象，那最后的resultObject就被替换了。

我们再看内部的实现(默认是javassist实现)：

```java
@Override
public Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
  return EnhancedResultObjectProxyImpl.createProxy(target, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
}

public static Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      final Class<?> type = target.getClass();
      EnhancedResultObjectProxyImpl callback = new EnhancedResultObjectProxyImpl(type, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
      Object enhanced = crateProxy(type, callback, constructorArgTypes, constructorArgs);
      PropertyCopier.copyBeanProperties(type, target, enhanced);
      return enhanced;
    }
```

在JavassistProxyFactory中，通过EnhancedResultObjectProxyImpl创建了一个代理。而这个代理的回调类是EnhancedResultObjectProxyImpl，我们关注构造EnhancedResultObjectProxyImpl的lazyLoader参数，这个lazyLoader不就是Mybatis在发现有内嵌的懒加载类型属性时，放入Loader的对象吗？

```java
lazyLoader.addLoader(property, metaResultObject, resultLoader);
```

我们再看这个lazyLoader被拿来干嘛用了:

```java
private EnhancedResultObjectProxyImpl(Class<?> type, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
  this.type = type;
  this.lazyLoader = lazyLoader;
  this.aggressive = configuration.isAggressiveLazyLoading();
  this.lazyLoadTriggerMethods = configuration.getLazyLoadTriggerMethods();
  this.objectFactory = objectFactory;
  this.constructorArgTypes = constructorArgTypes;
  this.constructorArgs = constructorArgs;
}
```

被赋值到EnhancedResultObjectProxyImpl中。这个时候，就很清晰了，如果EnhancedResultObjectProxyImpl可以拿到LazyLoader， 那么在动态代理对象执行回调方法的时候，不就可以拿到这个lazyLoader了吗？我们去看EnhancedResultObjectProxyImpl的invoke方法(javasssit规范)；

```java
  @Override
  public Object invoke(Object enhanced, Method method, Method methodProxy, Object[] args) throws Throwable {
    final String methodName = method.getName();
    try {
      synchronized (lazyLoader) {
        if (WRITE_REPLACE_METHOD.equals(methodName)) {
				...
        } else {
          if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
            if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
              lazyLoader.loadAll();
            } else if (PropertyNamer.isSetter(methodName)) {
              final String property = PropertyNamer.methodToProperty(methodName);
              lazyLoader.remove(property);
            } else if (PropertyNamer.isGetter(methodName)) {
              final String property = PropertyNamer.methodToProperty(methodName);
              if (lazyLoader.hasLoader(property)) {
                lazyLoader.load(property);
              }
            }
          }
        }
      }
      return methodProxy.invoke(enhanced, args);
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
}
```

如果lazyLoader.size>0, 并且不是finalize方法， 则发现如果是getter方法，就会执行lazyLoader.load(property)方法。这一步跟之前分析的某个地方(当发现有内嵌循环，并且不是懒加载的情况)是一致的，我们看lazyLoader.load(property)的内部实现:

```java
public boolean load(String property) throws SQLException {
  LoadPair pair = loaderMap.remove(property.toUpperCase(Locale.ENGLISH));
  if (pair != null) {
    pair.load();
    return true;
  }
  return false;
}
```

其实当时放入loader的时候:

```java
loaderMap.put(upperFirst, new LoadPair(property, metaResultObject, resultLoader));
```

所以我们看LoadPair的load()方法:

```java
public void load(final Object userObject) throws SQLException {
  ...
  this.metaResultObject.setValue(property, this.resultLoader.loadResult());
}
```

重点就在最下面哪行，调用了resultLoader.loadResult()方法：

```java
public Object loadResult() throws SQLException {
  // 查询列表数据
  List<Object> list = selectList();
  resultObject = resultExtractor.extractObjectFromList(list, targetType);
  // 返回结果
  return resultObject;
}
```

总结: Mybatis的懒加载是在一定配置下生效的，在查询主体Statement的时候，生成ResultLoader主要完成内嵌查询，并在发现有懒加载的情况下(如果没有的话，解析属性时，直接调用ResultLoader完成子查询)，动态生成代理对象，将ResultLoader作为参数构造回调对象，该代理对象在其回调方法中进行校验，如果是getter方法，则直接调用ResultLoader进行查询。这样就能实现，在调用的过程中，才会触发内嵌查询。
