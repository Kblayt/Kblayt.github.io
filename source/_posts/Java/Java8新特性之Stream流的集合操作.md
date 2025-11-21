---
title: Java8新特性之Stream流的集合操作
date: 2022-10-21 09:46:33
categories:
    - Java
tags:
    - Java8新特性
    - Stream流代替for循环
---

# 概JDK8 Stream详解

## 概念

>
Stream是Java8 API的新成员，它允许以声明性方式处理数据集合 。


## 特点

>
（1）代码简洁：函数式编程写出的代码简洁且意图明确，使用stream接口让你从此告别for循环。
（2）多核友好：Java函数式编程使得编写并行程序从未如此简单，你需要的全部就是调用一下方法。


## 流程

```
1）第一步：把集合转换为流stream
2）第二步：操作stream流
stream流在管道中经过中间操作（intermediate operation）的处理，最后由最终操作(terminal operation)得到前面处理的结果

```

## 操作符

>
两种：中间操作符、终止操作符


### 中间操作符

| 流方法 | 含义 | 示例 |
| - | - | - |
| filter | 用于通过设置的条件过滤出元素 | List&lt;String&gt; strings = Arrays.asList("abc", "", "bc", "efg", "abcd","", "jkl");List&lt;String&gt; filtered = strings.stream().filter(string -&gt; !string.isEmpty()).collect(Collectors.toList()); |
| distinct | 返回一个元素各异（根据流所生成元素的hashCode和equals方法实现）的流。 | List&lt;Integer&gt; numbers = Arrays.asList(1, 2, 1, 3, 3, 2, 4);numbers.stream().filter(i -&gt; i % 2 == 0).distinct().forEach(System.out::println); |
| limit | 会返回一个不超过给定长度的流。 | List&lt;String&gt; strings = Arrays.asList("abc", "abc", "bc", "efg", "abcd","jkl", "jkl");List&lt;String&gt; limited = strings.stream().limit(3).collect(Collectors.toList()); |
| skip | 返回一个扔掉了前n个元素的流。 | List&lt;String&gt; strings = Arrays.asList("abc", "abc", "bc", "efg", "abcd","jkl", "jkl");List&lt;String&gt; skiped = strings.stream().skip(3).collect(Collectors.toList()); |
| map | 接受一个函数作为参数。这个函数会被应用到每个元素上，并将其映射成一个新的元素（使用映射一词，是因为它和转换类似，但其中的细微差别在于它是&quot;创建一个新版本”而不是去&quot;修改”）。 | List&lt;String&gt; strings = Arrays.asList("abc", "abc", "bc", "efg", "abcd","jkl", "jkl");List&lt;String&gt; mapped = strings.stream().map(str-&gt;str+"-itcast").collect(Collectors.toList()); |
| flatMap | 使用flatMap方法的效果是，各个数组并不是分别映射成一个流，而是映射成流的内容。所有使用map(Arrays::stream)时生成的单个流都被合并起来，即扁平化为一个流。 | List&lt;String&gt; strings = Arrays.asList("abc", "abc", "bc", "efg", "abcd","jkl", "jkl");Stream&lt;Character&gt; flatMap = strings.stream().flatMap(Java8StreamTest::getCharacterByString); |
| sorted | 返回排序后的流 | List&lt;String&gt; strings1 = Arrays.asList("abc", "abd", "aba", "efg", "abcd","jkl", "jkl");List&lt;String&gt; sorted1 = strings1.stream().sorted().collect(Collectors.toList()); |


>
示例代码：
1）filter
```angular2html
/**
* 功能描述:根据条件过滤集合数据
* @return : void
*/
  @Test
  public void filter(){
  List<String> strings = Arrays.asList("abc", "", "bc", "efg", "abcd","", "jkl");
  List<String> filtered = strings.stream().filter(string -> !string.isEmpty()).collect(Collectors.toList());
  out.println(filtered);
  }
```
2）distinct
```angular2html
/**
* 功能描述:去除集合中重复数据
* @return : void
*/
  @Test
  public void distinct(){
  List<String> strings = Arrays.asList("abc", "abc", "bc", "efg", "abcd","jkl", "jkl");
  List<String> distincted = strings.stream().distinct().collect(Collectors.toList());
  out.println(distincted);
  }
```


3）limit
```angular2html
/**
* 功能描述:指定获取集合前x条数据，重新构造一个新的集合
* @return : void
*/
  @Test
  public void limit(){
  List<String> strings = Arrays.asList("abc", "abc", "bc", "efg", "abcd","jkl", "jkl");
  List<String> limited = strings.stream().limit(3).collect(Collectors.toList());
  out.println(limited);
  }
```


4）skip
```angular2html
/**
* 功能描述:排除集合前x条数据，把后面的数据重新构造一个新的集合
* @return : void
*/
  @Test
  public void skip(){
  List<String> strings = Arrays.asList("abc", "abc", "bc", "efg", "abcd","jkl", "jkl");
  List<String> skiped = strings.stream().skip(3).collect(Collectors.toList());
  out.println(skiped);
  }
```


