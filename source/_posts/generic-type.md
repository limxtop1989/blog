---
title: 一文读懂Java范型
date: 2023-09-15 21:00:07
tags: 范型
categories:
- [Java]
---

# Terms
| Term | Example |
| --- | --- |
| Parameterized type | List\<String\> |
| Actual type parameter | String |
| Generic type | List\<E\> |
| Formal type parameter | E |
| Unbonded wildcard type | List\<?\> |
| Raw type | List |
| Bounded type parameter | \<E extends Number\> |
| Recursive type bond | \<T extends Comparable\<T\>\> |
| Bounded widcard type | List\<? extends Numbers\> |
| Generic method | static \<E\> List\<E\> asList(E[] a) |
| Type token | String.class |

# 范型定义
一个类或者接口，它的声明里，有 type parameter (T)，它就是范型类或者接口。

假设我们要定义一个容器类，它可以存放任意类型的实力，我们先看[非范型](https://docs.oracle.com/javase/tutorial/java/generics/types.html)的实现。
```java
public class Box {
    private Object object;

    public void set(Object object) { this.object = object; }
    public Object get() { return object; }

    public static void main(String[] args, int argc) {
        Box i = new Box();
        i.set(new Integer(1));
        String s = (String) i.get(); // 1. run time error.
    }
}
```
这种实现有个缺陷，编译时无法验证Box 存放的实例类型，编译正常通过，但在运行时，一部分代码使用Box 存放 String， 理所当然地期望，取出来的也是 String，但如果有人意外地，在其他代码位置，放进一个 Integer，代码注释1 处，便发生运行时异常。

```java
/**
 * Generic version of the Box class.
 * @param <T> the type of the value being boxed
 */
public class Box<T> {
    // T stands for "Type"
    private T t;

    public void set(T t) { this.t = t; }
    public T get() { return t; }

    public static void main(String[] args, int argc) {
        Box<String> i = new Box<String>(); // 1. generic type invocation.
        i.set(new Integer(1)); // 2. compile error.
        String s = (String) i.get(); // 3. run time error.
    }
}
```
范型优点：改造成范型类后，由于我们定义Box 只能放 String 类型，注释2 处编译不通过。错误的代码，在编译时被检测出来，而不是运行时。再者，运行时的错误，并非容易找到根源，导致错误的位置可能离发生错误的位置很远。


## 范型类调用 [generic type invocation](https://docs.oracle.com/javase/tutorial/java/generics/types.html)

To reference the generic Box class from within your code, you must perform a generic type invocation, which replaces T with some concrete value, such as Integer:

```java
Box<Integer> integerBox;
```

You can think of a generic type invocation as being similar to an ordinary method invocation, but instead of passing an argument to a method, you are passing a type argument — Integer in this case — to the Box class itself.


## [范型方法](https://docs.oracle.com/javase/tutorial/java/generics/methods.html)
Generic methods are methods that introduce their own type parameters. This is similar to declaring a generic type, but the type parameter's scope is limited to the method where it is declared.

The syntax for a generic method includes a list of type parameters, inside angle brackets, which appears before the method's return type. (public static __<K, V>__ boolean compare)

```java
public class Util {
    public static <K, V> boolean compare(Pair<K, V> p1, Pair<K, V> p2) {
        return p1.getKey().equals(p2.getKey()) &&
               p1.getValue().equals(p2.getValue());
    }
}
```
The complete syntax for invoking this method would be (Util. __<Integer, String>__ compare(p1, p2);):

```java
Pair<Integer, String> p1 = new Pair<>(1, "apple");
Pair<Integer, String> p2 = new Pair<>(2, "pear");
boolean same = Util.<Integer, String>compare(p1, p2);
```

# 范型与数组
## 范型和数组有重要的不同
1. 数组是协变，即如果 Sub 是 Super 的子类，那么 Sub[] 是 Super[]的子类。相反，范型不是协变的，对于任意两个不同的类 Type1 & Type2， List<Type1> 既不是 List<Type2> 的子类，也不是它的超类。 协变带来缺陷：

```java
// Fails at runtime!
Object[] objectArray = new Long[1];
objectArray[0] = "I don't fit in"; // Throws ArrayStoreException

// Won't compile!
List<Object> ol = new ArrayList<Long>(); // Incompatible types.
ol.add("I don't fit in");
```

2. 数组是 reified， 意味着数组在运行时才知道和强制约束它的元素类型，但范型，与之相反，是通过范型擦除实现，意味着它在编译时强制类型约束，在运行时丢弃（擦除）范型信息。

## 范型数组
由于以上的区别，创建范型数组是非法的。 new List<E>(), new List<String>[], new E[], 在编译时会报错。
> It is illegal to create an array of a generic type, a parameterized type, or a type parameter. [^4]

```java
// Why generic array creation is illegal - won't compile!
List<String>[] stringLists = new List<String>[1] // 1. 假设Line1我们可以创建范型数组
List<Integer> intList = Arrays.asList(42);  // 2.
Object[] objects = stringLists;   // 3. 合法，数组是协变的
objects[0] = intList;  // 4. 合法，运行时范型已经被擦除，List<Integer> 成为List，List<String>[]成为List[]，没有ArrayStoreException
String s = stringLists[0].get(0);  //  5. ClassCastException at runtime.
```
## List<E> 优于 E[]
```java
static <E> E reduce(List<E> list, Function<E> f, E initVal) {
    // compiler can't check the safety of the cast at runtime because it doesn't know what E is at runtime.
    E[] snapshot = (E) list.toArray();
    E result = intVal;
    for (E e : snapShot) {
        result = apply(result, e);
    }
    return result;
}
```
```java
static <E> E reduce(List<E> list, Function<E> f, E initVal) {
    List<E> snapshot;
    synchronized(list) {
        snapshot = new ArrayList<E>(list);
    }
    E result = intVal;
    for (E e : snapShot) {
        result = apply(result, e);
    }
    return result;
}
```
## 范型擦除 Type Erasure

Generics were introduced to the Java language to provide tighter type checks at compile time and to support generic programming. To implement generics, the Java compiler applies type erasure to:

1. Replace all type parameters in generic types with their bounds or Object if the type parameters are unbounded. The produced bytecode, therefore, contains only ordinary classes, interfaces, and methods.
2. Insert type casts if necessary to preserve type safety.
3. Generate bridge methods to preserve polymorphism in extended generic types.
Type erasure ensures that no new classes are created for parameterized types; consequently, generics incur no runtime overhead.

### Erasure of Generic Types
During the type erasure process, the Java compiler erases all type parameters and replaces each with its first bound if the type parameter is bounded, or Object if the type parameter is unbounded.

Consider the following generic class that represents a node in a singly linked list:
```java 
public class Node<T> {

    private T data;
    private Node<T> next;

    public Node(T data, Node<T> next) {
        this.data = data;
        this.next = next;
    }

    public T getData() { return data; }
    // ...
}
```
Because the type parameter T is unbounded, the Java compiler replaces it with Object:
```java 
public class Node {

    private Object data;
    private Node next;

    public Node(Object data, Node next) {
        this.data = data;
        this.next = next;
    }

    public Object getData() { return data; }
    // ...
}
```
In the following example, the generic Node class uses a bounded type parameter:

```java
public class Node<T extends Comparable<T>> {

    private T data;
    private Node<T> next;

    public Node(T data, Node<T> next) {
        this.data = data;
        this.next = next;
    }

    public T getData() { return data; }
    // ...
}
```
The Java compiler replaces the bounded type parameter T with the first bound class, Comparable:

```java
public class Node {

    private Comparable data;
    private Node next;

    public Node(Comparable data, Node next) {
        this.data = data;
        this.next = next;
    }

    public Comparable getData() { return data; }
    // ...
}
```

### Extends generic type
[override & generic](https://stackoverflow.com/questions/77117276/how-to-explain-the-type-of-return-value-and-parameter-definition-at-sub-class-in)

### Effects of Type Erasure and Bridge Methods 
Sometimes type erasure causes a situation that you may not have anticipated. The following example shows how this can occur. The following example shows how a compiler sometimes creates a synthetic method, which is called a bridge method, as part of the type erasure process.

Given the following two classes:
```java
public class Node<T> {

    public T data;

    public Node(T data) { this.data = data; }

    public void setData(T data) {
        System.out.println("Node.setData");
        this.data = data;
    }
}

public class MyNode extends Node<Integer> {
    public MyNode(Integer data) { super(data); }

    public void setData(Integer data) {
        System.out.println("MyNode.setData");
        super.setData(data);
    }
}
```
Consider the following code:
```java
MyNode mn = new MyNode(5);
Node n = mn;            // A raw type - compiler throws an unchecked warning
n.setData("Hello");     // Causes a ClassCastException to be thrown.
Integer x = mn.data;    
```
After type erasure, this code becomes:
```java
MyNode mn = new MyNode(5);
Node n = mn;            // A raw type - compiler throws an unchecked warning
                        // Note: This statement could instead be the following:
                        //     Node n = (Node)mn;
                        // However, the compiler doesn't generate a cast because
                        // it isn't required.
n.setData("Hello");     // Causes a ClassCastException to be thrown.
Integer x = (Integer)mn.data; 
```
The next section explains why a ClassCastException is thrown at the n.setData("Hello"); statement.

Bridge Methods
When compiling a class or interface that extends a parameterized class or implements a parameterized interface, the compiler may need to create a synthetic method, which is called a bridge method, as part of the type erasure process. You normally don't need to worry about bridge methods, but you might be puzzled if one appears in a stack trace.

After type erasure, the Node and MyNode classes become:
```java
public class Node {

    public Object data;

    public Node(Object data) { this.data = data; }

    public void setData(Object data) {
        System.out.println("Node.setData");
        this.data = data;
    }
}

public class MyNode extends Node {

    public MyNode(Integer data) { super(data); }

    public void setData(Integer data) {
        System.out.println("MyNode.setData");
        super.setData(data);
    }
}
```
After type erasure, the method signatures do not match; the Node.setData(T) method becomes Node.setData(Object). As a result, the MyNode.setData(Integer) method does not override the Node.setData(Object) method.

To solve this problem and preserve the polymorphism of generic types after type erasure, the Java compiler generates a bridge method to ensure that subtyping works as expected.

For the MyNode class, the compiler generates the following bridge method for setData:
```java
class MyNode extends Node {

    // Bridge method generated by the compiler
    //
    public void setData(Object data) {
        setData((Integer) data);
    }

    public void setData(Integer data) {
        System.out.println("MyNode.setData");
        super.setData(data);
    }

    // ...
}
```
The bridge method MyNode.setData(object) delegates to the original MyNode.setData(Integer) method. As a result, the n.setData("Hello"); statement calls the method MyNode.setData(Object), and a ClassCastException is thrown because "Hello" can't be cast to Integer.

# Bounded Type Parameters
There may be times when you want to restrict the types that can be used as type arguments in a parameterized type. For example, a method that operates on numbers might only want to accept instances of Number or its subclasses. This is what __bounded__ type parameters are for.

To declare a bounded type parameter, list the type parameter's name, followed by the extends keyword, followed by its upper bound, which in this example is Number. Note that, in this context, extends is used in a general sense to mean either "extends" (as in classes) or "implements" (as in interfaces).
```java
public class Box<T> {

    private T t;          

    public void set(T t) {
        this.t = t;
    }

    public T get() {
        return t;
    }

    public <U extends Number> void inspect(U u){
        System.out.println("T: " + t.getClass().getName());
        System.out.println("U: " + u.getClass().getName());
    }

    public static void main(String[] args) {
        Box<Integer> integerBox = new Box<Integer>();
        integerBox.set(new Integer(10));
        integerBox.inspect("some text"); // error: this is still String!
    }
}
```
By modifying our generic method to include this bounded type parameter, compilation will now fail, since our invocation of inspect still includes a String:

```java
Box.java:21: <U>inspect(U) in Box<java.lang.Integer> cannot
  be applied to (java.lang.String)
                        integerBox.inspect("10");
                                  ^
1 error
```
In addition to limiting the types you can use to instantiate a generic type, __bounded type parameters allow you to invoke methods defined in the bounds__ :
```java
public static <T extends Comparable<T>> int countGreaterThan(T[] anArray, T elem) {
    int count = 0;
    for (T e : anArray)
        if (e.compareTo(elem) > 0)
            ++count;
    return count;
}
```
The countGreaterThan method invokes the compareTo method defined in the Comparable class through e.


# Bounded wildcards 
由于 parameterized types 不支持 协变 (invariant), 就是说，对于任意两个不同的类 Type1 & Type2， List<Type1> 既不是 List<Type2> 的子类，也不是它的超类。  List<String> 不是 List<Object> 的子类，初看起来有点反直觉，但它确实有意义，我们可以放任意 object 到 List<Object>， 但我们只能放 string 到 List<String>. 但这样非协变的类型没有伸缩性 (flexibility)。 如下展开解释：
## Upper Bounded Wildcards
```java
public class Stack<E> {
    public Stack();
    public void push(E e);
    public E pop();
    public boolean isEmpty();
}
```
假设我们要增加一个方法，它输入一个可迭代序列，然后 push 他们到这个 Stack， 以下第一次尝试的代码：

```java
// pushAll method without wildcard type -- works but is deficient!
public void pushAll(Iterable<E> src) {
    for (E e : src) {
        push(e);
    }
}
```
这代码能工作，但并不能满意。如果 src 的元素类型和 Stack的完全匹配，它运行没问题。但假设我们有一个 Stack<Number> ， 然后调用 push(intVal), intVal 是一个 Integer 类型。这完全没问题，因为 Integer 是 Number 的子类，因此，逻辑上，以下代码也应该没问题：

```java
Stack<Number> numberStack = new Stack<Number>();
Iterable<Integer> integers = ...;
numberStack.pushAll(integers);
```
但实际上，它编译时报错，因为 Iterable<Integer> 并非 Iterable<Number> 的子类，intergers 不能传参给 src。幸好，Java 提供 有边界通配符类型 (bounded wildcard type) 来解决这个场景。 pushAll __的入参类型不应该是 "Iterable of E", 由Iterable<E>表示，而应该是 "Iterable of some subtype of E"，由 Iterable<? extends E>__ 表示。如下改写 Stack pushAll 方法声明后，上方的客户端代码就可以编译通过了，因为 Iterable<Integer> 是 Iterable<? extends Number> 的子类
```java
// Wildcard type for marameter that serves as an E producer
public void pushAll(Iterable<? extends E> src) {
    for (E e: src) {
        push(e);
    }
}
```

## Lower Bounded Wildcards
假设我们现在需要实现一个 popAll 的方法，和 pushAll 配套，这个方法 pop stack 里面的元素，并逐一加到给定集合参数，看下第一次尝试的代码：
```java
// popAll method without wildcard type - deficient
public void popAll(Collection<E> dst) {
    while (!isEmpty()) {
        dst.add(pop());
    }
}
```
假设我们有一个 Stack<Number> 和一个 Object 类型的变量，如果我们 pop 一个元素并把它存放在该变量，这完全没问题。所以，如下代码也应该没问题的，对么？

```java
Stack<Number> numberStack = new Stack<Number>();
Collection<Object> objects = ...;
numberStack.popAll(objects);
```

但很遗憾，这个代码编译不通过，因为 Collection<Object> 不是 Collection<Number> 的子类。 下边界通配符类型提供了解决方案， __入参不应该是 "Collection of E" ，由Collection<E>表示，而应该是 "collection of some supertype of E" ， 由 Collection<? super E>__ 表示。
```java
// Wildcard type for parameter that serves as an E consumer
public void popAll(Collection<? super E> dst) {
    while (!isEmpty()) {
        dst.add(pop());
    }
}
```
由上，教训是很清晰的，为了API最大伸缩性，在代表生产者和消费者的入参使用通配符类型。助记口诀：

> PECS strands for producer-extends, consumer-super

The `reduce` method use `list` parameter as an `E` producer, so its declaration should use a wildcard type that extends `E`. whereas, The parameter `f` represents a function that both consumes and produces `E` instances, so a wildcard type would be inappropriate for it.
## 重构Reduce
```java
static <E> E reduce(List<? extends E> list, Function<E> f, E intVal)
```
Would this change make any difference in practice? As it turns out, it wold. Suppose you have a `List<Integer>`, and you want to reduce it with a `Function<Number>`. This would not compile with the original declarationo previously, but it does once you add the bounded wildcard type.

## Recursive type bound
Recursive type bound definition: A type parameter to be bounded by some expression involving that type parameter itself.
```java
public interface Comparable<T> {
    int compareTo(T 0);
}

public static <T extends Comparable<T>> T max(List<T> list) {
    Iterator<T> i = list.iterator();
    T result = i.next();
    while (i.hasNext()) {
        T t = i.next();
        if (t.compareTo(result) > 0) {
            result = t;
        }
    }
    return result;
}
```
The type bound `<T extends Comparable<T>>` may be read as "for every type T that can be compared to itself".

但我们不能调用 `max(List<ScheduledFuture<?>> scheduledFutures)`，原因在于，ScheduledFuture 并不实现 `Comparable<ScheduledFuture>` 接口，相反，它不过是 `Delayed` 子接口，它继承的是 `Comparable<Delayed>`，换句话说， `ScheduledFuture` 实例并不仅仅和其他 `ScheduledFuture` 实例比较，它还可以和任意的 `Delayed` 实例进行比较。

```java
public interface ScheduledFuture<V> extends Delayed, Future<V> {
}

public interface Delayed extends Comparable<Delayed> {
}

public interface Comparable<T> {
    int compareTo(T 0);
}

public static <T extends Comparable<? super T>> T max(List<? extends T> list) {
    Iterator<? extends T> i = list.iterator();
    T result = i.next();
    while (i.hasNext()) {
        T t = i.next();
        if (t.compareTo(result) > 0) {
            result = t;
        }
    }
    return result;
}
```

> You should always use Comparable<? super T> in preference to Comparable<T>. The same is true of comparators.

## Unbounded Wildcards
The unbounded wildcard type is specified using the wildcard character (?), for example, List<?>. This is called a list of unknown type. There are two scenarios where an unbounded wildcard is a useful approach:

1. If you are writing a method that can be implemented using functionality provided in the Object class.
2. When the code is using methods in the generic class that don't depend on the type parameter. For example, List.size or List.clear. In fact, Class<?> is so often used because most of the methods in Class<T> do not depend on T.
Consider the following method, printList:
```java
public static void printList(List<Object> list) {
    for (Object elem : list)
        System.out.println(elem + " ");
    System.out.println();
}
```
The goal of printList is to print a list of any type, but it fails to achieve that goal — it prints only a list of Object instances; it cannot print List<Integer>, List<String>, List<Double>, and so on, because they are not subtypes of List<Object>. To write a generic printList method, use List<?>:
到这里，其实可以理解 List<?> 是 List<? extends Object> 的简写
```java
public static void printList(List<?> list) {
    for (Object elem: list)
        System.out.print(elem + " ");
    System.out.println();
}
```
Because for any concrete type A, List<A> is a subtype of List<?>, you can use printList to print a list of any type:
```java
List<Integer> li = Arrays.asList(1, 2, 3);
List<String>  ls = Arrays.asList("one", "two", "three");
printList(li);
printList(ls);
```

It's important to note that `List<Object>` and `List<?>` are not the same. You can insert an Object, or any subtype of Object, into a `List<Object>`. But you can only insert null into a List<?>. (Because the type of element in the List<?> is not certain, the compiler can't check the inserting operation is type safe or not, unless you insert null value) The Guidelines for Wildcard Use section has more information on how to determine what kind of wildcard, if any, should be used in a given situation.

## Wildcards and Subtyping
如果 Sub 是 Super 的子类，那么
1. 范型不支持协变 `List<Sub>` & `List<Super>` 没有关系;
2. `List<Sub>` 是 List<? extends Super> 的子类；
3. `List<Super>` 是 List<? super Sub> 的子类；
| Invariant | Wildcard |
| --------- | -------- |
| ![](https://docs.oracle.com/javase/tutorial/figures/java/generics-subtypeRelationship.gif) | ![](https://docs.oracle.com/javase/tutorial/figures/java/generics-wildcardSubtyping.gif) | 

# Key points
1. Don't use a wildcard as a return type, because it forces programmers to use wildcard types in client codes.
2. If a input parameter is both producer and consumer, define it as exact type match, rather than wildcard types.




[^4]: Effect Java Item 25
