# CopyOnWriteArrayList vs ArrayList vs Vector
## 1. COWArrayList
线程安全的ArrayList，实现上所有写操作全部拷贝一份数组并在新数组上操作。操作完成后把底层数组变更为新数组
核心思想是保证读读和读写并发的性能，COW策略保证任何情况下都无需对读操作加锁。如果不使用这种策略接类似于Vector的策略需要对读取也加锁
### 1.1 结构

### 1.2 核心代码
#### 1.2.1 get
```java
    private E get(Object[] a, int index) {
        return (E) a[index];
    }

    /**
     * {@inheritDoc}
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E get(int index) {
        return get(getArray(), index);
    }
```
可见get是不加锁的

#### 1.2.2 set
```java

    /**
     * Replaces the element at the specified position in this list with the
     * specified element.
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E set(int index, E element) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            E oldValue = get(elements, index);

            if (oldValue != element) {
                int len = elements.length;
                Object[] newElements = Arrays.copyOf(elements, len);
                newElements[index] = element;
                setArray(newElements);
            } else {
                // Not quite a no-op; ensures volatile write semantics
                setArray(elements);
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
```
可见set操作需要获取锁，所有写操作在着呢个数组粒度上是互斥的。。
若新老元素不一致，则会cow

#### 1.2.3 add
```java

    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
add操作会复制一个当前数组扩容1的数组并在后面append元素，由于cow每次写都要复制，所以不用像ArrayList那样翻倍扩容。
当然这就造成了add操作的时间复杂度是O(n)【所有写操作都是】

#### 1.2.4 迭代器
```java
static final class COWIterator<E> implements ListIterator<E> {
        /** Snapshot of the array */
        private final Object[] snapshot;
        /** Index of element to be returned by subsequent call to next.  */
        private int cursor;

        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            snapshot = elements;
        }

        public boolean hasNext() {
            return cursor < snapshot.length;
        }

        public boolean hasPrevious() {
            return cursor > 0;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            if (! hasPrevious())
                throw new NoSuchElementException();
            return (E) snapshot[--cursor];
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor-1;
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code remove}
         *         is not supported by this iterator.
         */
        public void remove() {
            throw new UnsupportedOperationException();
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code set}
         *         is not supported by this iterator.
         */
        public void set(E e) {
            throw new UnsupportedOperationException();
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code add}
         *         is not supported by this iterator.
         */
        public void add(E e) {
            throw new UnsupportedOperationException();
        }

        @Override
        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            Object[] elements = snapshot;
            final int size = elements.length;
            for (int i = cursor; i < size; i++) {
                @SuppressWarnings("unchecked") E e = (E) elements[i];
                action.accept(e);
            }
            cursor = size;
        }
    }
```
cow迭代器直接关联了cow的一个快照，因此不会像其他不支持并发的容器那样使用fast-fail机制比对修改数，并且不一致时抛出异常

## 2. ArrayList
可变大小的列表，自动扩容，不支持并发读写
### 2.1 结构
### 2.2 核心方法
#### 2.1 set
```java

    /**
     * Replaces the element at the specified position in this list with
     * the specified element.
     *
     * @param index index of the element to replace
     * @param element element to be stored at the specified position
     * @return the element previously at the specified position
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```

#### 2.2 add
```java

    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            // DEFAULT_CAPACITY == 10
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
        grow(minCapacity);
    }


    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0) // MAX_ARRAY_SIZE == Integer.MAX_VALUE - 8，-8是因为某些vm会需要array中留一下位做header
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```
add时仅当发现容量不够时才扩容（没有像HashMap那种load factory的机制，因为不需要解决hash冲突的劣化效应）
扩容会先扩为当前容量的1.5倍，如果发现扩为1.5倍后超过MAX_ARRAY_SIZE的安全阈值则尽量不超过这个阈值，但是如果add后需要的空间超过阈值则尝试分配Integer.MAX_VALUE

## 3. Vector
### 3.1
### 3.2 核心方法
#### 3.2.1 add
```java

/**
 * Adds the specified component to the end of this vector,
 * increasing its size by one. The capacity of this vector is
 * increased if its size becomes greater than its capacity.
 *
 * <p>This method is identical in functionality to the
 * {@link #add(Object) add(E)}
 * method (which is part of the {@link List} interface).
 *
 * @param   obj   the component to be added
 */
public synchronized void addElement(E obj) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = obj;
}

    /**
     * This implements the unsynchronized semantics of ensureCapacity.
     * Synchronized methods in this class can internally call this
     * method for ensuring capacity without incurring the cost of an
     * extra synchronization.
     *
     * @see #ensureCapacity(int)
     */
    private void ensureCapacityHelper(int minCapacity) {
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
        capacityIncrement : oldCapacity);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
1. 方法粒度加锁
2. grow不是按比例增长的，而是定长，使得add的平摊时间复杂度是线性的

#### 3.2.2 index of
```java

    /**
     * Returns the index of the first occurrence of the specified element in
     * this vector, searching forwards from {@code index}, or returns -1 if
     * the element is not found.
     * More formally, returns the lowest index {@code i} such that
     * <tt>(i&nbsp;&gt;=&nbsp;index&nbsp;&amp;&amp;&nbsp;(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i))))</tt>,
     * or -1 if there is no such index.
     *
     * @param o element to search for
     * @param index index to start searching from
     * @return the index of the first occurrence of the element in
     *         this vector at position {@code index} or later in the vector;
     *         {@code -1} if the element is not found.
     * @throws IndexOutOfBoundsException if the specified index is negative
     * @see     Object#equals(Object)
     */
    public synchronized int indexOf(Object o, int index) {
        if (o == null) {
            for (int i = index ; i < elementCount ; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = index ; i < elementCount ; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```
同样方法粒度加锁

## 总结：
1. COW的核心思想是保证读取性能，读操作无需加锁。代价是写操作必须在容器粒度上加锁并且copy副本再修改。并发度、性能、空间占用都差。因此适用于读多写少的场景
2. 这三种容器都没实现缩容，所以删除大量元素之后如果需要优化内存空间则需要自行迁移
3. 并发读多写少的情况建议使用COW，否则建议使用Vector
