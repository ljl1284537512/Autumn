## SimpleDateFormat 线程安全问题



[现场回顾](#现场回顾)

[源码分析](#源码分析)

[如何解决](#如何解决)

[感谢](#感谢)



#### 现场回顾

代码示例：

```java
public class SimpleDateFormatTest {
    static SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
    public static void main(String[] args) throws InterruptedException {
        for(int t = 0; t < 3 ; t++){
            new Thread(()->{
                try {
                    System.out.println(simpleDateFormat.parse("2017-12-13 15:17:27"));
                } catch (ParseException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

输出结果：

```json
Exception in thread "Thread-2" Exception in thread "Thread-0" Exception in thread "Thread-1" java.lang.NumberFormatException: multiple points
	at sun.misc.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1890)
	at sun.misc.FloatingDecimal.parseDouble(FloatingDecimal.java:110)
	at java.lang.Double.parseDouble(Double.java:538)
	at java.text.DigitList.getDouble(DigitList.java:169)
	at java.text.DecimalFormat.parse(DecimalFormat.java:2056)
	at java.text.SimpleDateFormat.subParse(SimpleDateFormat.java:1869)
	at java.text.SimpleDateFormat.parse(SimpleDateFormat.java:1514)
	at java.text.DateFormat.parse(DateFormat.java:364)
	at com.example.concurrency.AtomicTest.lambda$main$0(AtomicTest.java:20)
	at java.lang.Thread.run(Thread.java:748)
...
```



#### 源码分析

SimpleDateFormat.parse方法部分源码如下：

```java
    @Override
    public Date parse(String text, ParsePosition pos)
    {
        ....
		//(一) 构造CalendarBuilder实例
        CalendarBuilder calb = new CalendarBuilder();

        ....
        try {
          	// (二) 使用已有的calendar完成新的calendar
            parsedDate = calb.establish(calendar).getTime();
		...
        }
        ...
        return parsedDate;
    }
```

calendar为DateFormat的内部变量，如果多线程操作SimpleDateFormat，就相当于多个线程共享calendar这个变量：

```java
public abstract class DateFormat extends Format {

    /**
     * The {@link Calendar} instance used for calculating the date-time fields
     * and the instant of time. This field is used for both formatting and
     * parsing.
     *
     * <p>Subclasses should initialize this field to a {@link Calendar}
     * appropriate for the {@link Locale} associated with this
     * <code>DateFormat</code>.
     * @serial
     */
    protected Calendar calendar;
```

而establish的内部实现如下：

```java
    Calendar establish(Calendar cal) {
        boolean weekDate = isSet(WEEK_YEAR)
                            && field[WEEK_YEAR] > field[YEAR];
        //（一）重置cal中的属性值
        cal.clear();
        //（二）设置cal属性值
        for (int stamp = MINIMUM_USER_STAMP; stamp < nextStamp; stamp++) {
            for (int index = 0; index <= maxFieldIndex; index++) {
                if (field[index] == stamp) {
                    cal.set(index, field[MAX_FIELD + index]);
                    break;
                }
            }
        }

        if (weekDate) {
            ...
            cal.setWeekDate(field[MAX_FIELD + WEEK_YEAR], weekOfYear, dayOfWeek);
        }
        //（三）返回cal变量
        return cal;
    }
```

如上所述代码，假设有A, 和B两个线程同时进行parse操作，那么当A完成 （一）和（二）步骤之后，正准备执行（三）返回，这个时候B线程获取到CPU的执行时间片，并且执行完第（一）步骤，CPU继续执行A线程，那么A拿到的cal就是B clear之后的，之后的操作无疑就很多问题了。
如下是clear()方法的实现，其内部操作了多个Calendar的状态数据：
```java
    public final void clear()
    {
        for (int i = 0; i < fields.length; ) {
            stamp[i] = fields[i] = 0; // UNSET == 0
            isSet[i++] = false;
        }
        areAllFieldsSet = areFieldsSet = false;
        isTimeSet = false;
    }
```
以上stamp，isSet等变量都是Calendar全局变量;

#### 如何解决

1. 每次使用都new SimpleDateFormate；

2. 加锁，保证其原子性：

   ```java
   public class SimpleDateFormatTest {
       static SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
       public static void main(String[] args) throws InterruptedException {
           for(int t = 0; t < 3 ; t++){
               new Thread(()->{
                   try {
                       synchronized(simpleDateFormat){
                       	System.out.println(simpleDateFormat.parse("2017-12-13 15:17:27"));
                       }
                   } catch (ParseException e) {
                       e.printStackTrace();
                   }
               }).start();
          }
       }
   }
   ```

   使用同步的话，会导致高并发情况下，系统的性能下降；

3. 将SimpleDateFormat封装到ThreadLocal对象中，这样就能保证每个线程都只能使用自己的SimpleDateFormat，不会产生问题；

   ```java
   public class SimpleDateFormatTest {
       static ThreadLocal<SimpleDateFormat> dateFormat = new ThreadLocal<SimpleDateFormat>(){
           @Override
           protected SimpleDateFormat initialValue() {
               return new SimpleDateFormat("yyyy-MM-dd");
           }
       };
   
       public static void main(String[] args) throws InterruptedException {
           for(int t = 0; t < 300 ; t++){
               new Thread(()->{
                   try {
                       System.out.println(dateFormat.get().parse("2017-12-13 15:17:27"));
                   } catch (ParseException e) {
                       e.printStackTrace();
                   }
               }).start();
           }
       }
   }
   ```



#### 感谢

本篇文章很感谢<http://ifeve.com/notsafesimpledateformat/>这篇文章的指导, 虽然大部分内容都像是抄袭的，但是笔者自己也是对代码做过实验，并且认真看过源码的。
