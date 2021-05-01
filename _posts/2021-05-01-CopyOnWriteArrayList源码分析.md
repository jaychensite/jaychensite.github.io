---
layout: post
title: "CopyOnWriteArrayList源码分析"
date: 2021-05-01
description: "CopyOnWriteArrayList源码分析"
tag: Java基础


---

### CopyOnWriteArrayList介绍

从名字CopyOnWrite就能看出是写时复制的容器。通俗来讲就是，当你修改数据时，拷贝一份数据拿出来修改。修改完后，再将原来对象的引用指向新的引用对象。为什么这样做了？因为大多数情况下，读操作是远远要大于写操作需求，如果再并发情况下读操作也进行加锁，可想而知性能得有多低效。因此就出现读写分离的思想。接下来看看java中CopyOnWriteArrayList底层是如何实现读写分离。

### 简单用列

```java
 // 创建一个 CopyOnWriteArrayList 对象
 CopyOnWriteArrayList copy = new CopyOnWriteArrayList();、
 // 添加值操作
 copy.add("读写分离");
 // 更改指定位置的值，因为底层还是数组的实现，所以这里可以直接根据下标修改值
 copy.set(0, "CopyOnWriteArrayList");
```



### 源码分析

##### ADD

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
  
  /** 锁：通过ReentrantLock对写操作进行加锁，来实现读写分离*/
  final transient ReentrantLock lock = new ReentrantLock();
	
  /** array数组，存储元素的数组 */
  private transient volatile Object[] array;
  
  /**
    * Appends the specified element to the end of this list.
    * 将元素添加进集合的末尾，因此也是有序列表
    * @param e element to be appended to this list 元素
    * @return {@code true} (as specified by {@link Collection#add})
    */
  public boolean add(E e) {
    		/** 获取ReentrantLock*/
        final ReentrantLock lock = this.lock;
    		/** 获取锁：这里就不详细讲ReentrantLock的实现方式，ReentrantLock下一期再分析，目前的话就先知道是一个获取锁的东西  */
    		/** 如果获取锁成功则就会进入添加的业务逻辑*/
        lock.lock();
        try {	
          	/** 获取锁成功后则会进入这里 */
          	/** 获取存储元素的数组 */
            Object[] elements = getArray();
          	/** 获取数组长度 */
            int len = elements.length;
          	/** 重点，这里就是将原来数组复制一份新的数组，长度 + 1 */
            Object[] newElements = Arrays.copyOf(elements, len + 1);
          	/** 将元素添加进新的数组中末尾中*/
            newElements[len] = e;
          	/** 将旧数组引用地址指向新数组的引用地址*/
            setArray(newElements);
          	/** 返回 */
            return true;
        } finally {
          	/** 释放锁 */
            lock.unlock();
        }
    }
 	
  /** 复制一份新的数组 */
  public static <T> T[] copyOf(T[] original, int newLength) {
        return (T[]) copyOf(original, newLength, original.getClass());
    }
  /** 将旧数组引用地址指向新数组的引用地址 */
  final void setArray(Object[] a) {
        array = a;
    }
}
```



##### SET

```java
	 	/**
     * Replaces the element at the specified position in this list with the
     * specified element.
     *
     * 替换指定下表位置的元素
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     * index：被替换的下标位置
     * element：替换后的值
     */
    public E set(int index, E element) {
      	/** 同样的写操作还是得先获取锁 */
        final ReentrantLock lock = this.lock;
      	/** 尝试获取锁 */
        lock.lock();
        try {
          	/** 获取锁成功后进入这里 */
          	/** 获取数组 */
            Object[] elements = getArray();
          	/** 获取指定位置的元素 */
            E oldValue = get(elements, index);
						/** 如果指定位置元素不为空 */
            if (oldValue != element) {
              	/** 下面就是将原来的数组copy一份新的出来，然后进行值替换，最后将旧数组的对象引用地址指向新的地址 */
                int len = elements.length;
                Object[] newElements = Arrays.copyOf(elements, len);
                newElements[index] = element;
                setArray(newElements);
            } else {
              	/** 这里为了保持可变的规范，其实并没有做什么重要操作 */
                setArray(elements);
            }
          	/** 返回旧值 */
            return oldValue;
        } finally {
          	/** 释放锁 */
            lock.unlock();
        }
    }
```



##### GET

```java
		/** 获取指定位置元素，我们发现并没有加锁操作 */
		public E get(int index) {
        return get(getArray(), index);
    }

		final Object[] getArray() {
        return array;
    }
```



##### 总结

CopyOnWriteArrayList内部还有其他的一些方法，这里就不带大家一一的进行分析了，比较简单，大同小异，可以尝试去进行分析。

CopyOnWrite并发容器用于读多写少的并发场景。比如白名单，黑名单等场景。

不管哪一种技术都有自己的缺点，那么CopyOnWriteArrayList的缺点如下：

**内存占用**

因为在每次写操作的时候都会copy一份数组出来，在写的时候旧的数组对象也还在使用。因此有两个对象存在堆中，占用两份内存。

**数据一致性**

CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。可以思考一下为什么会这样？？？

好了，这次就到这里吧。下一次我们来分析一下文中提到的ReentrantLock。

