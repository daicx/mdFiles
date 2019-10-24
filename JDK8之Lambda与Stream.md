---
title: JDK 1.8新特性之Lambda和Stream
date: 2019-03-04 23:13:03
tags:
- Lambda
- Stream
categories:
- java基础
- JDK1.8
---

# JDK 1.8新特性之Lambda和Stream

## 1.Lambda表达式

### 1.1 lambda简介

lambda是为了简化代码,代替一些匿名类而推出的.

### 1.2公式

```
(Type1 param1, Type2 param2, ..., TypeN paramN) -> {
  statment1;
  statment2;
  //.............
  return statmentM;
}
```

### 1.3 用例

#### 1 创建线程

```
// Java 8之前：
new Thread(new Runnable() {
    @Override
    public void run() {
    System.out.println("Before Java8, too much code for too little to do");
    }
}).start();

//Java 8方式：
new Thread( () -> System.out.println("In Java8, Lambda expression rocks !!") ).start();
```

#### 2 列表的循环

```
// Java 8之前：
List features = Arrays.asList("Lambdas", "Default Method", "Stream API", "Date and Time API");
for (String feature : features) {
    System.out.println(feature);
}

// Java 8之后：
List features = Arrays.asList("Lambdas", "Default Method", "Stream API", "Date and Time API");
features.forEach(n -> System.out.println(n));
 
// 使用Java 8的方法引用更方便，方法引用由::双冒号操作符标示，
// 看起来像C++的作用域解析运算符
features.forEach(System.out::println);
```

#### 3 结合函数式接口Predicate

```
public static void main(args[]){
    List languages = Arrays.asList("Java", "Scala", "C++", "Haskell", "Lisp");
 
    System.out.println("Languages which starts with J :");
    filter(languages, (str)->str.startsWith("J"));
 
    System.out.println("Languages which ends with a ");
    filter(languages, (str)->str.endsWith("a"));
 
    System.out.println("Print all languages :");
    filter(languages, (str)->true);
 
    System.out.println("Print no language : ");
    filter(languages, (str)->false);
 
    System.out.println("Print language whose length greater than 4:");
    filter(languages, (str)->str.length() > 4);
}
 
public static void filter(List names, Predicate condition) {
    for(String name: names)  {
        if(condition.test(name)) {
            System.out.println(name + " ");
        }
    }
}
```

#### 4.Lambda中this的概念

Lambda中的this指的是声明它的外部对象,不是闭包里面的.

```
public class WhatThis {

     public void whatThis(){
           //转全小写
           List<String> proStrs = Arrays.asList(new String[]{"Ni","Hao","Lambda"});
           List<String> execStrs = proStrs.stream().map(str -> {
                 System.out.println(this.getClass().getName());
                 return str.toLowerCase();
           }).collect(Collectors.toList());
           execStrs.forEach(System.out::println);
     }

     public static void main(String[] args) {
           WhatThis wt = new WhatThis();
           wt.whatThis();
     }
}

输出：

com.wzg.test.WhatThis
com.wzg.test.WhatThis
com.wzg.test.WhatThis
ni
hao
lambda
```



### 1.4 性能对比

#### 1. 遍历66万个对象对比.

```
ArrayList<user> objects = new ArrayList<>();
        for(int i1=0;i1<666666;i1++){
            objects.add(new user("dcx","12"));
        }
        System.out.println("对象大小是:"+objects.size());
        long start = System.currentTimeMillis();
        objects.forEach (n-> {
            System.out.println(n.name);
        });
        long end = System.currentTimeMillis();
        long start1 = System.currentTimeMillis();
        for (user u:objects){
            System.out.println(u.name);
        }
        long end1 = System.currentTimeMillis();
        System.out.println("lambda耗时="+ (end-start));
        System.out.println("普通方法耗时="+ (end1-start1));
```

结果:

```
lambda耗时=2978
普通方法耗时=2971
```

### 1.5 函数式接口

为了支持Lambda表达式,推出了函数式接口,注解是@FunctionInterface,其本质是一个接口.个人理解是,先定义一个数学中的公式(y=x+b),然后,在使用的时候,直接通过接口使用,而不用去实现这个接口.

#### 1.特点

a.只能有一个抽象方法.

b.可以定义静态方法.

c.可以定义一个或者多个默认方法.

#### 2.demo

//目前研究较浅,具体使用例子还无法定位.

