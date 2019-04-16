## Dubbo SPI 规范

SPI扩展点加载目录 

1. META-INF/services/

2. META-INF/dubbo/

3. META-INF/dubbo/internal



如何扩展：

1. 定义全路径接口名文件;

2. 实现内容例如:myProtocol=com.riseusn….CustomProtocol

   

## 源码分析

#### Dubbo的SPI入口

```java
// 获取协议的自适应扩展
ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()
```

> 和ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("")的区别是，getExtension("")加载一个指定名称的扩展点；



#### Dubbo Extension

> Dubbo的Extension是基于SPI来实现的。

- Factory
  - AdaptiveExtension
  - SPIExtension
- Extension
  - SpringExtension
- @SPI : 代表当前类是扩展点，注解的值为默认选择的扩展实现。
- @Adaptive
  - 如果是配置在类级别上，表示自定义适配器；
  - 如果是配置在方法级别上，表示需要动态生成适配器类；
- @Activate

@SPI : 代表该接口是扩展点

```java
@SPI("dubbo") //默认是dubbo
public interface Protocol {}
```

@Adaptive : 适配器注解;

```java
@SPI("dubbo")
public interface Protocol {

    int getDefaultPort();
		...
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
		...

}
```

getExtensionLoader : 根据一个类型，得到一个Extension;

