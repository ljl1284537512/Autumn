## SimpleDateFormat 线程安全问题



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

2️⃣
