---
layout: post
title: "Java基础：HashMap"
date: 2013-03-05 10:04
comments: true
categories: Java
---

<!-- HashMap/ConcurrentHashMap/TreeMap/ConcurrentSkipListMap -->

基于哈希表的 Map 接口的实现。此实现提供所有可选的映射操作，并允许使用 null 值和 null 键。（除了非同步和允许使用 null 之外，HashMap 类与 Hashtable 大致相同。）


**HashMap的数据成员**

    /**
     * The default initial capacity - MUST be a power of two.
     */
	//  默认的初始容量大小，必须为2的整数倍
    static final int DEFAULT_INITIAL_CAPACITY = 16;

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
	// 最大容量大小
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     */
	// 默认的加载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The table, resized as necessary. Length MUST Always be a power of two.
     */
	// 用来存储entry项的数组（即哈希桶）
    transient Entry[] table;

    /**
     * The number of key-value mappings contained in this map.
     */
	// 存放的entry个数
    transient int size;

    /**
     * The next size value at which to resize (capacity * load factor).
     * @serial
     */
	// 阈值 = 容器 * 加载因子，达到该值时进行resize扩容操作
    int threshold;

    /**
     * The load factor for the hash table.
     *
     * @serial
     */
	// 加载因子
    final float loadFactor;

    /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     */
	// 累计HashMap结构修改的次数，用来迭代器在判断并发修改时快速失败
    transient int modCount;


**HashMap的默认构造函数**
	
	public HashMap() {
		// 采用默认值进行构造
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        threshold = (int)(DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR);
        table = new Entry[DEFAULT_INITIAL_CAPACITY];
        init();
    }

	// 用来给子类在构造后进行拓展的方法
    void init() {
    }

**hash方法**

    /**
     * Applies a supplemental hash function to a given hashCode, which
     * defends against poor quality hash functions.  This is critical
     * because HashMap uses power-of-two length hash tables, that
     * otherwise encounter collisions for hashCodes that do not differ
     * in lower bits. Note: Null keys always map to hash 0, thus index 0.
     */
    static int hash(int h) {
        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }

    /**
     * Returns index for hash code h.
     */
	// 用来定位把key 放到哪个hash桶中
    static int indexFor(int h, int length) {
		// 等价于 h % (length -1) 
        return h & (length-1);
    }


**put方法**

    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
		// key 为null
        if (key == null)
            return putForNullKey(value);

		// 获取key的hash值
        int hash = hash(key.hashCode());
		// 根据hash值计算当前key 对应的hash桶index
        int i = indexFor(hash, table.length);

		// 遍历hash桶的链表
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
			
			// 判断hash值相等，并且 (是同一个key 或 key.equals(k) ）
			// 这也就是为什么把对象放进HashMap时，需要实现 hashCode 与 equals方法
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
				// 该key已存在，则更新value，并返回记录oldValue
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);// 记录该项被访问过
                return oldValue;
            }
        }
		
		// 修改数加1
        modCount++; // modCount不是volatile修饰的，且自增操作无法保证线程安全！！！

		// key不存在Map中，则添加该项
        addEntry(hash, key, value, i);
        return null;
    }

    /**
     * Offloaded version of put for null keys
     */
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
	

**addEntry方法**
	
	// 添加一个Entry
    void addEntry(int hash, K key, V value, int bucketIndex) {
		// 获取相应的hash桶
        Entry<K,V> e = table[bucketIndex];
		// 修改hash桶的head头指针为新建的Entry
		// 旧的head则作为构造函数传入Entry,新建Entry的next将指向旧的head.
        table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
        
		if (size++ >= threshold)
			// 如果大小大于或等于阈值，则进行resize重新hash操作
            resize(2 * table.length);// 增长为原来大小的2倍

		// 如果恶意制造大量相同hash桶index的值，则会将HashMap退化为链表
		// 从而产生HashMap碰撞攻击
    }


