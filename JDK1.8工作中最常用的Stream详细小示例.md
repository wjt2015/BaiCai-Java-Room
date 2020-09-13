> 白菜Java自习室 涵盖核心知识

## Java 8 Stream

Java 8 API添加了一个新的抽象称为流**Stream**，可以让你以一种声明的方式处理数据。

Stream 使用一种类似用 SQL 语句从数据库查询数据的直观方式来提供一种对 Java 集合运算和表达的高阶抽象。
Stream API可以极大提高Java程序员的生产力，让程序员写出高效率、干净、简洁的代码。
这种风格将要处理的元素集合看作一种流， 流在管道中传输， 并且可以在管道的节点上进行处理， 比如**筛选， 排序，聚合**等。
元素流在管道中经过中间操作（intermediate operation）的处理，最后由最终操作(terminal operation)得到前面处理的结果。

##  Stream详细示例

#### 1. forEach 循环
```
@Test
public void forEach() {
    // 你不鸟我,我也不鸟你
    List<String> list = Arrays.asList("you", "don't", "bird", "me", ",", 
                                       "I", "don't", "bird", "you");

    // 方式一：JDK1.8之前的循环方式
    for (String item: list) {
        System.out.println(item);
    }

    // 方式二：使用Stream的forEach方法
    // void forEach(Consumer<? super T> action)
    list.stream().forEach(item -> System.out.println(item));

    // 方式三：方式二的简化方式
    // 由于方法引用也属于函数式接口，所以方式二Lambda表达式也可以使用方法引用来代替
    // 此种方式就是方式一、方式二的简写形式
    list.stream().forEach(System.out::println);
}
```
#### 2. filter 过滤
```
public class User {
    private Long id;
    private String phone;
    private Integer age;

    public User() {}
    public User(Long id, String username, Integer age) {
        this.id = id;
        this.username = username;
        this.age = age;
    }
    // Getter & Setter & toString
}


@Test
public void filter() {
    List<User> users = Arrays.asList(
            new User(1L, "mengday", 28),
            new User(2L, "guoguo", 18),
            new User(3L, "liangliang", 17)
    );

    // Stream<T> filter(Predicate<? super T> predicate);
    users.stream().filter(user -> user.getAge() > 18).forEach(System.out::println);
}
```
#### 3. map 映射
```
@Test
public void map() {
    List<String> list = Arrays.asList("how", "are", "you", "how", "old", "are", "you", "?");
    // <R> Stream<R> map(Function<? super T, ? extends R> mapper);
    list.stream().map(item -> item.toUpperCase()).forEach(System.out::println);
}
```
#### 4. flatMap
```
@Test
public void flatMap() {
    List<Integer> a = Arrays.asList(1, 2, 3);
    List<Integer> b = Arrays.asList(4, 5, 6);

    // <R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper)
    List<List<Integer>> collect = Stream.of(a, b).collect(Collectors.toList());
    // [[1, 2, 3], [4, 5, 6]] 
    System.out.println(collect);

    // 将多个集合中的元素合并成一个集合
    List<Integer> mergeList = Stream.of(a, b).flatMap(list -> list.stream()).collect(Collectors.toList());
    // [1, 2, 3, 4, 5, 6]
    System.out.println(mergeList);

    // 通过Builder模式来构建
    Stream<Object> stream = Stream.builder().add("hello").add("hi").add("byebye").build();
}
```
#### 5. sorted 排序
```
@Test
public void sort() {
    List<String> list = Arrays.asList("c", "e", "a", "d", "b");
    // Stream<T> sorted(Comparator<? super T> comparator);
    // int compare(T o1, T o2);
    list.stream().sorted((s1, s2) -> s1.compareTo(s2)).forEach(System.out::println);
}
```
#### 6. distinct 去重复
```
@Test
public void distinct() {
    // 知之为知之,不知为不知
    Stream<String> stream = Stream.of("know", "is", "know", "noknow", "is", "noknow");
    stream.distinct().forEach(System.out::println); // know is noknow
}
```
#### 7. count 总数量
```
@Test
public void count() {
    Stream<String> stream = Stream.of("know", "is", "know", "noknow", "is", "noknow");
    long count = stream.count();
    System.out.println(count);
}
```
#### 8. min、max
```
@Test
public void min() {
    List<String> list = Arrays.asList("1", "2", "3", "4", "5");
    // Optional<T> min(Comparator<? super T> comparator);
    Optional<String> optional = list.stream().min((a, b) -> a.compareTo(b));
    String value = optional.get();
    System.out.println(value);
}
```
#### 9. skip、limit
```
@Test
public void skip() {
    List<String> list = Arrays.asList("a", "b", "c", "d", "e");
    // Stream<T> skip(long n)
    list.stream().skip(2).forEach(System.out::println);  // c、d、e
}

@Test
public void limit() {
    List<String> list = Arrays.asList("a", "b", "c", "d", "e");
    list.stream().skip(2).limit(2).forEach(System.out::println);    // c、d
}
```
#### 10. collect
```
@Test
public void collect() {
    List<String> list = Arrays.asList("a", "b", "c", "d", "e");
    // Stream -> Collection
    List<String> collect = list.stream().collect(Collectors.toList());

    // Stream -> Object[]
    Object[] objects = list.stream().toArray();
}
```
#### 11. concat
```
@Test
public void concat() {
    List<String> list = Arrays.asList("a", "b");
    List<String> list2 = Arrays.asList("c", "d");
    Stream<String> concatStream = Stream.concat(list.stream(), list2.stream());
    concatStream.forEach(System.out::println);
}
```
#### 12. anyMatch、allMatch
```
@Test
public void match() {
    // 你给我站住
    List<String> list = Arrays.asList("you", "give", "me", "stop");
    // boolean anyMatch(Predicate<? super T> predicate);
    // parallelStream可以并行计算，速度比stream更快
    boolean result = list.parallelStream().anyMatch(item -> item.equals("me"));
    System.out.println(result);
}

/**
* anyMatch伪代码
 * 如果集合中有一个元素满足条件就返回true
 * @return
 */
public boolean anyMatch() {
    List<String> list = Arrays.asList("you", "give", "me", "stop");
    for (String item : list) {
        if (item.equals("me")) {
            return true;
        }
    }
    return false;
}
```
#### 13. reduce 归纳
```
@Test
public void reduce() {
    Stream<String> stream = Stream.of("you", "give", "me", "stop");
    // Optional<T> reduce(BinaryOperator<T> accumulator);
    Optional<String> optional = stream.reduce((before, after) -> before + "," + after);
    optional.ifPresent(System.out::println);    // you,give,me,stop
}

public static void main(String[] args) {
    List<BigDecimal> list = Arrays.asList(
            new BigDecimal("11.11"),
            new BigDecimal("22.22"),
            new BigDecimal("33.33")
    );
    // 66.66
    BigDecimal sum = list.stream().reduce(BigDecimal.ZERO, BigDecimal::add);
    System.out.println(sum);
}
```
#### 14. findFirst、findAny
```
@Test
public void findFirst() {
    Stream<String> stream = Stream.of("you", "give", "me", "stop");
    String value = stream.findFirst().get();
    System.out.println(value);
}

@Test
public void findAny() {
    Stream<String> stream = Stream.of("you", "give", "me", "stop");
    String value2 = stream.findAny().get();
    System.out.println(value2);
}
```