# 面试官：HashMap是怎么实现的？

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200901230324525.png?)
## 前言
HashMap可以说是面试的重中之重，去10家公司面试，8家都会问道，为什么大家都爱用HashMap打开话题？

HashMap是怎么实现的？

1. jdk1.7的HashMap是用数组+链表实现的
2. jdk1.8的HashMap是用数组+链表+红黑树实现的

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200305191229358.PNG?)
HashMap的主干是一个数组，假设我们有3个键值对dnf:1，cf:2，lol:3，每次放的时候会根据key.hash % table.length（对象的hashcode进行一些操作后对数组的长度取余）确定这个键值对应该放在数组的哪个位位置

1 = indexFor(dnf)，我们将键值对放在数组下标为1的位置
![这里写图片描述](https://img-blog.csdn.net/20180405003817602?)    
3 = indexFor(cf)   
![这里写图片描述](https://img-blog.csdn.net/2018040500460914?)
1 = indexFor(lol)，这时发现数组下标为1的位置已经有值了，我们把lol:3放到链表的第一位，将原先的dnf:1用链表的形式放到lol键值对的下面

jdk1.7是头插法
jdk1.8是尾插法
![这里写图片描述](https://img-blog.csdn.net/20180407133946722?)
在获取key为dnf的键值对时，1=hash(dnf)，得到这个键值对在数组下标为1的位置，dnf和lol不相等，和下一个元素比较，相等返回。set和get的过程就是这么简单。先定位到槽的位置（即数组中的位置），再遍历链表找到相同的元素。

由上图可以看出，HashMap在发生hash冲突的时候用的是链地址法，解决hash冲突并不只有这一种方法，常见的有如下四种方法

1. 开放定址法
2. 链地址法
3. 再哈希法
4. 公共溢出区域法。

## JDK1.7源码

几个重要的属性

```java
//初始容量是16，且容量必须是2的倍数
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

//最大容量是2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;

//负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

static final Entry<?,?>[] EMPTY_TABLE = {};

//HashMap的主干是一个Entry数组,在需要的时候进行扩容，长度必须是2的被数
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;

//放置的key-value对的个数
transient int size;

//进行扩容的阈值，值为 capacity * load factor，即容量 * 负载因子
int threshold;

//负载因子
final float loadFactor;
```
这里说一下threshold和loadFactor，threshold = capacity * load factor，即扩容的阈值=数组长度 * 负载因子，如果hashmap中放了16个元素，负载因子为0.75，则扩容阈值为16*0.75=12

1. 负载因子越小，容易扩容，浪费空间，但查找效率高
2. 负载因子越大，不易扩容，对空间的利用更加充分，查找效率低（链表拉长）


存储数据的静态内部类，数组+链表，这里的数组指的就是Entry数组

```java
static class Entry<K,V> implements Map.Entry<K,V> {
	final K key;
	V value;
	Entry<K,V> next;//存储指向下一个Entry的引用，单链表结构
	int hash;//对key的hashcode值进行hash运算后得到的值，存储在Entry，避免重复计算

	Entry(int h, K k, V v, Entry<K,V> n) {
		value = v;
		next = n;
		key = k;
		hash = h;
	}
}
```
**构造函数**

其他都是在此基础上的扩展，主要就是设置初始容量和负载因子，这2个参数前面介绍过了哈。

```java
public HashMap(int initialCapacity, float loadFactor) {
	if (initialCapacity < 0)
		throw new IllegalArgumentException("Illegal initial capacity: " +
				initialCapacity);
	if (initialCapacity > MAXIMUM_CAPACITY)
		initialCapacity = MAXIMUM_CAPACITY;
	if (loadFactor <= 0 || Float.isNaN(loadFactor))
		throw new IllegalArgumentException("Illegal load factor: " +
				loadFactor);

	this.loadFactor = loadFactor;
	threshold = initialCapacity;
	init();
}
```
最重要的知识点来了，对着流程看源码比较好理解

**put方法的执行过程**

1. key为null直接放在table[0]处，对key的hashCode()做hash运算，计算index;
2. 如果节点已经存在就替换old value(保证key的唯一性)，并返回old Value
3.  如果达到扩容的阈值（超过capacity * load factor），并且发生碰撞，就要resize
4. 将元素放到bucket的首位，即头插法

```java
public V put(K key, V value) {
	//hashmap的数组为空
	if (table == EMPTY_TABLE) {
		inflateTable(threshold);
	}
	if (key == null)
		return putForNullKey(value);
	//获取hash值
	int hash = hash(key);
	//找到应该放到table的哪个位置
	int i = indexFor(hash, table.length);
	//遍历table[i]位置的链表，查找相同的key,若找到则使用新的value替换oldValue,并返回oldValue
	for (Entry<K,V> e = table[i]; e != null; e = e.next) {
		Object k;
		//如果key已经存在，将value设置为新的，并返回旧的value值
		if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
			V oldValue = e.value;
			e.value = value;
			e.recordAccess(this);
			return oldValue;
		}
	}

	modCount++;
	//将元素放到table[i]，新的元素总在table[i]位置的第一个元素，原来的元素后移
	addEntry(hash, key, value, i);
	return null;
}
```
为空时，HashMap还没有创建这个数组，有可能用的是默认的16的初始值，还有可能自定义了长度，这时需要把数组长度变为2的最小倍数，并且这个2的倍数大于等于初始容量

```java
private void inflateTable(int toSize) {
	//返回大于或等于最接近2的幂数
	int capacity = roundUpToPowerOf2(toSize);

	threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
	table = new Entry[capacity];
	initHashSeedAsNeeded(capacity);
}
```
若key为null，则将值放在table[0]这个链上

```java
private V putForNullKey(V value) {
	for (Entry<K,V> e = table[0]; e != null; e = e.next) {
		if (e.key == null) {
			V oldValue = e.value;
			e.value = value;
			e.recordAccess(this);
			return oldValue;
		}
	}
	modCount++;
	addEntry(0, null, value, 0);
	return null;
}
```

找到应该放在数组的位置，h & (length-1)这个式子你可以认为hash值对数组长度取余，后面会说到这个式子
```java
static int indexFor(int h, int length) {
	// assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
	return h & (length-1);
}
```
添加元素

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
	// 容量超过阈值，并且发生碰撞时进行扩容
	if ((size >= threshold) && (null != table[bucketIndex])) {
		// 数组扩容为原来的2倍，并将元素复制到新数组上
		resize(2 * table.length);
		// 重新计算hash值，如果不做特殊设置的话，和之前算出来的hash值一样
		hash = (null != key) ? hash(key) : 0;
		bucketIndex = indexFor(hash, table.length);
	}

	createEntry(hash, key, value, bucketIndex);
}
```
将新增加的元素放到table的第一位，并且将其他元素跟在第一个元素后面

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
	Entry<K,V> e = table[bucketIndex];
	table[bucketIndex] = new Entry<>(hash, key, value, e);
	size++;
}
```

容量超过阈值并且发生碰撞，开始扩容

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    //容量已经达到最大
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```

重新计算元素在新的数组中的位置，并进行复制处理，initHashSeedAsNeeded函数默认情况下会一直返回false，即rehash在默认情况下为false
```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    // 遍历数组
    for (Entry<K,V> e : table) {
        // 遍历链表
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```
这个transfer函数挺有意思的，如果你仔细理解它的复制过程，会发现有如下2个特别有意思的地方

1. **原来在oldTable[i]位置的元素，会被放到newTable[i]或者newTable[i+oldTable.length]的位置**
2. **链表在复制的时候会反转**

这2个点需要注意一下，我会在JDK1.8中再次提到这2个点

**get方法的执行过程**

1. key为null直接从table[0]处取，对key的hashCode()做hash运算，计算index;
2. 通过key.equals(k)去查找对应的Entry，接着返回value

```java
public V get(Object key) {
	if (key == null)
		return getForNullKey();
	Entry<K,V> entry = getEntry(key);

	return null == entry ? null : entry.getValue();
}
```
从table[0]初获取key为null的值

```java
private V getForNullKey() {
	if (size == 0) {
		return null;
	}
	for (Entry<K,V> e = table[0]; e != null; e = e.next) {
		if (e.key == null)
			return e.value;
	}
	return null;
}
```
key不为null时

```java
final Entry<K,V> getEntry(Object key) {
	if (size == 0) {
		return null;
	}

	int hash = (key == null) ? 0 : hash(key);
	for (Entry<K,V> e = table[indexFor(hash, table.length)];
		 e != null;
		 e = e.next) {
		Object k;
		if (e.hash == hash &&
			((k = e.key) == key || (key != null && key.equals(k))))
			return e;
	}
	return null;
}
```
## JDK1.8源码
jdk1.8存取key为null的数据并没有进行特判，而是通过将hash值返回为0将其放在table[0]处

**put执行过程**

1. 对key的hashcode()高16位和低16位进行异或运算求出具体的hash值
2. 如果table数组没有初始化，则初始化table数组长度为16
3. 根据hash值计算index，如果没碰撞则直接放到bucket里（bucket可为链表或者红黑树）
4. 如果碰撞导致链表过长，就把链表转为红黑树
5. 如果key已经存在，用new value替换old value，并返回old value
6. 如果超过扩容的阈值则进行扩容，threshold = capacity * load factor
```java
public V put(K key, V value) {
	return putVal(hash(key), key, value, false, true);
}
```

```java
static final int hash(Object key) {
	int h;
	// 对象的hashCode高16位和低16位进行异或操作
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
			   boolean evict) {
	Node<K,V>[] tab; Node<K,V> p; int n, i;
	// 如果HashMap的初始容量没有指定，则为16
	if ((tab = table) == null || (n = tab.length) == 0)
		n = (tab = resize()).length;
	// 用hash值求出bucket的位置
	if ((p = tab[i = (n - 1) & hash]) == null)
		// bucket位置上没有放元素,放置第一个元素
		tab[i] = newNode(hash, key, value, null);
	else {
		// bucket位置上已经有了元素
		Node<K,V> e; K k;
		// 有同名key存在
		if (p.hash == hash &&
			((k = p.key) == key || (key != null && key.equals(k))))
			e = p;
		else if (p instanceof TreeNode)
			// 判断该链为红黑树
			e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
		else {
			// 判断该链为链表
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
				// key相等用新值替换旧值
				e.value = value;
			afterNodeAccess(e);
			return oldValue;
		}
	}
	++modCount;
	// 超过扩容阈值则扩容
	if (++size > threshold)
		resize();
	afterNodeInsertion(evict);
	return null;
}
```
移动的过程和jdk1.7相比变化比较大
jdk1.8和jdk1.7重新获取元素值在新数组中所处的位置的算法发生了变化（实际效果没发生改变）
1. jdk1.7，hash & (length-1)
2. jdk1.8，判断原来hash值要新增的bit位是0还是1。假如是0，放到newTable[i]，否则放到newTable[i+oldTable.length]
```java
final Node<K,V>[] resize() {
	Node<K,V>[] oldTab = table;
	int oldCap = (oldTab == null) ? 0 : oldTab.length;
	int oldThr = threshold;
	int newCap, newThr = 0;
	if (oldCap > 0) {
		// 查过最大值就不再扩充
		if (oldCap >= MAXIMUM_CAPACITY) {
			threshold = Integer.MAX_VALUE;
			return oldTab;
		}
		// 没超过最大值，就扩充为原来的2倍
		else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
				 oldCap >= DEFAULT_INITIAL_CAPACITY)
			newThr = oldThr << 1; // double threshold
	}
	else if (oldThr > 0) // initial capacity was placed in threshold
		newCap = oldThr;
	else {               // zero initial threshold signifies using defaults
		newCap = DEFAULT_INITIAL_CAPACITY;
		newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
	}
	// 重新计算扩容阈值
	if (newThr == 0) {
		float ft = (float)newCap * loadFactor;
		newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
				  (int)ft : Integer.MAX_VALUE);
	}
	threshold = newThr;
	@SuppressWarnings({"rawtypes","unchecked"})
		Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
	table = newTab;
	if (oldTab != null) {
		// 把每个bucket都移动到新的bucket中
		for (int j = 0; j < oldCap; ++j) {
			Node<K,V> e;
			if ((e = oldTab[j]) != null) {
				oldTab[j] = null;
				if (e.next == null)
					newTab[e.hash & (newCap - 1)] = e;
				else if (e instanceof TreeNode)
					((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
				else { // preserve order
					// 链表优化重hash的代码块
					Node<K,V> loHead = null, loTail = null;
					Node<K,V> hiHead = null, hiTail = null;
					Node<K,V> next;
					do {
						next = e.next;
						if ((e.hash & oldCap) == 0) {
							if (loTail == null)
								loHead = e;
							else
								loTail.next = e;
							loTail = e;
						}
						else {
							if (hiTail == null)
								hiHead = e;
							else
								hiTail.next = e;
							hiTail = e;
						}
					} while ((e = next) != null);
					if (loTail != null) {
						loTail.next = null;
						newTab[j] = loHead;
					}
					if (hiTail != null) {
						hiTail.next = null;
						newTab[j + oldCap] = hiHead;
					}
				}
			}
		}
	}
	return newTab;
}
```

**get执行过程**

1. 对key的hashcode()高16位和低16位进行异或运算求出具体的hash值
5. 如果在bucket里的第一个节点直接命中，则直接返回
6. 如果有冲突，通过key.equals(k)去查找对应的Node，并返回value。在树中查找的效率为O(logn)，在链表中查找的效率为O(n)

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

```java
final Node<K,V> getNode(int hash, Object key) {
	Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
	if ((tab = table) != null && (n = tab.length) > 0 &&
		(first = tab[(n - 1) & hash]) != null) {
		if (first.hash == hash && // always check first node
			((k = first.key) == key || (key != null && key.equals(k))))
			return first;
		if ((e = first.next) != null) {
			if (first instanceof TreeNode)
				return ((TreeNode<K,V>)first).getTreeNode(hash, key);
			do {
				if (e.hash == hash &&
					((k = e.key) == key || (key != null && key.equals(k))))
					return e;
			} while ((e = e.next) != null);
		}
	}
	return null;
}
```

## 常见面试题
### HashMap，HashTable，ConcurrentHashMap之间的区别
|       对象        |  key和value是否允许为空  | 是否线程安全 |
| :---------------: | :----------------------: | :----------: |
|      HashMap      |  key和value都允许为null  |      否      |
|     HashTable     | key和value都不允许为null |      是      |
| ConcurrentHashMap | key和value都不允许为null |      是      |
### HashMap在什么条件下扩容
**jdk1.7**

1. 超过扩容的阈值
2. 发生碰撞

**jdk1.8**
1. 超过扩容的阈值
### HashMap的大小为什么是$2^n$
为了通过hash值确定元素在数组中存的位置，我们需要进行如下操作hash%length，当时%操作比较耗时间，所以优化为 hash & (length - 1)

当length为2的n次方时，hash & (length - 1) =hash % length

我们假设数组的长度为15和16，hash码为8和9
| h & (length - 1) | h    | length | index |
| ---------------- | :--- | :----- | :---- |
| 8 & (15 - 1)     | 0100 | 1110   | 0100  |
| 9 & (15 - 1)     | 0101 | 1110   | 0100  |
| 8 & (16 - 1)     | 0100 | 1111   | 0100  |
| 9 & (16 - 1)     | 0101 | 1111   | 0101  |
可以看出数组长度为15的时候，hash码为8和9的元素被放到数组中的同一个位置形成链表，键低了查询效率，当hahs码和15-1(1110)进行&时，最后一位永远是0，这样0001，0011，0101，1001，1011，0111，1101这些位置永远不会被放置元素，这样会导致

1. 空间浪费大
2. 增加了碰撞的几率，减慢查询的效率

当数组长度为$2^n$时，$2^n-1$的所有位都是1，如8-1=7即111，那么进行低位&运算时，值总与原来的hash值相同，降低了碰撞的概率

## JDK1.8发生了哪些变化？
1. 由数组+链表改为数组+链表+红黑树，当链表的长度超过8时，链表变为红黑树。
	**为什么要这么改？**
	我们知道链表的查找效率为O(n)，而红黑树的查找效率为O（logn），查找效率变高了。
	**为什么不直接用红黑树？**
	因为红黑树的查找效率虽然变高了，但是插入效率变低了，如果从一开始就用红黑树并不合适。从概率学的角度选了一个合适的临界值为8
2. 优化了hash算法
3. 计算元素在新数组中位置的算法发生了变化，新算法通过新增位判断oldTable[i]应该放在newTable[i]还是newTable[i+oldTable.length]
4. 头插法改为尾插法，扩容时链表没有发生倒置（避免形成死循环）
## HashMap在高并发下会发生什么问题？
1. 多线程扩容，会让链表形成环，从而造成死循环
2. 多线程put可能导致元素丢失

**jdk1.8中死循环问题已经解决，元素丢失问题还存在**

## 如何避免HashMap在高并发下的问题？
1. 使用ConcurrentHashMap
2. 用Collections.synchronizedMap(hashMap)包装成同步集合