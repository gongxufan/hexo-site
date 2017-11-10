---
layout: post
title: "如何正确使用Java8的Optional机制"
date: 2017-10-23 14:54
tags: java
category: 随笔
description: Java8的Optional机制的正确认识和使用方法。

---
Java8带来的函数式编程特性对于习惯命令式编程的程序员来说还是有一定的障碍的，我们只有深入了解这些机制的方方面面才能运用自如。Null的处理在JAVA编程中是出了try catch之外的另一个头疼的问题，需要大量的非空判断模板代码，程序逻辑嵌套层次太深。尤其是对集合的使用，需要层层判空。

首先来看下Optional类的结构图：
![img](/upload/images/java/optional.png)
## 属性
```java
   /** 
    * Common instance for {@code empty()}. 
    */  
   private static final Optional<?> EMPTY = new Optional<>();  
  
   /** 
    * If non-null, the value; if null, indicates no value is present 
    */  
   private final T value;  
```
1) EMPTY持有某个类型的空值结构,调用empty()返回的即是该实例
```java
public static<T> Optional<T> empty() {  
       @SuppressWarnings("unchecked")  
       Optional<T> t = (Optional<T>) EMPTY;  
       return t;  
   }  
```
2) T vaule是该结构的持有的值

## 方法
### 构造函数
```java
private Optional() {  
        this.value = null;  
    }  
private Optional(T value) {  
        this.value = Objects.requireNonNull(value);  
    }  
```
Optional(T value)如果vaule为null就会抛出NullPointer异常,所以对于使用的场景这两个构造器都适用.

### 生成Optional对象
有两个方法 of(T)和ofNullable(T)
```java
public static <T> Optional<T> of(T value) {  
        return new Optional<>(value);  
    }  
  
 public static <T> Optional<T> ofNullable(T value) {  
        return value == null ? empty() : of(value);  
    } 
```
of是直接调用的构造函数,因此如果T为null则会抛出空指针异常,ofNullable对null进行了处理,会返回EMPTY的实例,因此不会出现异常。

所以只有对于明确不会为null的对象才能直接使用of

###  获取Optional对象的值
需要摈弃的使用方式
>if(value.isPresent){
  ....
  }else{
  T t  = value.get();
  }
    
这种使用方式无异于传统的if(vaule != null)
正确的使用姿势:
>orElse:如果值为空则返回指定的值
orElseGet:如果值为空则调用指定的方法返回
orElseThrow:如果值为空则直接抛出异常

```java
public T orElse(T other) {  
    return value != null ? value : other;  
}  
  
public T orElseGet(Supplier<? extends T> other) {  
    return value != null ? value : other.get();  
}  
  
public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {  
    if (value != null) {  
        return value;  
    } else {  
        throw exceptionSupplier.get();  
    }  
}  
```

一般我们使用orElse来取值,如果不存在返回默认值.
### Optional的中间处理
filter,map,flatMap,这几个操作跟Stream的处理类似,只是要注意flatMap处理必须手动指定返回类型为Optional<U>,而map会自动将返回值包装成Optional.举个栗子,我们有商品很订单的结构:
```java
package model;  
  
import java.util.List;  
  
/** 
 * @auth gongxufan 
 * @Date 2017/10/23 
 **/  
public class Goods {  
    private String goodsName;  
    private double price;  
    private List<Order> orderList;  
  
    public String getGoodsName() {  
        return goodsName;  
    }  
  
    public void setGoodsName(String goodsName) {  
        this.goodsName = goodsName;  
    }  
  
    public double getPrice() {  
        return price;  
    }  
  
    public void setPrice(double price) {  
        this.price = price;  
    }  
  
    public List<Order> getOrderList() {  
        return orderList;  
    }  
  
    public void setOrderList(List<Order> orderList) {  
        this.orderList = orderList;  
    }  
}  

package model;  
  
import java.time.LocalDateTime;  
  
/** 
 * @auth gongxufan 
 * @Date 2017/10/23 
 **/  
public class Order {  
    private LocalDateTime createTime;  
    private LocalDateTime finishTime;  
    private String orderName;  
    private String orderUser;  
  
    public LocalDateTime getCreateTime() {  
        return createTime;  
    }  
  
    public void setCreateTime(LocalDateTime createTime) {  
        this.createTime = createTime;  
    }  
  
    public LocalDateTime getFinishTime() {  
        return finishTime;  
    }  
  
    public void setFinishTime(LocalDateTime finishTime) {  
        this.finishTime = finishTime;  
    }  
  
    public String getOrderName() {  
        return orderName;  
    }  
  
    public void setOrderName(String orderName) {  
        this.orderName = orderName;  
    }  
  
    public String getOrderUser() {  
        return orderUser;  
    }  
  
    public void setOrderUser(String orderUser) {  
        this.orderUser = orderUser;  
    }  
}  
```
现在我有一个goodsOptional
```java
Optional<Goods> goodsOptional = Optional.ofNullable(new Goods());  
```
现在我需要获取goodsOptional里边的orderList,应该这样你操作
```java
goodsOptional.flatMap(g ->Optional.ofNullable(g.getOrderList())).orElse(Collections.emptyList())  
```
flatMap里头返回的是Optional<List<Order>>,然后我们再使用orElse进行unwraap.因此faltMap可以解引用更深层次的的对象链.
### 检测Optional并执行动作
```java
public void ifPresent(Consumer<? super T> consumer) {  
        if (value != null)  
            consumer.accept(value);  
    }  

```
这是一个终端操作,不像上边的可以进行链式操作.在Optional实例使用直接调用,如果value存在则会调用指定的消费方法.举个栗子:
```java
Goods goods = new Goods();  
Optional<Goods> goodsOptional = Optional.ofNullable(goods);  
List<Order> orderList = new ArrayList<>();  
goods.setOrderList(orderList);  
goodsOptional.flatMap(g ->Optional.ofNullable(g.getOrderList())).ifPresent((v)-> System.out.println(v));  
```

## end
至此该类的方法和使用介绍都差不多了,最后总结需要注意的地方:
1) Optional应该只用处理返回值,而不应该作为类的字段或者方法的参数.因为这样会造成额外的复杂度.
2) 使用Option应该避免直接适应构造器和get,而应该使用isElse的系列方法避免频繁的非空判断
3) map和flatMap要注意区分使用场景
