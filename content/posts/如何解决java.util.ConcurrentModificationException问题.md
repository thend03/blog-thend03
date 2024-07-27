---
draft: false
date: 2024-07-26T17:07:58+08:00
title: "如何解决java.util.ConcurrentModificationException问题"
slug: "how-to-fix-java.util.ConcurrentModificationException" 
tags: ["java","juc","exception"]
categories: ["java"]
authors: ["since"]
description: "如何解决java.util.ConcurrentModificationException问题"
disableShare: true # 底部不显示分享栏
---



## 问题

最近在debug的时候，莫名奇妙的会遇到`java.util.ConcurrentModificationException`问题。



根据我的历史经验，发生这种问题肯定是for循环里调了remove或者add。



但是看了一圈代码，没发现此类操作，有点蒙圈，这是为啥。。。







## 根因分析

平常遇到的`java.util.ConcurrentModificationException`大多是下面的第一种，迭代器遍历和集合类的add/remove方法同时调用了。



像增强for循环底层也属于迭代器遍历，所以这种错误是比较常见的。



我这次遇到的就真的是多线程场景下的并发修改错误。



### 并发修改错误原因分析

`java.util.ArrayList.Itr`是ArrayList的内部类，expectedModCount是属于Itr的成员变量,。



modCount是`java.util.AbstractList`的成员变量。



首先看一下Itr类的定义

```java
    /**
     * An optimized version of AbstractList.Itr
     */
    private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        Itr() {}

        public boolean hasNext() {
            return cursor != size;
        }
       //省略其他代码
    }
```



在生成迭代器Itr的时候，expectedModCount相当于是拿的当前ArrayList的modCount的值。



后续list.add()或者list.remove()只会修改modCount，expectedModCount是不会受list.add()或者list.remove()影响的。



再进行迭代器遍历的时候就会抛出`java.util.ConcurrentModificationException`。





### 迭代器和集合类方法同时使用 

java的集合类有如下2个字段, 翻译过来就是expectedModCount!=modCount的时候，会抛出并发异常。

期望是修改的数量和期望值相同的，不同的时候肯定是有问题了。

```java
        /**
         * The modCount value that the iterator believes that the backing
         * List should have.  If this expectation is violated, the iterator
         * has detected concurrent modification.
         */
        int expectedModCount = modCount;
```



像如下这种写法，肯定会抛出`java.util.ConcurrentModificationException`异常的。

因为for循环底层是使用的迭代器,这种情况就会导致并发修改错误。

```java
public static void main(String[] args) {
        List<String> list = new ArrayList<>();

        list.add("a");
        list.add("b");
        list.add("c");
        list.add("d");
        list.add("e");

        for (String a : list) {
            System.out.println(a);
            list.remove(0);
        }
    }
```



这段代码编译为class，结果如下, 可以看到for循环变成了iterator迭代遍历。

遍历是使用的iterator，但是remove方法是集合类的自己的方法。

```java
public static void main(String[] args) {
        List<String> list = new ArrayList();
        list.add("a");
        list.add("b");
        list.add("c");
        list.add("d");
        list.add("e");
        Iterator var2 = list.iterator();

        while(var2.hasNext()) {
            String a = (String)var2.next();
            System.out.println(a);
            list.remove(0);
        }

    }
```



`java.util.ArrayList.Itr#next`迭代器的next会先校验modCount != expectedModCount, 2个值不相等就抛出异常

```java
@SuppressWarnings("unchecked")
public E next() {
  checkForComodification();
  int i = cursor;
  if (i >= size)
    throw new NoSuchElementException();
  Object[] elementData = ArrayList.this.elementData;
  if (i >= elementData.length)
    throw new ConcurrentModificationException();
  cursor = i + 1;
  return (E) elementData[lastRet = i];
}
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```



正常使用迭代器的话modCount == expectedModCount，这种情况是不会抛异常的。

但是list.remove(0)，会修改modCount，而不修expectedModCount,导致下一次迭代报错

```java
public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
}
```





总结一下就是: 

> 迭代器遍历不能和集合类自身的add/remove方法一起调用，这样会导致modCount和expectedModCount不相等，从而抛出`java.util.ConcurrentModificationException`。
>
> add/remove都是集合类自身的方法，都只修改modCount而不修改expectedModCount