```
@FunctionalInterface
public interface AvageUtil<Y,X,A> {
    Y avage(X x,A a);
    //求平均值
    default int avager(int x,int a){
        return (a+x)/2;
    }
    static void demo(int a){
        System.out.println(a);
    }
    public static void main(String[] args) {
        AvageUtil avageUtil=(x,a)->(int) x+ (int) a;
        //结果是8
        System.out.println(avageUtil.avage(3,5));
        //结果是4
        System.out.println(avageUtil.avager(3,5));
        //结果是3
        AvageUtil.demo(3);
    }
}

```

#### 3.jdk中的函数式接口

1.对于单个入参

```
public class FunctionalInterfaceMain {
 
	public static void main(String[] args) {
		//接收一个T类型的参数，返回一个R类型的结果
		Function<String,String> function1 = item -> item +"返回值";
		// 接收一个T类型的参数，不返回值
		Consumer<String> function2 = iterm -> {System.out.println(iterm);};//lambda语句，使用大括号，没有return关键字，表示没有返回值
		// 接收一个T类型的参数，返回一个boolean类型的结果
		Predicate<String> function3 = iterm -> "".equals(iterm);
		//不接受参数，返回一个T类型的结果
		Supplier<String> function4 = () -> new String("");
		
		/**
		 * 再看看怎么使用
		 * demo释义：
		 * 1、创建一个String类型的集合
		 * 2、将集合中的所有元素的末尾追加字符串'1'
		 * 3、选出长度大于2的字符串
		 * 4、遍历输出所有元素
		 */
		List<String> list = Arrays.asList("zhangsan","lisi","wangwu","xiaoming","zhaoliu");
		
		list.stream()
			.map(value -> value + "1") //传入的是一个Function函数式接口
			.filter(value -> value.length() > 2) //传入的是一个Predicate函数式接口
			.forEach(value -> System.out.println(value)); //传入的是一个Consumer函数式接口
	}
	
}
```

2.对于多个入参

```
public class FunctionalInterfaceTest {
 
	public static void main(String[] args) {
		
		 /**
		  * Bi类型的接口创建
		  */
		  //接收T类型和U类型的两个参数，返回一个R类型的结果
		 BiFunction<String, String, Integer> biFunction = (str1,str2) -> str1.length()+str2.length();
		 //接收T类型和U类型的两个参数，不返回值
		 BiConsumer<String, String> biConsumer = (str1,str2) -> System.out.println(str1+str2);
		 //接收T类型和U类型的两个参数，返回一个boolean类型的结果
		 BiPredicate<String, String> biPredicate = (str1,str2) -> str1.length() > str2.length();
		 
		 
		 /**
		  * Bi类型的接口使用
		  */
		 int length = getLength("hello", "world", (str1,str2) -> str1.length() + str2.length()); //输出10
		 boolean boolean1 = getBoolean("hello", "world", (str1,str2) -> str1.length() > str2.length()); //输出false
		 
		 System.out.println(length);
		 System.out.println(boolean1);
		 
		 noResult("hello", "world", (str1,str2) -> System.out.println(str1+" "+str2)); //没有输出
 
		 
	}
	
	public  static int getLength(String str1,String str2,BiFunction<String, String, Integer> function){
		return function.apply(str1, str2);
	}
	
	public static void noResult(String str1,String str2,BiConsumer<String, String> biConcumer){
		biConcumer.accept(str1, str2);
	}
	
	public static boolean getBoolean(String str1,String str2,BiPredicate<String, String> biPredicate){
		return biPredicate.test(str1, str2);
	}
}
```



## 2.stream API

### 1.stream是什么?

stream是元素的集合,可以类比于iterator,并可以支持顺序或者并行的对原stream进行汇聚的操作.相比于iterator的逐个去操作每个集合的元素,stream支持对所有元素的一口气操作.比如:求平均值,找出大于10的个数等等.

### 2.操作流程示例

//找出大于3的个数

```
List<Integer> nums = Lists.newArrayList(1,2,3,4,5,6);
nums.stream().filter(num -> num >3).count();
```