**Entry相当于链表中的一个节点（Node）**  
Entry类的成员与构建函数:

	    static class Entry<K,V> implements Map.Entry<K,V> {
	        final K key;
	        V value;
	        Entry<K,V> next;
	        final int hash;

	        /**
	         * Creates new entry.
	         */
	        Entry(int h, K k, V v, Entry<K,V> n) {
	            value = v;
	            next = n;
	            key = k;
	            hash = h;
	        }

**resize方法**
	
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
		// 如果已经达到最大容量上限，则调整threshold为整数的最大值，然后返回
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

		// 创建一个新容器大小的hash桶
        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable);
        table = newTable;
		// 重新计算阈值
        threshold = (int)(newCapacity * loadFactor);
    }

    /**
     * Transfers all entries from current table to newTable.
     */
	// 迁移所有的旧hash桶的entry到新的hash桶内
    void transfer(Entry[] newTable) {
        Entry[] src = table;
        int newCapacity = newTable.length;
		// 遍历每个hash桶
        for (int j = 0; j < src.length; j++) {
            Entry<K,V> e = src[j];
			// hash链表不为空
            if (e != null) {
                src[j] = null;// 将src的hash桶的头指针置空

                do {// 遍历hash链表

                    Entry<K,V> next = e.next;
					// 计算在新hash桶中的桶索引
                    int i = indexFor(e.hash, newCapacity);
					// 与addEntry一样，修改Entry的next
                    e.next = newTable[i];
					// 使hash桶的头指针指向entry
                    newTable[i] = e;
                    
					e = next;// 遍历下一项
                } while (e != null);
            }
        }
    }

**迭代器的实现**

    public Set<Map.Entry<K,V>> entrySet() {
        return entrySet0();
    }

    private Set<Map.Entry<K,V>> entrySet0() {
        Set<Map.Entry<K,V>> es = entrySet;
        return es != null ? es : (entrySet = new EntrySet());
    }

    private final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public Iterator<Map.Entry<K,V>> iterator() {
            return newEntryIterator();
        }
        public boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<K,V> e = (Map.Entry<K,V>) o;
            Entry<K,V> candidate = getEntry(e.getKey());
            return candidate != null && candidate.equals(e);
        }
        public boolean remove(Object o) {
            return removeMapping(o) != null;
        }
        public int size() {
            return size;
        }
        public void clear() {
            HashMap.this.clear();
        }
    }

    Iterator<Map.Entry<K,V>> newEntryIterator()   {
        return new EntryIterator();
    }

    private final class EntryIterator extends HashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() {
            return nextEntry();
        }
    }


    private abstract class HashIterator<E> implements Iterator<E> {
        Entry<K,V> next;        // next entry to return
        int expectedModCount;   // For fast-fail
        int index;              // current slot
        Entry<K,V> current;     // current entry

        HashIterator() {
			// 初始化为保存当前的modCount
            expectedModCount = modCount;
            if (size > 0) { // advance to first entry
                Entry[] t = table;
				// 遍历hash桶，获取第一个不为空的hash桶
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Entry<K,V> nextEntry() {
			// 如果expectedModCount 与 modCount 不一致，则说明其它地方对Map进行了修改
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();

			// 如果当前项的next为空，则找到一个不为空的项，如果没有则为null
            if ((next = e.next) == null) {
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
			
			// 返回当前项
            current = e;
            return e;
        }

        public void remove() {
            if (current == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Object k = current.key;
            current = null;
            HashMap.this.removeEntryForKey(k);
            expectedModCount = modCount;
        }

    }


#### 注意事项 
1. HashMap不是线程安全的。
2. 迭代器的快速失败行为不能得到保证。  
一般来说，存在非同步的并发修改时，不可能作出任何坚决的保证。快速失败迭代器尽最大努力抛出 ConcurrentModificationException。因此，**编写依赖于此异常的程序的做法是错误的**，正确做法是：迭代器的快速失败行为应该仅用于检测程序错误。  
3. 如果你有一个已知大小的HashMap,初始化时最好带上容量参数，以避免频繁进行resize操作。
4. HashMap碰撞拒绝服务漏洞
Apache的方案是在Tomcat中增加一个新的选项maxParameterCount，用来限制单个请求中的最大参数量。参数默认值设为10000，确保既不会对应用程序造成影响（对多数应用来说已经足够），也足以减轻DoS攻击的压力。 


####参考资料

- [Apache曝HashTable碰撞拒绝服务漏洞，Java、PHP、Asp.Net及v8引擎等都受影响](http://www.iteye.com/news/23859)



