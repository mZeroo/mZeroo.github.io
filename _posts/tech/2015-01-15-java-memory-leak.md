---
layout: post
title: Java 内存泄露 case
category: 技术
tags: java
keywords: java, 内存泄露
description: 
---  
Java 自带垃圾回收器，在编写代码的过程中好像比较少出现内存泄露的情况。 那么是不是真的在编写 Java 代码的过程中就不用在意内存泄露的情况呢？从 GC 的原理出发， 当对象不在被活着的对象引用的时候对象就可以被回收， 那么也就是说如果对象错误的被其他活着的对象引用就会造成该对象不能被回收也就造成了内存泄露。造成内存泄露的原因如下图所示：

![内存泄露](/public/img/posts/object-life-time.jpg)

#### 下面收集一些可能造成内存泄露的情况:
##### 1. <b>错误的 hashcode 导致内存泄露</b>

首先来看一个简单的例子:

    
    static class TestClass {
        private int value = 0;

        public void setValue(int value) {
            this.value = value;
        }

        @Override
        public int hashCode() {
            return value;
        }
    }

    public static void main(String[] args) {
        Set<TestClass> set = new HashSet<TestClass>();
        TestClass testClass = new TestClass();
        set.add(testClass);
        System.out.println(set.size());
        testClass.value = 10;
        set.add(testClass);
        System.out.println(set.size());
        set.remove(testClass);
        System.out.println(set.size());
    }
   
执行的输出结果如下所示：

	1
	2
	1

了解 Hashset 原理的都知道， hashset 根据 hashcode 将 item 哈希到不同的 slot，上述的例子因为更改了 value 的值，导致 hashcode 发生变化，那么再将 item 添加到 set 的时候， 由于 hashcode 不一样， 那么 set 将其视为另一个对象， 这样 set 中保留了 2 个对于 item 的引用。 在后续执行删除的过程中只删除了其中一个引用， 由于 set 还持有对于 item 的一个引用，所以 GC 无法将该对象回收，也就造成了内存泄露。  

<br>
##### 2. <b>匿名内部类导致的内存泄露</b>

先看一下如下的例子:

    
    static class OuterClass {
        public Comparator<Integer> getIntComparator() {
            return new Comparator<Integer>() {
                @Override
                public int compare(Integer o1, Integer o2) {
                    return o1 - o2;
                }
            };
        }
    }

    public static void main(String[] args) {
        Comparator<Integer> comparator = new OuterClass().getIntComparator();
        Collections.sort(Lists.newArrayList(1, 2, 3, 4), comparator);
    }

上面的例子返回了一个匿名内部类的对象， 这样有什么问题呢？我们再看下面的例子：

   
    static class OuterClass {
        private String value = "Get U";

        public Comparator<Integer> getIntComparator() {
            return new Comparator<Integer>() {
                @Override
                public int compare(Integer o1, Integer o2) {
                    System.out.println(OuterClass.this.value);
                    return o1 - o2;
                }
            };
        }
    }

    public static void main(String[] args) {
        Comparator<Integer> comparator = new OuterClass().getIntComparator();
        Collections.sort(Lists.newArrayList(1, 2, 3, 4), comparator);
    }

执行之后输出结果为：

	Get U
	Get U
	Get U


从上面的测试中可以发现产生的匿名内部类会持有 OuterClass 的引用，这样在这个匿名内部类对象能够被回收之前 OuterClass 产生的对象都不能被 GC 回收，因此会导致内存泄露。  

<br>
##### 3. <b>集合类造成的内存泄露</b>

如下例:

    
    static class TestClass {
    }

    public static void main(String[] args) {
        List<TestClass> list = Lists.newArrayList();
        for (int i = 0; i < 100; i++) {
            TestClass testClass = new TestClass();
            list.add(testClass); // do something testClass = null;
        }
    }
将上述例子中的 List 换成 Map，Set 等也是一样的， 归结起来就是把对象放入到集合类中， 在集合类能够被回收之前放入的对象都不能被 GC 所回收， 可能会因此造成内存泄露。 其中 WeakHashMap 的出现可以解决因此造成的内存泄露。  
可能上述例子有点 trick， 那么我们看看网上一些 stack 的实现， 一些代码是搜索 “java simple stack implement” 的其中一个:

    
    public class MyStack {
        private int maxSize;
        private Object[] stackArray;
        private int top;

        public MyStack(int s) {
            maxSize = s;
            stackArray = new Object[maxSize];
            top = -1;
        }

        public void push(Object j) {
            stackArray[++top] = j;
        }

        public Object pop() {
            return stackArray[top--];
        }

        public Object peek() {
            return stackArray[top];
        }

        public boolean isEmpty() {
            return (top == -1);
        }

        public boolean isFull() {
            return (top == maxSize - 1);
        }
    }
这个有什么问题吗？ 我们注意到 pop 的时候仅仅将 top–， 并未将 stackArray 中 top 的位置置为 null，那么也就是说虽然执行了 pop， 但是 stackArray 仍然持有对象的引用，那么也就无法被 GC 回收， 可能会造成内存泄露。  

// TODO：其他类型的内存泄露，classloader 导致的内存泄露， native 内存泄露等。

