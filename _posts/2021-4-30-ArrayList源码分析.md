---
layout: post
title: "ArrayList源码分析"
date: 2021-4-30
description: "ArrayList源码分析"
tag: Java基础

---

### ArrayList

##### 数据结构

ArrayList底层是数组实现

##### 特性

有序，且能重复。

##### 优缺点

优点：查找性能高

缺点：插入与删除性能比较低。



### 源码分析

这里就不在介绍ArrayList的使用了。直接进入主题分析源码吧(jdk1.8)。

```java
ArrayList<String> list = new ArrayList<>();
list.add("test");
```

##### 属性与构造器

```java
    // 数组默认大小
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     */
    // 空数组
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    // 这也是一个空数组（用于默认大小的空实列）
    // 这里为什么会说用于默认大小的空实列，在添加元素时，确定数组大小时，会进行判断
    // elementData是否等于DEFAULTCAPACITY_EMPTY_ELEMENTDATA，如果条件成立则使用默认大小（DEFAULT_CAPACITY）
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    // 储存元素的数组
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    // ArrayList包含的元素大小
    private int size;	
		
    // 数组最大容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    // 构造器，创建一个指定大小的数组
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            // 注意这里传入initialCapacity == 0 的情况下，这里是将EMPTY_ELEMENTDATA赋值给elementData
            // 我认为是为了节约内存空间吧，既然传入的是0就没必要给你整内存分配
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
    // 无参构造器
    public ArrayList() {
      	// 将DEFAULTCAPACITY_EMPTY_ELEMENTDATA赋值给elementData
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
	
    // 赋值集合的构造器
    public ArrayList(Collection<? extends E> c) {
        // 转换为数组
        elementData = c.toArray();
      	// 判断传入的集合是否是空实列
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
              	// 复制
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            // 赋值为一个空实列数组
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

##### add

```java
   // 添加元素
   public boolean add(E e) {
      	// 确定数组容量大小
        ensureCapacityInternal(size + 1);  // Increments modCount!!
      	// 将元素添加进数组中
        elementData[size++] = e;
      	// 返回
        return true;
    }
		
   private void ensureCapacityInternal(int minCapacity) {
	// 这里会分为两步骤
      	// 1：calculateCapacity，计算大小
      	// 2：ensureExplicitCapacity是否需要扩容
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
		
   // 1
   private static int calculateCapacity(Object[] elementData, int minCapacity) {
      	// 判断elementData是否为默认大小数组实列
      	// 也就是在创建ArrayList时是否是无参构造
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
          	// 返回大小
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
   // 2
   private void ensureExplicitCapacity(int minCapacity) {
      	// 记录修改次数，在并发的情况下，会对比此值与预期的值是否相等，如果不想等则会快速失败。
      	// protected transient int modCount = 0;
      	// AbstractList类中
        modCount++;

        // overflow-conscious code
      	// 判断是否大于数组的容量，如果大于则扩容。
      	// 还记得开头的列子吧
      	// ArrayList<String> list = new ArrayList<>();
        // list.add("test");
      	// 这里通过无参构造器创建了ArrayList。则数组的默认容量会为10。
      	// 前面提到的calculateCapacity方法，有一句代码比较重要
      	// Math.max(DEFAULT_CAPACITY, minCapacity);当我们添加第一个元素时，则我们的minCapacity = 1，所以此时会返回默认
      	// 大小（DEFAULT_CAPACITY = 10）
      	// 所以此时这里条件不成立
      	// 条件成立的话则进行扩容
        if (minCapacity - elementData.length > 0)
            // 扩容
            grow(minCapacity);
    }
		
    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    // 扩容
    private void grow(int minCapacity) {
        // overflow-conscious code
      	// 旧数组的容量
        int oldCapacity = elementData.length;
      	// 扩容1.5倍
      	// oldCapacity + (oldCapacity >> 1);
      	// 我们拆解一下
      	// oldCapacity >> 1 等价于 oldCapacity/2  ----->右移1位相当于除于2，也就是原来的一半
      	// oldCapacity + 加上原来的一半 等价于 1 + 0.5 = 1.5
      	// 这里为什么是扩容的倍数是1.5，后面再说，先把主流程给整通顺
        int newCapacity = oldCapacity + (oldCapacity >> 1);
      	// 这里为什么会这样判断？？？
      	// 我的理解可能在并发的情况下，其他线程已经扩容了
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
      	
      	// 判断扩容后的容量是否大于数组最大容量
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
      	// 将数据copy到新的数组
      	// elementData：数组数据
      	// newCapacity：新数组的大小
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
		
   private static int hugeCapacity(int minCapacity) {
      	// 小于则数组越界
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
      	// 如果大于数组最大容量则返回Integer最大值
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
    // copy
    public static <T> T[] copyOf(T[] original, int newLength) {
      	// original：源数据
      	// newLength：新数组的长度
      	// original.getClass()：源数组class
        return (T[]) copyOf(original, newLength, original.getClass());
    }
		
   // 具体copy的细节
   public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
      	// 创建一个新的数组
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
      	// 通过System.arraycopy进行copy
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```

至此ArrayList添加源码已经分析完了，是不是很简单。前面提到过一个遗漏问题，扩容的时候为什么是扩容到1.5倍？？？

总结就两点吧

1. 扩容容量不能太小，防止频繁扩容，频繁申请内存空间 + 数组频繁复制
2. 扩容容量不能太大，需要充分利用空间，避免浪费过多空间；

总归就是作者经过千锤百炼得到的这个数字是最优的。



##### remove

接下来我们分析一下remove方法

```java
    /**
     * Removes the element at the specified position in this list.
     * Shifts any subsequent elements to the left (subtracts one from their
     * indices).
     *
     * @param index the index of the element to be removed
     * @return the element that was removed from the list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    // 删除列表指定位置的元素
    public E remove(int index) {
      	// 检查指定删除位置是否越界
        rangeCheck(index);
			  // 修改次数+1
        modCount++;
      	// 获取指定位置元素
        E oldValue = elementData(index);
				// 计算需要移动元素的个数
        int numMoved = size - index - 1;
        if (numMoved > 0)
            // 左移动
            // 这也是为什么ArrayList删除元素为什么效率很低的原因，
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
      	// 将最后一位元素设置为null，因为元素向前移动了一位。
      	// 方便gc更好的回收
        elementData[--size] = null; // clear to let GC do its work
	// 返回旧值
        return oldValue;	
    }

    /**
     * @param      src      源数组.
     * @param      srcPos   源数组起始位置
     * @param      dest     目标数组
     * @param      destPos  目标数组起始位置
     * @param      length   移动数量
     */
     public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);

```







