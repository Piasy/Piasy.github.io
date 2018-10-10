---
layout: post
title: 再谈 Java 深浅拷贝
tags:
    - Java
---

[去年的一篇文章](http://blog.piasy.com/2015/09/16/Java-Defensive-Copy-Immutable-Unmodifiable/){:target="_blank"}总结了一下深浅拷贝，Immutable 和 unmodifiable 这三个概念，今天再看看 Java 的深浅拷贝。

考虑以下代码：

``` java
static class ImmutablePerson {
    final String mName;

    ImmutablePerson(String name) {
        mName = name;
    }
}

List<ImmutablePerson> list1 = new ArrayList<>();
list1.add(new ImmutablePerson("Piasy"));
List<ImmutablePerson> list2 = new ArrayList<>(list1);
```

`list2 = new ArrayList<>(list1)` 是否是一次深拷贝呢？实际情况是这样的：

+ `list2` 和 `list1` 是两个不同的对象，为 `list2` 增加一个元素，并不会导致 `list1` 也增加一个元素
+ 但是 `list2` 内的元素，和 `list1` 内的元素，也就是它俩唯一的一个元素 `Person` 对象，是同一个对象，内存地址都是一样的
+ 查看 ArrayList 的构造函数，我们发现它通过 `System.arraycopy` 来拷贝传入的数据，这说明 [`System.arraycopy` 进行的是浅拷贝](http://stackoverflow.com/questions/6101684/does-java-lang-system-arraycopy-use-a-shallow-copy){:target="_blank"}，而[这个测试用例](https://github.com/Piasy/AndroidPlayground/blob/master/reproduce/NotificationTest/src/test/java/com/github/piasy/notificationtest/ShallowDeepCopyTest.java#L13){:target="_blank"}的结果也同样证明了这个结论
+ 但是由于 `Person` 类的 `mName` 声明为了 `final`，而 `String` 又是 immutable 的，所以实际上，`Person` 类就是 immutable 的，因此，这里 `System.arraycopy` 进行的浅拷贝不会带来任何问题
+ 但是必须注意，一旦 `Person` 类不是 immutable 的，那这里就可能会产生 bug 了，例如， `list1.get(0).mName = "Test"`，就会导致 `list2.get(0).mName` 也变成了 `Test`，[这个测试用例](https://github.com/Piasy/AndroidPlayground/blob/master/reproduce/NotificationTest/src/test/java/com/github/piasy/notificationtest/ShallowDeepCopyTest.java#L43){:target="_blank"}的结果证明了这个结论

所以去年的文章中描述的深拷贝并不严谨，实际上还是浅拷贝。那么 Java 里面什么才是深拷贝呢？（实现正确的） serialization！为什么需要说是实现正确的呢？因为只有在 `clone` 方法中 new 了一个新对象，并且对于每个成员，都是 new 了一个新对象再赋值，这才是彻彻底底的深拷贝，没有任何一个成员变量是以前的副本的成员变量的浅拷贝。[这个测例](https://github.com/Piasy/AndroidPlayground/blob/master/reproduce/NotificationTest/src/test/java/com/github/piasy/notificationtest/ShallowDeepCopyTest.java#L63){:target="_blank"}实际上就不是一个实现正确的 serialization。

但这样彻底的深拷贝开销太大，所以并不实用，更好的做法还是 Immutable，一旦实现了 Immutable，浅拷贝反倒是最好的搭档了，既高效又安全。而为了降低开销，并且保证语义的准确，最好的实践还是用 unmodifiable 来实现 immutable。