```java
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type (" + type +
                    ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
        }
        //尝试从缓冲中获取
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            //缓冲中没有的话，实例化一个，并且放入缓存
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            //从缓冲中获取,返回实例
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

初始化ExtensionLoader:

```java
    /**
     * 扩展点加载器
     * @param type
     */
    private ExtensionLoader(Class<?> type) {
        //初始化ExtensionFactory的时候，type=ExtensionFactory
        this.type = type;
        //ExtensionFactory的objectFactory = null , 如果不是获取ExtensionFactory的扩展器, 则对应的objectFactory要根据ExtensionFactory获取
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```



getAdaptiveExtension（获取自适应扩展点）方法实现如下：

```java
    public T getAdaptiveExtension() {
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            //创建实例
                            instance = createAdaptiveExtension();
                            //放入缓存
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("Failed to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }
        //返回实例
        return (T) instance;
    }
```

创建自适应扩展点方式如下：

```java
    private T createAdaptiveExtension() {
        try {
            //注入实例(怎么注入？)
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
```

获取适配器类

```java
    private Class<?> getAdaptiveExtensionClass() {
      	// 记载扩展点的实现类
        getExtensionClasses();
      	//getExtensionClasses方法会缓存@Adaptive注解的类到cachedAdaptiveClass,也就是说如果SPI的扩展实现类的类上有@Adaptive注解，则直接返回cachedAdaptiveClass
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
```

创建适配器扩展点：

```java
    private Class<?> createAdaptiveExtensionClass() {
        //type=Protocol.class , cachedDefaultName="dubbo"
      	// 创建一个动态的字节码文件Protocol$Adaptive,该类实现Protocol接口
        String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
        // 获取类加载器
      	ClassLoader classLoader = findClassLoader();
        org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }
```



#### Dubbo加载扩展点篇

> Dubbo获取扩展点，首先去查询下有没有自定义的@Adaptive适配器扩展点，如果没有，才会去生成一个自适应的扩展点实现

```java
    // 记载扩展点的实现类
    private Map<String, Class<?>> getExtensionClasses() {
        //com.alibaba.dubbo.rp.Protocol => [DubboProtocol,MemecachedProtocol,InjvmProtocol.]
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    //加载扩展类
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```

从Dubbo SPI规范对应的三个目录中加载对应的类	

```java
// synchronized in getExtensionClasses
// 从路径中查询对应SPI实现类,Dubbo的SPI实现指定了3个目录：
// META-INF/services/ , META-INF/dubbo/, META-INF/dubbo/internal/
private Map<String, Class<?>> loadExtensionClasses() {
		cacheDefaultExtensionName();
		Map<String, Class<?>> extensionClasses = new HashMap<>();
		//cachedDefaultName="dubbo"(SPI注解上的value值), type.getName = Protocol;
		loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
		loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
		loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
		loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
		loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
		loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
		return extensionClasses;
}
```

cacheDefaultExtensionName方法：

```java
    private void cacheDefaultExtensionName() {
        //当前类是否标有@SPI注解
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if ((value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("More than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if (names.length == 1) {
                    //获取注解中的值,例如@SPI("dubbo"),则cachedDefaultName="dubbo";
                    cachedDefaultName = names[0];
                }
            }
        }
    }
```

真正去加载的loadDirectory:

```java
    /**
     * 从对应的目录中找出SPI指定name对应的实现
     * type =Protocol , 因为文件名字，必须是以protocol结尾的所以，目录就相当于拼接上了type的值
     * @param extensionClasses
     * @param dir
     * @param type
     */
    private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type) {
        String fileName = dir + type;
        try {
            Enumeration<java.net.URL> urls;
            ClassLoader classLoader = findClassLoader();
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL resourceURL = urls.nextElement();
                    loadResource(extensionClasses, classLoader, resourceURL);
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
```

以上loadResource实现部分代码如下：

```java
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, 			URL resourceURL) {
  	//对于某个SPI接口，对应有多个实现
  	/**
       * interface = file.name
       * >>>
       * aaa=com.....aaa.impl
       * bbb=com.....bbb.impl
       * ccc=com.....ccc.impl
      */
      String line;
      while ((line = reader.readLine()) != null) {
     			loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name);
      }
}
```

loadClass的实现如下：

```java
    private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        //判断clazz是否实现了type , type = Protocol.class , clazz = DubboProtocol.class
        if (!type.isAssignableFrom(clazz)) {
            throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                    type + ", class line: " + clazz.getName() + "), class "
                    + clazz.getName() + " is not subtype of interface.");
        }
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            //如果类（扩展实现类）上标注了@Adaptive注解，则缓存到cachedAdaptiveClass;
            cacheAdaptiveClass(clazz);
        } else if (isWrapperClass(clazz)) {
            //如果是包装类，则缓存到cachedWrapperClasses集合中,准备做成包装类
          	// TODO？ 为什么要包装？
            cacheWrapperClass(clazz);
        } else {
            clazz.getConstructor();
            if (StringUtils.isEmpty(name)) {
                name = findAnnotationName(clazz);
                if (name.length() == 0) {
                    throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
                }
            }

            String[] names = NAME_SEPARATOR.split(name);
            if (ArrayUtils.isNotEmpty(names)) {
                cacheActivateClass(clazz, names[0]);
                for (String n : names) {
                    cacheName(clazz, n);
                    saveInExtensionClass(extensionClasses, clazz, name);
                }
            }
        }
    }
```

Adaptive:

- 如果是配置在类级别上，表示自定义适配器；
- 如果是配置在方法级别上，表示需要动态生成适配器类；

```java
    /**
     * test if clazz is a wrapper class
     * <p>
     * which has Constructor with given class type as its only argument
     * 是否是包装类 ： 某个SPI的实现类中的构造函数仅有一个参数，并且该参数是SPI扩展接口
     * 假如现在是要获取Protocol的扩展实现，如果在SPI中获取的对应的扩展实现类的构造函数带有Protocol的参数，则
     * 该SPI实现类则被称为是包装类.
     * public DubboProtocol(Protocol protocol){}
     */
    private boolean isWrapperClass(Class<?> clazz) {
        try {
            clazz.getConstructor(type);
            return true;
        } catch (NoSuchMethodException e) {
            return false;
        }
    }
```

如何进行包装？

以



#### 小结

我们的入口是：

```java
// 获取协议的自适应扩展
ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()
```

而真正调用调用的时候，先加载的Protocol的扩展，如果该扩展类上有@Adaptive注解，则直接使用这个扩展类，如果没有的话，则去生成一个，那如何生成自适应的扩展？看生成的字节码，Protocol$Adaptive类，这个类内部就会根据Url来判断是要用hessian协议，还是用dubbo的协议(默认使用dubbo),原来自适应指的是这个地方；

所以最后，getAdaptiveExtension()得到了什么？

- 自适应适配器(Protocol$Adaptive)；

- 自定义适配器；

  

#### Extension 依赖注入

> 加载出来的Extension是有属性的，这些数据需要被注入。

以AdaptiveCompiler为例：

```java
/**
 * AdaptiveCompiler. (SPI, Singleton, ThreadSafe)
 */
@Adaptive
public class AdaptiveCompiler implements Compiler {

    private static volatile String DEFAULT_COMPILER;

    public static void setDefaultCompiler(String compiler) {
        DEFAULT_COMPILER = compiler;
    }

    @Override
    public Class<?> compile(String code, ClassLoader classLoader) {
        Compiler compiler;
        ExtensionLoader<Compiler> loader = ExtensionLoader.getExtensionLoader(Compiler.class);
        String name = DEFAULT_COMPILER; // copy reference
        if (name != null && name.length() > 0) {
            compiler = loader.getExtension(name);
        } else {
            compiler = loader.getDefaultExtension();
        }
        return compiler.compile(code, classLoader);
    }

}
```

它也是一个扩展点，对应的Dubbo-SPI配置文件：

```java
adaptive=org.apache.dubbo.common.compiler.support.AdaptiveCompiler
jdk=org.apache.dubbo.common.compiler.support.JdkCompiler
javassist=org.apache.dubbo.common.compiler.support.JavassistCompiler
```

具体依赖注入代码如下：

```java
private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                  	// (一)
                    if (isSetter(method)) {
                        if (method.getAnnotation(DisableInject.class) != null) {
                            continue;
                        }
                        Class<?> pt = method.getParameterTypes()[0];
                        if (ReflectUtils.isPrimitives(pt)) {
                            continue;
                        }
                        try {
                          	// (二)
                            String property = getSetterProperty(method);
                          	// (三)
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("Failed to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```

- 判断类中是否定义符合要求的方法：

```java
private boolean isSetter(Method method) {
  	// 方法名为set
  	return method.getName().startsWith("set")
      	// 参数类型只有一个
    		&& method.getParameterTypes().length == 1
    		&& Modifier.isPublic(method.getModifiers());
}
```

- objectFactory 为最初设置的ExtensionLoader的扩展点-AdaptiveExtensionFactory

```java
objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
```

因为ExtensionFactory的其中一个实现-AdaptiveExtensionFactory的源码：

```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
      	// 初始化Factory的扩展点加载器
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
      	// (一)获取所有扩展实现
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
          	// (二)
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }

}
```

- 获取所有的扩展点实现，这个getSupportFactoryExtensions()的实现如下：

  ```java
   public Set<String> getSupportedExtensions() {
     	// 加载所有的扩展点实现
     	Map<String, Class<?>> clazzes = getExtensionClasses();
     	return Collections.unmodifiableSet(new TreeSet<>(clazzes.keySet()));
   }
  ```

- 加载**具体**的扩展点:

  ```java
  Object object = objectFactory.getExtension(pt, property);
  ```

  会执行以上getExtension(Class<T> type, String name)方法。



#### 疑问

1. @Activate的使用；
2. 如何自实现一个扩展点？
3. 指定扩展点实现是如何获取的？



