<iframe frameborder="0" style="width:100%;height:513px;" src="https://www.draw.io/?lightbox=1&highlight=0000ff&layers=1&nav=1&title=stream.drawio#R5VlNc5swEP01OsaDEB%2FyEWzcHtqZzKQzSXpTQAEajFwhx3Z%2FfSUQBiySpglJmmlyCLvSCum91ZOWALRY7z9xssm%2BsoQWwLaSPUBLYNvQsubyj%2FIcGo%2BnLOVIeZ7oTp3jIv9F20jt3eYJrQYdBWOFyDdDZ8zKksZi4COcs92w2y0rhm%2FdkJQajouYFKb3Mk9E1nix7Xf%2BzzRPs%2FbN0NMLXpO2s15JlZGE7XouFAG04IyJ5mm9X9BCgdfi0sStHmg9TozTUjwlAPHLBF1dfkt21z7F1p1%2FVX4%2Fg6gZ5p4UW71iELkgsAD29LTFocWCs22ZUDUcBCjcZbmgFxsSq9adZF%2F6MrEudPNtXhQLVjBexyIKE5f60l8Jzu5or2Xu%2BYh4suWecpFL5IMiT0vZdsOEYGvZoKcom%2Bn%2BwcXDI6QyFylbU8EPsosO8PU6dRY6bXrtOk5dT%2FuyPp%2BWdhKdR%2Blx6A5q%2BaDR%2FgvkXRP4KfG%2BddXvGN5e%2FaMiWCl6%2FuZnGrixNYDbHoHbs8fg9l4LbjgCt1cIDcMAd%2B%2FnlrUNZ1WtSIHsgNBm3zXKp1Q08FKyls0g8sAcAbwEkQPCJZgvlCd0AYYg8kGAAQ7bV8oVNG9txjCIl8CLIbucyomQm7qDJe0Ny0tRY%2BSGwF1KD9kK1ky2DiB6ExX0VozsLcFU%2BlQym%2FIy%2FaaM5ZkzDfe2OyQfOib59thes1%2BLe1PJaCJFXpuMi4ylrCRF1HnDbu8puLs%2BX1gNloL4BxXioE8shf6QMbrPxVXv%2BVoNNXO1tdzrkWvj0BqlXO5V3%2BhFKbMLq602rlmfWtTjnEkM2JbH9BGstAoJwlMqHumHxnOA04KI%2FH44j%2Bk3s2UwOhlfHUfXvZZxvhJSZUd97sibOQqbjsAza2ajP1BYW%2BeU5xIhyt%2BPV%2FxCXnXoudKnThPgfKgJzuleb%2Balo06y4ziN5yeMbah%2FuV1Xs1a8pzx3cUzjeOzcvcGu41rTaCxEJwesZ2osHDtgX%2B189T%2BKxk64p9AT95Tzrlpp3iv7zJSsnJSKv5HPnmIifKqY8DmCORDkkfvtavU%2BGQAf2M5vkwJmjSdFS4H28YQPIe8fEz7HADeWKIoPiK3r%2FmuHCjawhTNVTM0dMF%2Bp8gpjIG8WquCS5VX7EAT6XFcfMlYgnIMIgxCCABqkvF%2Bh9bg4vYjH07uWO0Ijesv6C5rFtz1T7GCvrpc9ECwUO09gtqZyBbBfF9chCBw1juyJozo8ANj5v9lH7smnLoQM%2Bo%2FfY96GflMizStJmQTqa6204oJUVR4PiXnSwd67Txwr5uMt5G3K5zbVn3sn7H%2BPHOGo9b2wHPOHCuHiE%2BqbZRrVmFnW2ScDoanKOml2X8ab7t3%2FF1D0Gw%3D%3D"></iframe>

### 3.创建

1.通过Stream的静态工厂方法.

2.通过collection接口的方法.col.stream().

4.功能

主要针对的是对Collection集合的操作.详情可看Stream源码.

### 4.性能对比

//求平均值,6千万个数据

```
ArrayList<Integer> objects = new ArrayList<>();
        for (int i=0;i<66666666;i++){
            objects.add(i);
        }
        //求平均值
        //传统循环
        long start = System.currentTimeMillis();
        long n=0;
        for (int i=0;i<objects.size();i++){
            n+=objects.get(i);
        }
        System.out.println(n/objects.size());
        System.out.println("传统方式耗时:"+(System.currentTimeMillis()-start));
        //stream API
        long start1 = System.currentTimeMillis();
        IntSummaryStatistics intSummaryStatistics = objects.stream().mapToInt((x) -> x).summaryStatistics();
        System.out.println(intSummaryStatistics.getAverage());
        System.out.println("stream API 耗时:"+(System.currentTimeMillis()-start1));
```

结果:stream耗时会比传统方法长.

```
33333332
传统方式耗时:126
3.33333325E7
stream API 耗时:195
```