5）map
```angular2html
    /**
    * 功能描述:对集合中所有元素统一处理
* @return : void
  */
  @Test
  public void map(){
  List<String> strings = Arrays.asList("abc", "abc", "bc", "efg", "abcd","jkl", "jkl");
  List<String> mapped = strings.stream().map(str->str+"-itcast").collect(Collectors.toList());
  out.println(mapped);
  }
```


6）flatMap
```angular2html
/**
* 功能描述:对集合中所有元素统一处理
* @return : void
*/
  @Test
  public void flatMap(){
  List<String> strings = Arrays.asList("abc", "abc", "bc", "efg", "abcd","jkl", "jkl");
  Stream<String> stringStream = strings.stream().map(x -> x);
  Stream<String> stringStream1 = strings.stream().flatMap(x -> Arrays.asList(x.split(" ")).stream());
  }
```


7）sorted
```angular2html
/**
* 功能描述 : 对集合进行排序
* @return : void
*/
  @Test
  public void sorted(){
  List<String> strings1 = Arrays.asList("abc", "abd", "aba", "efg", "abcd","jkl", "jkl");
  List<String> strings2 = Arrays.asList("张三", "李四", "王五", "赵柳", "张哥","李哥", "王哥");
  List<Integer> strings3 = Arrays.asList(10, 2, 30, 22, 1,0, -9);
  List<String> sorted1 = strings1.stream().sorted().collect(Collectors.toList());
  List<String> sorted2 = strings2.stream().sorted(Collections.reverseOrder(Collator.getInstance(Locale.CHINA))).collect(Collectors.toList());
  List<Integer> sorted3 = strings3.stream().sorted().collect(Collectors.toList());
  out.println(sorted1);
  out.println(sorted2);
  out.println(sorted3);
  }
```


Map、flatMap区别


```
map：对流中每一个元素进行处理
flatMap：流扁平化，让你把一个流中的“每个值”都换成另一个流，然后把所有的流连接起来成为一个流 
总结：map是对一级元素进行操作，flatmap是对二级元素操作。

```

本质区别：map返回一个值；flatmap返回一个流，多个值。

应用场景：map对集合中每个元素加工,返回加工后结果；flatmap对集合中每个元素加工后，做扁平化处理后（拆分层级，放到同一层）然后返回

```

/**
 * 方法一
 * 功能描述:  通过使用map、flatMap把字符串转换为字符输出对比区别
 * @return : void
 */
@Test
public void flatMap2Map(){
    List<String> strings = Arrays.asList("abc", "abc", "bc", "efg", "abcd","jkl", "jkl");
    final Stream<Character> flatMap = strings.stream().flatMap(Java8StreamTest::getCharacterByString);
    flatMap.forEach(System.out::println);
    //----------------------------------------------
    final Stream<Stream<Character>> mapStream = strings.stream().map(Java8StreamTest::getCharacterByString);
    //mapStream.forEach(System.out::println);
    out.println("------------------------------------------------");
    mapStream.forEach(stream-> {stream.forEach(character->{System.out.println(character);});});

}

```

>
公共方法（字符串转换为字符流）


```
/**
* 功能描述:字符串转换为字符流
* @param str
* @return : java.util.stream.Stream<java.lang.Character>
*/
public static Stream<Character> getCharacterByString(String str) {
    List<Character> characterList = new ArrayList<>();
    for (Character character : str.toCharArray()) {
    	characterList.add(character);
    }
    return characterList.stream();
}

```

### 终止操作符

| 流方法 | 含义 | 示例 |
| - | - | - |
| anyMatch | 检查是否至少匹配一个元素，返回boolean。 | List&lt;String&gt; strings = Arrays.asList("abc", "abd", "aba", "efg", "abcd","jkl", "jkl");boolean b = strings.stream().anyMatch(s -&gt; s == "abc"); |
| allMatch | 检查是否匹配所有元素，返回boolean。 | List&lt;String&gt; strings = Arrays.asList("abc", "abd", "aba", "efg", "abcd","jkl", "jkl");boolean b = strings.stream().allMatch(s -&gt; s == "abc"); |
| noneMatch | 检查是否没有匹配所有元素，返回boolean。 | List&lt;String&gt; strings = Arrays.asList("abc", "abd", "aba", "efg", "abcd","jkl", "jkl");boolean b = strings.stream().noneMatch(s -&gt; s == "abc"); |
| findAny | 将返回当前流中的任意元素。 | List&lt;String&gt; strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");Optional&lt;String&gt; any = strings.stream().findAny(); |
| findFirst | 返回第一个元素 | List&lt;String&gt; strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");Optional&lt;String&gt; first = strings.stream().findFirst(); |
| forEach | 遍历流 | List&lt;String&gt; strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");strings.stream().forEach(s -&gt; out.println(s)); |
| collect | 收集器，将流转换为其他形式。 | List&lt;String&gt; strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");Set&lt;String&gt; set = strings.stream().collect(Collectors.toSet());List&lt;String&gt; list = strings.stream().collect(Collectors.toList());Map&lt;String, String&gt; map = strings.stream().collect(Collectors.toMap(v -&gt;v.concat("\_name"), v1 -&gt; v1, (v1, v2) -&gt; v1)); |
| reduce | 可以将流中元素反复结合起来，得到一个值。使用 reduce((T, T) -&gt; T) 和 reduce(T, (T, T) -&gt; T) 用于组合流中的元素，如求和，求积，求最大值等。 | List&lt;String&gt; strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");Optional&lt;String&gt; reduce = strings.stream().reduce((acc,item) -&gt; {return acc+item;});if(reduce.isPresent())out.println(reduce.get()); |
| count | 返回流中元素总数。 | List&lt;String&gt; strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");long count = strings.stream().count(); |


