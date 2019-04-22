## ThradLocal的实现原理

# ThreadLocal的简单使用
```java
public class TestThreadLocal {
    static ThreadLocal<String> threadLocal = new ThreadLocal<>();

    public static void setValue(String value){
        threadLocal.set(value);
    }
    public static String getValue(){
        return threadLocal.get();
    }

    public static void main(String[] args) {
        new Thread(()->{
            TestThreadLocal.setValue("1");
            System.out.println(Thread.currentThread().getName()+TestThreadLocal.getValue());
        },"Thread-A").start();
        new Thread(()->{
            TestThreadLocal.setValue("2");
            System.out.println(Thread.currentThread().getName()+TestThreadLocal.getValue());
        },"Thread-B").start();
    }
}
```
输出：
```java
Thread-A1
Thread-B2
```

## 源码分析

首先来看set操作:
```java
    public void set(T value) {
		//获取当前线程
        Thread t = Thread.currentThread();
		//取出该线程中的ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null)
			// 如果该线程已经生成threadLocals，则直接设置对应的值,Map<ThreadLocal, value>
            map.set(this, value);
        else
			// 如果该线程还未生成threadLocals，则创建一个ThreadLocalMap,并设置对应的数据
            createMap(t, value);
    }
```
getMap(Thread)源码如下：
```java
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
t.threadLocals定义如下：
```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```
创建ThreadLocalMap如下：
```java
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
而ThreadLocalMap底层就是一个Map结构：
```java
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
			// 底层维护数组
            table = new Entry[INITIAL_CAPACITY];
			// 通过取模算法找到索引位置
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
			// 生成key-value，设置到对应的索引位置中
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```
ThreadLocalMap中有一个比较棒的设计(如下为其定义的部分代码):
```java
static class ThreadLocalMap {
	// map中的entry定义为弱引用类型
	static class Entry extends WeakReference<ThreadLocal<?>> {
		Object value;
		Entry(ThreadLocal<?> k, Object v) {
			super(k);
			value = v;
		}
	}
	...
}
```
