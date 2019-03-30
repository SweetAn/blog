---
title: Hash和HashCode的介绍
date: 2016-08-29 21:01:45
tags: 
- Hash
- HashCode
categories: 
- Java
---

### 1.Hash的作用介绍
> 散列表（Hash table，也叫哈希表），是根据键（Key）而直接访问在内存存储位置的数据结构。也就是说，它通过计算一个关于键值的函数，将所需查询的数据映射到表中一个位置来访问记录，这加快了查找速度。这个映射函数称做散列函数，存放记录的数组称做散列表。
一个通俗的例子是，为了查找电话簿中某人的号码，可以创建一个按照人名首字母顺序排列的表（即建立人名 x到首字母 F(x)的一个函数关系），在首字母为W的表中查找“王”姓的电话号码，显然比直接查找就要快得多。这里使用人名作为关键字，“取首字母”是这个例子中散列函数的函数法则  F()，存放首字母的表对应散列表。关键字和函数法则理论上可以任意确定。

#### 基本概念
1.若关键字为 k，则其值存放在$f(k)$的存储位置上。由此，不需比较便可直接取得所查记录。称这个对应关系 f为散列函数，按这个思想建立的表为散列表。
2.对不同的关键字可能得到同一散列地址，即 $k_1 \neq k_2$，而 $f(k_1)=f(k_2)$，这种现象称为 哈希冲突（英语：Collision）。具有相同函数值的关键字对该散列函数来说称做同义词。综上所述，根据散列函数 $ f(k)$和处理冲突的方法将一组关键字映射到一个有限的连续的地址集（区间）上，并以关键字在地址集中的“像”作为记录在表中的存储位置，这种表便称为散列表，这一映射过程称为散列造表或散列，所得的存储位置称散列地址。
3.若对于关键字集合中的任一个关键字，经散列函数映象到地址集合中任何一个地址的概率是相等的，则称此类散列函数为均匀散列函数（Uniform Hash function），这就使关键字经过散列函数得到一个“随机的地址”，从而减少冲突。

### 2、HashCode的作用

- HashCode是Object的一个方法，hashCode方法返回一个hash code值，且这个方法是为了更好的支持hash表，比如String，Set，HashTable、HashMap等;

### 2.1.  HashCode的特性
（1）HashCode的存在主要是用于查找的快捷性，如Hashtable，HashMap等，HashCode经常用于确定对象的存储地址；
（2）如果两个对象相同， equals方法一定返回true，并且这两个对象的HashCode一定相同；
（3）两个对象的HashCode相同，并不一定表示两个对象就相同，即equals()不一定为true，只能够说明这两个对象在一个散列存储结构中。
（4）如果对象的equals方法被重写，那么对象的HashCode也尽量重写。

> 正确的equals0方法必须满足下列5个条件:
1)自反性。对任意x, x.equals(x)一定返回true。
2)对称性。对任意x和y,如果y.equals(x)返 回true,则x.equals(y)也返 回true。
3)传递性。对任意x、y、z,如果有x.equals(y)返 回ture, y.equals(z)返 回true,则x.equals(z) 一定返回true。
4)一致性。对任意x和y，如果对象中用于等价比较的信息没有改变，那么无论调用
x.equals(y)多少次，返回的结果应该保持-致，要么- -直是true,要么-直是false。
5)对任何不是null的x, x.equals(nu)- 定返 回false。


### 3. Java 中HashCode的作用

> 从Object角度看，JVM每new一个Object，它都会将这个Object丢到一个Hash表中去，这样的话，下次做Object的比较或者取这个对象的时候（读取过程），它会根据对象的HashCode再从Hash表中取这个对象。这样做的目的是提高取对象的效率。若HashCode相同再去调用equal。

#### 3.1 String对象

``` java
    public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
    
    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
```


#### 3.2 Integer对象

```java
    public boolean equals(Object obj) {
        if (obj instanceof Integer) {
            return value == ((Integer)obj).intValue();
        }
        return false;
    }
    
    @Override
    public int hashCode() {
        return Integer.hashCode(value);
    }
```

### 4、在HashMap中put的逻辑

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

```
从上面代码中，在HashMap的put方法调用的过程中，使用 key的hash 值进行重复性的检测


部分资料来自：http://www.importnew.com/18851.html