>
示例代码
1）anyMatch
```angular2html
/**
* 功能描述 : 判断集合中是否至少存在一个元素满足条件
* @return : void
*/
  @Test
  public void anyMatch(){
  List<String> strings = Arrays.asList("abc", "abd", "aba", "efg", "abcd","jkl", "jkl");
  boolean b = strings.stream().anyMatch(s -> s == "abc");
  out.println(b);
  }
```

2）allMatch
```angular2html
/**
* 功能描述 : 判断集合中是否所有元素都满足条件
* @return : void
*/
  @Test
  public void allMatch(){
  List<String> strings = Arrays.asList("abc", "abd", "aba", "efg", "abcd","jkl", "jkl");
  boolean b = strings.stream().allMatch(s -> s == "abc");
  out.println(b);
  }
```

3）noneMatch
```angular2html
/**
* 功能描述 : 判断集合中是否所有元素都不满足条件
* @return : void
*/
  @Test
  public void noneMatch(){
  List<String> strings = Arrays.asList("abc", "abd", "aba", "efg", "abcd","jkl", "jkl");
  boolean b = strings.stream().noneMatch(s -> s == "abc");
  out.println(b);
  }
```


4）findAny
```angular2html
/**
* 功能描述 : 返回当前流中任意元素
* @return : void
*/
  @Test
  public void findAny(){
  List<String> strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");
  Optional<String> any = strings.stream().findAny();
  if(any.isPresent()) out.println(any.get());
  }
```

5）findFirst
```angular2html
/**
* 功能描述 : 返回当前流中第一个元素
* @return : void
*/
  @Test
  public void findFirst(){
  List<String> strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");
  Optional<String> first = strings.stream().findFirst();
  if(first.isPresent()) out.println(first.get());
  }
```


6）forEach java
```angular2html
/**
* 功能描述 : 遍历流
* @return : void
*/
  @Test
  public void foreach(){
  List<String> strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");
  strings.stream().forEach(s -> out.println(s));
  }
```


7）collect
```angular2html
/**
* 功能描述 : 流转换为其他形式
* @return : void
*/
  @Test
  public void collect(){
  List<String> strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");
  Set<String> set = strings.stream().collect(Collectors.toSet());
  List<String> list = strings.stream().collect(Collectors.toList());
  Map<String, String> map = strings.stream().collect(Collectors.toMap(v ->v.concat("_name"), v1 -> v1, (v1, v2) -> v1));
  out.println(set);
  out.println(list);
  out.println(map);
  }
```

8）reduce
```angular2html
/**
* 功能描述 : 将流中元素反复结合起来，得到一个值
* @return : void
*/
  @Test
  public void reduce(){
  List<String> strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");
  //reduce方法一
  Optional<String> reduce1 = strings.stream().reduce((acc,item) -> {return acc+item;});
  //reduce方法二
  String reduce2 = strings.stream().reduce("itcast", (acc, item) -> {
  return acc + item;
  });
  //reduce方法三
  ArrayList<String> reduce3 = strings.stream().reduce(
  new ArrayList<String>(),
  new BiFunction<ArrayList<String>, String, ArrayList<String>>() {
  @Override
  public ArrayList<String> apply(ArrayList<String> acc, String item) {
  acc.add(item);
  return acc;
  }
  },
  new BinaryOperator<ArrayList<String>>() {
  @Override
  public ArrayList<String> apply(ArrayList<String> acc, ArrayList<String> item) {
  return acc;
  }
  }
  );
  if(reduce1.isPresent())out.println(reduce1.get());
  out.println(reduce2);
  out.println(reduce3);
  }
```

9）count
```
/**
* 功能描述 : 返回流中元素总数
* @return : void
*/
  @Test
  public void count(){
  List<String> strings = Arrays.asList("cv", "abd", "aba", "efg", "abcd","jkl", "jkl");
  long count = strings.stream().count();
  out.println(count);
  }
```

注意：文章中因排序部分用到外部比较器，需要导入外部jar包

```xml
<!--apache集合操作工具包-->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.4</version>
</dependency>

```


# https://www.cnblogs.com/shoshana-kong/p/14406683.html