### 并发修改导致的异常

之前遇到的都是上面一种导致的异常，这次真的就遇到多线程场景下的`java.util.ConcurrentModificationException`了。



看一下下面的代码

```java
 public static void main(String[] args) {
        List<String> list = new ArrayList<>();

        list.add("a");
        list.add("b");
        list.add("c");
        list.add("d");
        list.add("e");

        Thread thread = new Thread(() -> list.forEach(a -> { System.out.println(a); }));
        thread.start();

        Thread thread1 = new Thread(() -> list.sort((o1, o2) -> o2.length() - o1.length()));
        thread1.start();
    }
```



这段代码直接执行是没有问题的，可以正常结束，但是稍微修改一下，加点延迟，就会有问题了

```java
   public static void main(String[] args) {
        List<String> list = new ArrayList<>();

        list.add("a");
        list.add("b");
        list.add("c");
        list.add("d");
        list.add("e");

        Thread thread = new Thread(() -> list.forEach(a -> {
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println(a);
        }));
        thread.start();

        Thread thread1 = new Thread(() -> list.sort((o1, o2) -> o2.length() - o1.length()));
        thread1.start();
    }

//运行异常
Exception in thread "Thread-0" java.util.ConcurrentModificationException
	at java.util.ArrayList.forEach(ArrayList.java:1262)
	at com.fc.se.list.ListTest.lambda$main$1(ListTest.java:23)
	at java.lang.Thread.run(Thread.java:750)
```





看异常堆栈是`java.util.ArrayList#forEach`方法, 会校验modCount != expectedModCount，不符合预期就报错

```java
    @Override
    public void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        final int expectedModCount = modCount;
        @SuppressWarnings("unchecked")
        final E[] elementData = (E[]) this.elementData;
        final int size = this.size;
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            action.accept(elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
```



现在这个场景变成了在2个线程里对同一个list进行遍历和sort操作.



看下sort操作

```java
 @Override
 @SuppressWarnings("unchecked")
 public void sort(Comparator<? super E> c) {
     final int expectedModCount = modCount;
     Arrays.sort((E[]) elementData, 0, size, c);
     if (modCount != expectedModCount) {
     throw new ConcurrentModificationException();
     }
     modCount++;
 }
```



这2个代码放一起比较一下就可以看出端倪了，sort()会修改modeCount, 但是不会修改expectedModCount。

再次进行list.foreach()时，由于`modCount != expectedModCount`，就会抛出`ConcurrentModificationException`。



如果不加`Thread.sleep(50);`，thread会迅速执行完成，相当于2个线程串行执行，所以不会有并发修改问题。

加了`Thread.sleep(50);`，2个线程会并发执行，就会抛异常了。



## Fail-Fast

上面的栗子就是fail-fast的一种场景，不符合预期，直接报错。

### 什么是fail-fast

首先我们看下维基百科中关于fail-fast的解释：

> In systems design, a fail-fast system is one which immediately reports at its interface any condition that is likely to indicate a failure. Fail-fast systems are usually designed to stop normal operation rather than attempt to continue a possibly flawed process. Such designs often check the system's state at several points in an operation, so any failures can be detected early. The responsibility of a fail-fast module is detecting errors, then letting the next-highest level of the system handle them.



大概意思是：在系统设计中，快速失效系统一种可以立即报告任何可能表明故障的情况的系统。

快速失效系统通常设计用于停止正常操作，而不是试图继续可能存在缺陷的过程。

这种设计通常会在操作中的多个点检查系统的状态，因此可以及早检测到任何故障。

快速失败模块的职责是检测错误，然后让系统的下一个最高级别处理错误。



其实，这是一种理念，说白了就是在做系统设计的时候先考虑异常情况，一旦发生异常，直接停止并上报。



## Fail-Safe

与之相对的还有fail-safe，这是一种并发安全机制。



为了避免触发fail-fast机制，导致异常，我们可以使用Java中提供的一些采用了fail-safe机制的集合类。

这样的集合容器在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。

java.util.concurrent包下的容器都是fail-safe的，可以在多线程下并发使用，并发修改。同时也可以在foreach中进行add/remove 。



## 参考文章

[一不小心就踩坑的fail-fast是个什么鬼?](https://www.hollischuang.com/archives/3542)
