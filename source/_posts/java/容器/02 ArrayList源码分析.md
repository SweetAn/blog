---
title:  ArrayList源码分析
date: 2016-08-30 20:30:25
tags: 
- 容器
categories: 
- Java
- 容器
---


# ArrayList源码分析
## 一、ArrayList的特点
1.ArrayList是动态数组实现的List，其容量能自动增长
2.ArrayList是非线程安全的，线程安全的考虑Collections.synchronizedList(list)或concurrent并发包下的CopyOnWriteArrayList类
3.ArrayList实现了RandomAccess接口，支持快速随机访问，实际上就是通过下标序号进行快速访问，所以适用于快速访问和修改

```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable

```
4.ArrayList每次插入或删除元素，就要大量地移动元素，插入删除元素的效率低, 所以不适用随机插入和删除
5.ArrayList允许元素为null
6.ArrayList默认容量大小为10，扩容为当前容量的1.5倍，当容量不够时，每次增加元素，都要将原来的元素拷贝到一个新的数组中，非常之耗时，因此建议在事先能确定元素数量的情况下，才使用ArrayList，否则建议使用LinkedList

## 源码分析

1、ArrayList的全局变量
```
    /**
     * Default initial capacity.
     */
    //默认的 容量的大小
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     */
    // 当数据不符合的时候使用的空数组
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    //new ArrayList() 对象时使用的数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    //ArrayList 存放数据的数组
    transient Object[] elementData; // non-private to simplify nested class access

 /**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    // 数组存储最大的容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

```
2、构造函数
```
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```


数组扩展
```
    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        //数组每次扩容按照 每次增加现在容量的0.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```