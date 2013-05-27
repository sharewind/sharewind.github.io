---
layout: post
title: "LinkedHashMap"
date: 2013-03-18 01:04
comments: true
categories: Java
---

LinkedHashMap 是 Map 接口的哈希表和链接列表实现，具有可预知的迭代顺序。此实现与 HashMap 的不同之处在于，后者维护着一个运行于所有条目的双重链接列表。此链接列表定义了迭代顺序，该迭代顺序通常就是将键插入到映射中的顺序（插入顺序）。  
注意，如果在映射中重新插入 键，则插入顺序不受影响。

1.LinkedHashMap数据成员

    /**
     * The head of the doubly linked list.
     */
	// 双向链接的头指针
    private transient Entry<K,V> header;

    /**
     * The iteration ordering method for this linked hash map: <tt>true</tt>
     * for access-order, <tt>false</tt> for insertion-order.
     *
     * @serial
     */
	// 迭代的排序方式：设置为true则按访问顺序，否则按插入顺序
    private final boolean accessOrder;



2.LinkedHashMap构造方法

    /**
     * Constructs an empty <tt>LinkedHashMap</tt> instance with the
     * specified initial capacity, load factor and ordering mode.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @param  accessOrder     the ordering mode - <tt>true</tt> for
     *         access-order, <tt>false</tt> for insertion-order
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }

    /**
     * Called by superclass constructors and pseudoconstructors (clone,
     * readObject) before any entries are inserted into the map.  Initializes
     * the chain.
     */
	// 被父类HashMap的构造函数调用
    void init() {
		// 初始化双向链接的头指针
        header = new Entry<K,V>(-1, null, null, null);
        header.before = header.after = header;
    }

3.LinkedHashMap的Entry  

添加了before,after 两个引用，用来构造双向链表

    /**
     * LinkedHashMap entry.
     */
    private static class Entry<K,V> extends HashMap.Entry<K,V> {
        // These fields comprise the doubly linked list used for iteration.
        Entry<K,V> before, after;

        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
            super(hash, key, value, next);
        }

        /**
         * Removes this entry from the linked list.
         */
		// 从双向链表中移除
        private void remove() {
            before.after = after;
            after.before = before;
        }

        /**
         * Inserts this entry before the specified existing entry in the list.
         */
        private void addBefore(Entry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }

        /**
         * This method is invoked by the superclass whenever the value
         * of a pre-existing entry is read by Map.get or modified by Map.set.
         * If the enclosing Map is access-ordered, it moves the entry
         * to the end of the list; otherwise, it does nothing.
         */
		// Override了父类的recordAccess方法，用来记录访问顺序
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
				// 添加到链表的头部之前（即末尾）
				// put 方法修改会引起顺序的变更
                addBefore(lm.header);
            }
        }

        void recordRemoval(HashMap<K,V> m) {
            remove();
        }
    }

addEntry方法

    /**
     * This override alters behavior of superclass put method. It causes newly
     * allocated entry to get inserted at the end of the linked list and
     * removes the eldest entry if appropriate.
     */
    void addEntry(int hash, K key, V value, int bucketIndex) {
        createEntry(hash, key, value, bucketIndex);

        // Remove eldest entry if instructed, else grow capacity if appropriate
		// 根据方法判断是否移除最老的元素，如果返回true，可以实现一个简单的LRUCache
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        } else {
			// 大小超过threshold，进行rehash 扩容
            if (size >= threshold)
                resize(2 * table.length);
        }
    }

    /**
     * This override differs from addEntry in that it doesn't resize the
     * table or remove the eldest entry.
     */
    void createEntry(int hash, K key, V value, int bucketIndex) {
        HashMap.Entry<K,V> old = table[bucketIndex];
        Entry<K,V> e = new Entry<K,V>(hash, key, value, old);
        table[bucketIndex] = e;
		// 创建一个entry，并添加到双向链表的末尾
        e.addBefore(header);
        size++;
    }

    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }


可以重写 **removeEldestEntry(Map.Entry)** 方法来实施策略，以便在将新映射关系添加到映射时自动移除旧的映射关系。   
示例用法：此重写允许映射增加到 100 个条目，然后每次添加新条目时删除最旧的条目，始终维持 100 个条目的稳定状态。 

     private static final int MAX_ENTRIES = 100;

     protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > MAX_ENTRIES;
     }


transfer方法


    /**
     * Transfers all entries to new table array.  This method is called
     * by superclass resize.  It is overridden for performance, as it is
     * faster to iterate using our linked list.
     */
    void transfer(HashMap.Entry[] newTable) {
		// 由于记录了双向链表，遍历所有entry变得更加简单
        int newCapacity = newTable.length;
        for (Entry<K,V> e = header.after; e != header; e = e.after) {
            int index = indexFor(e.hash, newCapacity);
            e.next = newTable[index];
            newTable[index] = e;
        }
    }


迭代器的实现 

    private abstract class LinkedHashIterator<T> implements Iterator<T> {
		// 第一次next元素 是 header.after
        Entry<K,V> nextEntry    = header.after;
        Entry<K,V> lastReturned = null;

        /**
         * The modCount value that the iterator believes that the backing
         * List should have.  If this expectation is violated, the iterator
         * has detected concurrent modification.
         */
        int expectedModCount = modCount;

        public boolean hasNext() {
            return nextEntry != header;
        }

        public void remove() {
            if (lastReturned == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            LinkedHashMap.this.remove(lastReturned.key);
            lastReturned = null;
            expectedModCount = modCount;
        }

        Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (nextEntry == header)
                throw new NoSuchElementException();

			// 通过迭代链表的指针进行遍历
            Entry<K,V> e = lastReturned = nextEntry;
            nextEntry = e.after;
            return e;
        }
    }

    private class EntryIterator extends LinkedHashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() { return nextEntry(); }
    }






