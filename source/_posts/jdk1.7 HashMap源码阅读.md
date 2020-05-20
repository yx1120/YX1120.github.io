---
title: jdk1.7 HashMap源码
date: 2020-05-20 
categories: Java
tags: 源码
---

> 1.7 HashMap底层使用的是（Entry）数组+链表实现
>
> 线程不安全，在多个线程进行插入操作时，可能会出现死循环，数据覆盖问题
>
> 1.8 中HashMap底层使用的是（Node）数组+链表+红黑树实现，不会出现死循环，但仍有数据覆盖问题
>
> 需要线程安全的话，建议使用ConcurrentHashMap

<img src='https://pic.downk.cc/item/5ec486b7c2a9a83be5103d16.png'>

### 1）HashMap中的字段

~~~java
	
	//默认初始容量=16，必须为2的幂
    static final int DEFAULT_INITIAL_CAPACITY = 16;
    //映射最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
    //构造函数未指定负载因子时，使用的默认负载因子：0.75
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * 散列表，长度始终是2的幂。
     * 为什么不是向HashTable使用一个素数呢？为了方便后面的扩容操作
     */
    transient Entry<K,V>[] table;

    //hash表中存储的实际元素个数
    transient int size;

    /**
	 * 阈值 = hash表容器capacity * 负载因子loadfactor
	 * size > 阈值，就进行扩容
     */
    int threshold;
   	// 手动指定的负载因子
    final float loadFactor;
    // 此字段用于使HashMap的Collection-view上的迭代器快速失败
    transient int modCount;
    // 默认阈值，可以使用上面的 threshold覆盖该值
    static final int ALTERNATIVE_HASHING_THRESHOLD_DEFAULT = Integer.MAX_VALUE;

 	//如果值为true，则对字符串键执行备用哈希处理，以减少由于哈希码计算能力较弱而导致的冲突发生率。
    transient boolean useAltHashing;
~~~

### 2）节点类Entry

~~~java
static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;

        /**
         * Creates new entry.
         */
        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v;
            next = n;
            key = k;
            hash = h;
        }
    //...
}
~~~

### 2）构造函数

~~~java
	/**
	 * 使用指定的容量和负载因子初始化HashMap
	 */
	public HashMap(int initialCapacity, float loadFactor) {
        //参数有效判断
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

        // Find a power of 2 >= initialCapacity
        int capacity = 1;
        // 确保初始容量是2的幂，比如指定initialCapacity=12，经过这里后，capacity=16
        // 即：找到不小于“指定容量”的最小2次幂
        while (capacity < initialCapacity)
            capacity <<= 1;

        //初始化负载因子、计算阈值
        this.loadFactor = loadFactor;
        threshold = (int)Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
        table = new Entry[capacity];
        
        useAltHashing = sun.misc.VM.isBooted() &&
                (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        init();
    }

    /**
     * 指定初始容量
     * 使用默认的负载因子0.75
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * 无参构造
     * 使用默认的初始容量16 和负载因子0.75
     */
    public HashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
    }	
~~~

### 3）添加元素

> 总体流程：
>
> 1. 计算散列位置，判断是否有该key,如果有，则使用新值覆盖旧值
> 2. 没有的话，进入addEntry()
>    1. 判断当前元素个数是否大于阈值，大于的话
>       - 进行扩容resize()，2倍扩容hash表长
>       - 然后将元素重新映射到新表上
>    2. 小于当前阈值，进入createEntry()，使用头插法插入元素

#### ->1 put()：入口

> 作用：
>
> - 如果key已经存在，则新值取代旧值。
>
> - 不存在则进入addEntry()方法

~~~java
	/**
	 * 添加元素，如果key已经存在，则新值取代旧值
     */
    public V put(K key, V value) {
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        // 计算插入的索引
        int i = indexFor(hash, table.length);
        // 在以table[i]为头结点的链表上寻找是否已经有该key,如果有，用新值替代旧值
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                //返回旧值
                return oldValue;
            }
        }

        modCount++;
        //添加Entry节点
        addEntry(hash, key, value, i);
        return null;
    }

	/**
     * 计算哈希码h的在散列表(Entry<K,V> [] table)上的索引
     * 		h: key的hashCode
     * 		length: table的表长
     */
    static int indexFor(int h, int length) {
        return h & (length-1);
    }

	//计算key的hash值
	final int hash(Object k) {
        int h = 0;
        if (useAltHashing) {
            if (k instanceof String) {
                return sun.misc.Hashing.stringHash32((String) k);
            }
            h = hashSeed;
        }

        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
~~~

#### ->2 addEntry()

> 作用：
>
> - 判断size是否大于阈值，大于则进行扩容resize()，并重新计算key的散列位置
> - 小于则进入createEntry() 使用头插法插入元素

~~~java
/**
 * 添加元素操作-2
 * 		hash: key的hash值
 * 		bucketIndex:插入的散列表索引位置
 */
void addEntry(int hash, K key, V value, int bucketIndex) {
    	//如果当前元素个数 size>= 阈值 ，同时插入的位置已经有元素了--->扩容
        if ((size >= threshold) && (null != table[bucketIndex])) {
            //扩容操作，2倍扩容。扩的是散列表的长度
            resize(2 * table.length);
            //重新计算key的hash值
            hash = (null != key) ? hash(key) : 0;
            //重新计算散列位置
            bucketIndex = indexFor(hash, table.length);
        }
    	//最终的插入操作
        createEntry(hash, key, value, bucketIndex);
    }
~~~

#### ->3 resize()：扩容操作

> 作用：
>
> - 创建一个新的hash表（Entry数组），进入`transfer()`，将原来的hash表中元素散列到新hash表
> - 重新计算阈值

~~~java
/**
 * 扩容操作：
 */
void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
    	// 如果老散列表长度已经=HashMap最大容量【2^30】
    	// -->修改阈值为Integer的最大值【2^31-1】，不再扩容
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
		
    	// 根据新散列表长创建一个新的散列表
        Entry[] newTable = new Entry[newCapacity];
    	
        boolean oldAltHashing = useAltHashing;
        useAltHashing |= sun.misc.VM.isBooted() &&
                (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        boolean rehash = oldAltHashing ^ useAltHashing;
    	//数据转移到新表上
        transfer(newTable, rehash);
        table = newTable;
    	//计算新阈值
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }

 	/**
     * 将当前表中元素散列到新表上
     */
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            //e：每一个链表头
            while(null != e) {
                // 先保留头结点下一元素
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
               	// 计算新的散列位置
                int i = indexFor(e.hash, newCapacity);
                // 取旧表的表头使用头插法再次插入新表
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
~~~

#### ->4 createEntry()：最终的插入操作

> 作用：
>
> - 使用头插法插入元素

~~~java
//采用的是头插法，
void createEntry(int hash, K key, V value, int bucketIndex) {
    	//先保留老头结点，后面会连上
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
}
~~~

<br>

### 4）获取元素

#### ->1 get()：入口

> 作用：
>
> - 如果键为空，则返回空键对应的值或null
> - 不为空，进入getEntry()

~~~java
public V get(Object key) {
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);

        return null == entry ? null : entry.getValue();
    }
~~~

#### --> getEntry()：获取key对应的Entry节点

> 作用：
>
> - 根据key计算散列索引，然后判断该索引对应的链表是否含有key
> - 有则返回key对应的节点，无返回null

~~~java
final Entry<K,V> getEntry(Object key) {
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
~~~

