Optional是Java8中引入的API，主要为了解决NPE（Null Pointer Exception）问题。使对空指针的检查变得更加简单便捷。

## 引言

先看一下不使用Optional API时通常的情形：

```java
String code = user.getAddress().getCountry().getPostCode().toUpperCase();
```

以上一连串的方法调用中，如果要确保不会触发NPE，需要在访问每个值进行方法调用之前，明确的检查对象是否为空：

```java
String code = null;
if(user != null) {
		Address address = user.getAddress();
  	if(address != null) {
      	Country country = address.getCountry();
      	if(country != null) {
          	String postCode = country.getPostCode();
          	if(postCode != null) {
              	code = postCode.toUpperCase();
            }
        }
    }
}
```

上面的代码十分冗长、丑陋，但是不进行检查又会触发异常。可以使用`Optional`类来简化这个过程。



## 使用Optional

#### 

```java
of(T t);// t 为 null 会抛出异常
ofNullable(T t);
isPresent(); // 非null返回true；否则返回false
ifPresent(Consumer<?> con); // 存在执行传入的consumer.accept();
map(Function<? super T, ? extends U> mapper); // 为空返回empty()，否则执行mapper.apply(value)
flatMap(Function<? super T, Optional<U>> mapper);
//与map()不同的是，map以mapper的值映射输出然后包装成Optional，而flatMap直接返回Optional
orElse(T t); // 不存在返回传入的t
orElseGet(Supplier s); // 不存在执行s.get()
orElseThrow(Supplier<? extends X> exceptionSupplier); // 不存在抛出传入的异常处理

```

