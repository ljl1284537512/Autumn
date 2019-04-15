>  Mybatis 问题- Illegal overloaded getter method with ambiguous type for property tempPlateNo in class class com.souche.lease.model.domain.OrderCarDO. This breaks the JavaBeans specification and can cause unpredictable results.at org.apache.ibatis.reflection.Reflector.resolveGetterConflicts(Reflector.java:138)



Mybatis 相关代码：

```java
  private void resolveGetterConflicts(Map<String, List<Method>> conflictingGetters) {
    //双重for循环判断
    for (Entry<String, List<Method>> entry : conflictingGetters.entrySet()) {
      Method winner = null;
      String propName = entry.getKey();
      for (Method candidate : entry.getValue()) {
        if (winner == null) {
          //标记name对应的第一个method为最终选中的方法
          winner = candidate;
          continue;
        }
        Class<?> winnerType = winner.getReturnType();
        //candidate代表同一个属性的另一个方法(getter/is)对应的返回值类型
        Class<?> candidateType = candidate.getReturnType();
        //如果两个方法的返回值是一样的
        if (candidateType.equals(winnerType)) {
          //不允许两个方法的返回值是一样并且不为boolean
          if (!boolean.class.equals(candidateType)) {
            throw new ReflectionException(
                "Illegal overloaded getter method with ambiguous type for property "
                    + propName + " in class " + winner.getDeclaringClass()
                    + ". This breaks the JavaBeans specification and can cause unpredictable results.");
          } else if (candidate.getName().startsWith("is")) {
            //如果返回值是boolean，则优先选择isXXX方法
            winner = candidate;
          }
        //判定此 Class 对象所表示的类或接口与指定的 Class 参数所表示的类或接口是否相同，或是否是其超类或超接口
        } else if (candidateType.isAssignableFrom(winnerType)) {
          // OK getter type is descendant
        } else if (winnerType.isAssignableFrom(candidateType)) {
          winner = candidate;
        } else {
          //否则报错
          throw new ReflectionException(
              "Illegal overloaded getter method with ambiguous type for property "
                  + propName + " in class " + winner.getDeclaringClass()
                  + ". This breaks the JavaBeans specification and can cause unpredictable results.");
        }
      }
      addGetMethod(propName, winner);
    }
  }
```

 这个resolveGetterConflicts调用的地方：

```java
  private void addGetMethods(Class<?> cls) {
    Map<String, List<Method>> conflictingGetters = new HashMap<String, List<Method>>();
    Method[] methods = getClassMethods(cls);
    for (Method method : methods) {
      if (method.getParameterTypes().length > 0) {
        continue;
      }
      String name = method.getName();
      //只要是get或者是is开头的方法，都需要装进这个conflictingGetters对应的Map集合中
      if ((name.startsWith("get") && name.length() > 3)
          || (name.startsWith("is") && name.length() > 2)) {
        //根据方法名提取出属性名称
        //1. get/is截取方法名；2. 首字母小写；
        name = PropertyNamer.methodToProperty(name);
        //最终要的conflictingGetters的数据是：
        //conflictingGetters = {
        //	name=[method1,method2] ,
        // 	name2 = [method4,method5]
      	//}
        // method1和method2方法有可能是一个isXXX,一个getXXX的格式，如果只有一个的话，那么name对应的数组只有一个元素
        addMethodConflict(conflictingGetters, name, method);
      }
    }
    //检查方法名，看其是否发生冲突
    resolveGetterConflicts(conflictingGetters);
  }
```

再回过头看这个resolveGetterConflicts方法的内部实现，就非常清晰了。

解决办法：不用让isXXX和getterXXX对同一个属性同时存在。