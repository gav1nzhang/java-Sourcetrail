

```java

package java.util;

import java.util.function.Consumer;
import java.util.function.Predicate;
import java.util.function.UnaryOperator;
import jdk.internal.access.SharedSecrets;

public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    /**
     * 此处用来放一些系统Utils方法，如System.arrayCopy
     * 
	 * void arraycopy(Object src, int srcPos, Object dest, int destPos, int    		 * length)
     * src 源数组，起始位置，目标数组，目标数组的起始位置，复制元素的数量
     * 至于分析源码，此方法底层用c++写的，jvm_arrayCopy里也只是说明了简单的检测，而非真正
     * 的复制代码 pd_conjoint_oops_atomic(真正的复制方法)
     *
     * 
     * fail-fast: 一个系统通用设计思想，并非java独有机制，检测到可能发生错误，就立马抛出异常 
     * https://www.cnblogs.com/54chensongxia/p/12470446.html 详见介绍
     * inheritDoc: java注释标记，指示子类或实现类继承父类/接口的注释，会自动复制 保证了一致性
     * 
     * modCount != expectedModCount 并发修改
     */
    
    /**
     * 序列化校验的版本号，https://stackoverflow.com/questions/285793/what-is-a-serialversionuid-and-why-should-i-use-it
     */
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * 定义初始容量
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 空数组用于分享空实例？ 初始化能用到
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 默认大小的空实例分享空数组，区分EMPTY_ELEMENTDATA以知道添加元素的时候会膨胀多少
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 这个集合缓冲区存储列表的元素.列表的容量是集合缓冲区的长度。
     * 当第一个元素被添加，elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA将会拓展到			 * DEFAULT_CAPACITY.
     */
    transient Object[] elementData; // 非私有，简化嵌套类访问

    /**
     * 列表长度 
     *
     * @serial
     */
    private int size;

    /**
 	* 初始化ArrayList
 	* @params 参数 initalCapacity
 	* @throws 小于0时抛出
 	*/
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * 构造空函数使用预先定义的空实例
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * 使用其他Collection集合框架来构建ArrayList，并判断size是否为0，然后进行CopyOf
     * @param 集合框架中
     * @throws 如果为null，替换空集合
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // defend against c.toArray (incorrectly) not returning Object[]
            // (see e.g. https://bugs.openjdk.java.net/browse/JDK-6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

    /**
     * 修正当前ArrayList容量，调整当前List大小，最小化实例当前List
     * 当size小于元素长度，size为0时 元素替换为空，否则以size CopyOf
     */
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }

    /**
     * 如果必要的话，提升当前ArrayList实例的容量, 确保 至少容纳minCapacity的数量
     *
     * @param minCapacity 所需最小容量
     * 如果min 大于 当前元素长度 且 不会空 且 不小于 10 跳入grow，开始增长
     */
    public void ensureCapacity(int minCapacity) {
        if (minCapacity > elementData.length
            && !(elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
                 && minCapacity <= DEFAULT_CAPACITY)) {
            modCount++;
            grow(minCapacity);
        }
    }

    /**
     * 最长度为0x7fffffff (2^31 - 1) - 8 
	 * https://stackoverflow.com/questions/35756277/why-the-maximum-array-size-of-arraylist-is-integer-max-value-8
	 * 对象头不能超过8个字节，所以-8
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * 提升容量，确保至少能增长到minCapacity
     *
     * @param minCapacity 所需最小容量
     * @throws OutOfMemoryError OOM，超出可以拓展的栈 - jvm
     */
    private Object[] grow(int minCapacity) {
        return elementData = Arrays.copyOf(elementData,
                                           newCapacity(minCapacity));
    }
	
    /**
     * 无参数时，增长当前size+1
     */
    private Object[] grow() {
        return grow(size + 1);
    }

    /**
     * 返回一个容量，至少跟给的最小 minCapacity 一样大
     * 如果足够(suffices)， 提升至少50%当前的数量 (位运算向右移一位 0100 -> 0010 从4变2)
     * 不会比列表最长长度还长(MAX_ARRAY_SIZE)
     *
     * @param minCapacity the desired minimum capacity
     * @throws OutOfMemoryError if minCapacity is less than zero
     * old 原先列表元素长度
     * new 1.5倍old
     * 如果newCapacity - minCapacity <= 0，且长度为0，返回Default（10），如果长度小于0，抛出
     * 正常流程，如果new - 最长长度 <=0 说明正常 返回new，否则huge容量
     */
    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity <= 0) {
            if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
                return Math.max(DEFAULT_CAPACITY, minCapacity);
            if (minCapacity < 0) // overflow
                throw new OutOfMemoryError();
            return minCapacity;
        }
        return (newCapacity - MAX_ARRAY_SIZE <= 0)
            ? newCapacity
            : hugeCapacity(minCapacity);
    }

    /**
     * 增加容量
     * @params minCapacity 小于0 抛出
     * 看容量是否大于MAX_ARRAY_SIZE, 大于置为Integer.MAX_VALUE(真实的 最长长度，对象头都不要了)
     * 
     */
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE)
            ? Integer.MAX_VALUE
            : MAX_ARRAY_SIZE;
    }

    /**
     * 返回当前List元素数量
     *
     * @return the number of elements in this list
     */
    public int size() {
        return size;
    }

    /**
     * 返回 True/False，以当前列表容量是否为0作为判断
     *
     * @return {@code true} if this list contains no elements
     */
    public boolean isEmpty() {
        return size == 0;
    }

    /**
     * 如果列表存在需要寻找的值或有多个需要寻找的值，返回true
     */
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    /**
     * 返回在列表第一个出现的元素的索引值，不存在返回-1，
     * 准确说，反应最低位(第一次出现)的index，不存在返回-1
     */
    public int indexOf(Object o) {
        return indexOfRange(o, 0, size);
    }
    
	/**
     * param o 元素 start 起始位 end 终止位
     * 如果元素为空，那么遍历数组缓冲区，发现第一个为null的返回下标
	 * equals若未重写，则判断两个对象是否引用指向同一个对象，或者说 检查对象是否指向内存的同一个地址
     */
    int indexOfRange(Object o, int start, int end) {
        Object[] es = elementData;
        if (o == null) {
            for (int i = start; i < end; i++) {
                if (es[i] == null) {
                    return i;
                }
            }
        } else {
            for (int i = start; i < end; i++) {
                if (o.equals(es[i])) {
                    return i;
                }
            }
        }
        return -1;
    }

    /**
     * Returns the index of the last occurrence of the specified element
     * in this list, or -1 if this list does not contain the element.
     * More formally, returns the highest index {@code i} such that
     * {@code Objects.equals(o, get(i))},
     * or -1 if there is no such index.
     */
    public int lastIndexOf(Object o) {
        return lastIndexOfRange(o, 0, size);
    }
	
    /**
     * 与indexOfRange基本相同，只是倒序查找
     * 寻找最高位的element
     */
    int lastIndexOfRange(Object o, int start, int end) {
        Object[] es = elementData;
        if (o == null) {
            for (int i = end - 1; i >= start; i--) {
                if (es[i] == null) {
                    return i;
                }
            }
        } else {
            for (int i = end - 1; i >= start; i--) {
                if (o.equals(es[i])) {
                    return i;
                }
            }
        }
        return -1;
    }

    /**
     * 返回一个浅克隆的ArrayList实例
     * 元素本身不会被复制
     * @return ArrayList实例的克隆
     */
    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }

    /**
     * 返回一个数组，里面包含了这个列表所有元素，并且以正确顺序排列(从第一个到最后一个)
     *
     * 这个返回的数组是"安全的"，列表不会对它引用进行维护(就是生成一个新数组)
     * 调用方可以随意修改返回的数组
     * 
     * 这个方法是作为 基于数组 和 基于列表的一个 桥梁?
     *
     * @return an array containing all of the elements in this list in
     *         proper sequence
     */
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }

    /**
     * 返回一个数组包含了以正常顺序的列表全部元素(从第一个到最后一个)
     * 返回数组的运行时类型是 指定数组的运行时类型
     * 否则，将分配一个具有指定运行时类型 以及 此列表大小的数组
     *
     * 如果列表适合指定列表，并腾出的多余空间(比如数组元素比列表更多)，
     * 那么立即，数组跟随着列表最后一个元素后，都置为null(如果调用方知道list没包含null元素，这是很有用的对于确定list长度)
     *
     * @param a 这个列表将把所有list的元素存储起来, 如果它足够大，否则，分配一个相同运行时类型的新数组
     * @return 一个数组，包含了list的元素
     * @throws ArrayStoreException 如果指定数组的运行时类型 不是 list每个元素的运行时类型的超类
     *         例如 ArrayList LinkedList -> List
     * @throws NullPointerException 抛出空指针 如果指定数组为null
     * @SuppressWarnings 注解可以用于抑制编译器产生的警告信息，包括但不限于未检查的转换、未检查的方法调用、未检查的参数、未检查的返回值类型、非法注释、过时的方法等。
     * systeam.arrayCopy(留意一下 很重要)
     */
    @SuppressWarnings("unchecked")
    public <T> T[] toArray(T[] a) {
        if (a.length < size)
            // Make a new array of a's runtime type, but my contents:
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }

    // Positional Access Operations
    // 位置访问操作

    @SuppressWarnings("unchecked")
    E elementData(int index) {
        return (E) elementData[index];
    }

    @SuppressWarnings("unchecked")
    static <E> E elementAt(Object[] es, int index) {
        return (E) es[index];
    }

    /**
     * 返回列表中元素的指定位置
     *
     * @param  index 要返回元素的index
     * @return 返回列表中元素的指定位置
     * @throws IndexOutOfBoundsException {@inheritDoc} 越界报错
     */
    public E get(int index) {
        Objects.checkIndex(index, size);
        return elementData(index);
    }

    /**
     * 以指定元素 替换当前列表指定位置元素的值
     *
     * @param index 要替换元素的index
     * @param element 元素被存储指定位置
     * @return 之前在指定位置元素的值
     * @throws IndexOutOfBoundsException {@inheritDoc} 越界报错
     */
    public E set(int index, E element) {
        Objects.checkIndex(index, size);
        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }

    /**
     * 则个帮助方法 分离出来add(E),以保持字节码大小 小于 35 (-XX:MaxInlineSize 默认值)
     * 在c1编译循环中调用add(E) 有用(待补充)
     * 如果s == 总长度 调grow 否则 当前s位置为e 总长度+1
     */
    private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length)
            elementData = grow();
        elementData[s] = e;
        size = s + 1;
    }

    /**
     * 在列表尾部添加一个指定元素
     *  
     * modCount AbstractList中有写 结构修改次数
     * @param e 被添加到列表的元素
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        modCount++;
        add(e, elementData, size);
        return true;
    }

    /**
     * 在列表的指定位置插入指定元素，将当前位于该位置的元素(如果有)和任意的后续元素向右移动
     * 将元素添加到索引内
     * 
     * 判断range范围是否越界 -> 增加结构修改次数 ->定义 参数 是否grow 然后copy
     * 
     * @param index 指定元素在列表中要被插入的位置
     * @param element 被插入的元素
     * @throws IndexOutOfBoundsException {@inheritDoc} 越界
     */
    public void add(int index, E element) {
        rangeCheckForAdd(index);
        modCount++;
        final int s;
        Object[] elementData;
        if ((s = size) == (elementData = this.elementData).length)
            elementData = grow();
        System.arraycopy(elementData, index,
                         elementData, index + 1,
                         s - index);
        elementData[index] = element;
        size = s + 1;
    }

    /**
     * 移除列表中指定位置上的元素 移动后续元素向左(索引中减去一个)
     * 
     * objects 检查索引是否越界 未检查suppressWarning在前面
     *
     * @param index 被溢出的元素的位置
     * @return 被移除的元素
     * @throws IndexOutOfBoundsException {@inheritDoc} 越界
     */
    public E remove(int index) {
        Objects.checkIndex(index, size);
        final Object[] es = elementData;

        @SuppressWarnings("unchecked") E oldValue = (E) es[index];
        fastRemove(es, index);

        return oldValue;
    }

    /**
     * 重写equals
     * 列表相同 true
     * equals对象不为List类 false
     * ArrayList可以被子类化并赋予任意行为，但可以处理常见情况（虽然不知道什么意思）
     * 校验期待的修改次数有无更改
     * {@inheritDoc}
     */
    public boolean equals(Object o) {
        if (o == this) {
            return true;
        }

        if (!(o instanceof List)) {
            return false;
        }

        final int expectedModCount = modCount;
        // ArrayList can be subclassed and given arbitrary behavior, but we can
        // still deal with the common case where o is ArrayList precisely
        boolean equal = (o.getClass() == ArrayList.class)
            ? equalsArrayList((ArrayList<?>) o)
            : equalsRange((List<?>) o, 0, size);

        checkForComodification(expectedModCount);
        return equal;
    }
	
    /**
     * 与List进行范围比较equals 
     * 首先看to 是否越界 否则抛出异常(但没具体报错?)
     * 然后进行迭代器，去判断是否有下一个，或者是否当前迭代器是否相同 否则返回false
     */
    boolean equalsRange(List<?> other, int from, int to) {
        final Object[] es = elementData;
        if (to > es.length) {
            throw new ConcurrentModificationException();
        }
        var oit = other.iterator();
        for (; from < to; from++) {
            if (!oit.hasNext() || !Objects.equals(es[from], oit.next())) {
                return false;
            }
        }
        return !oit.hasNext();
    }

    
    /**
     * 比较ArrayList列表
     * 记录modCount，size 
     * 首先比较size 然后比较大小是否越界，然后一个个去对比元素 false跳出
     * 最后再check一下是否有改动
     */
    private boolean equalsArrayList(ArrayList<?> other) {
        final int otherModCount = other.modCount;
        final int s = size;
        boolean equal;
        if (equal = (s == other.size)) {
            final Object[] otherEs = other.elementData;
            final Object[] es = elementData;
            if (s > es.length || s > otherEs.length) {
                throw new ConcurrentModificationException();
            }
            for (int i = 0; i < s; i++) {
                if (!Objects.equals(es[i], otherEs[i])) {
                    equal = false;
                    break;
                }
            }
        }
        other.checkForComodification(otherModCount);
        return equal;
    }
	
    /**
     * 检查是否有改动，改动数不一样 抛出异常 
     * Constructs a ConcurrentModificationException with no detail message..(报错的备注)
     */
    private void checkForComodification(final int expectedModCount) {
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    /**
     * 重写hashCode 那么注意 hashcode相同 一定相同，hashcode不相同 一定不相同
     * equals不必然
     * {@inheritDoc}
     */
    public int hashCode() {
        int expectedModCount = modCount;
        int hash = hashCodeRange(0, size);
        checkForComodification(expectedModCount);
        return hash;
    }
	
    /**
     * hashCode范围 从form 到 to
     * 首先还是比长度抛出异常，后来每次倍乘31 + 元素e的hashCode
     */
    int hashCodeRange(int from, int to) {
        final Object[] es = elementData;
        if (to > es.length) {
            throw new ConcurrentModificationException();
        }
        int hashCode = 1;
        for (int i = from; i < to; i++) {
            Object e = es[i];
            hashCode = 31 * hashCode + (e == null ? 0 : e.hashCode());
        }
        return hashCode;
    }

    /**
     * 移除列表中的指定元素(如果存在，则第一个)，如果列表不包含元素，那么不改变，移除最低位(最前的)
     * {@code i} such that
     * {@code Objects.equals(o, get(i))}
     * 如果存在元素，返回true(如果列表因调用更改，也返回true)
     * 如果为null，找有没有为null的，返回这个found(闭包?),包含的时候，移除最低位，返回true
     *
     * @param o 如果存在，移除列表中元素
     * @return {@code true} 列表是否包含要移除的元素
     */
    public boolean remove(Object o) {
        final Object[] es = elementData;
        final int size = this.size;
        int i = 0;
        found: {
            if (o == null) {
                for (; i < size; i++)
                    if (es[i] == null)
                        break found;
            } else {
                for (; i < size; i++)
                    if (o.equals(es[i]))
                        break found;
            }
            return false;
        }
        fastRemove(es, i);
        return true;
    }

    /**
     * 结构修改次数+1，预定义-1后的size，找到指定位置i，进行copy，将为之后的元素移前面
     * 最后一个元素置为null 避免空引用
     */
    private void fastRemove(Object[] es, int i) {
        modCount++;
        final int newSize;
        if ((newSize = size - 1) > i)
            System.arraycopy(es, i + 1, es, i, newSize - i);
        es[size = newSize] = null;
    }

    /**
     * 移除列表中所有元素.  调用后列表为空
     */
    public void clear() {
        modCount++;
        final Object[] es = elementData;
        for (int to = size, i = size = 0; i < to; i++)
            es[i] = null;
    }

    /**
     * 在列表的尾部添加指定集合的所有元素，按照指定集合的迭代器返回顺序(就原来啥样，导入就啥		 * 样)，如果在In Prgs时，修改了指定集合，那么这个行为将变成undefined. 意味着这个指定集	   * 合在列表中的行为是未定义的，列表且不为空
     *
     * 先转换集合，修改次数+1，判断长度。定义elementData，判断新集合长度 如果大于当前列表的
     * 长度(elememtData 长度初始为10 每次扩充1.5倍) 减去实际的长度，那么就说明 原来列表
     * 长度不够，需要扩充
     * 
     * @param c 包含将要被添加到列表中的集合
     * @return {@code true} 如果列表因调用而更改
     * @throws NullPointerException 如果指定集合为空
     */
    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        modCount++;
        int numNew = a.length;
        if (numNew == 0)
            return false;
        Object[] elementData;
        final int s;
        if (numNew > (elementData = this.elementData).length - (s = size))
            elementData = grow(s + numNew);
        System.arraycopy(a, 0, elementData, s, numNew);
        size = s + numNew;
        return true;
    }

    /**
     * 在列表指定位置中插入指定集合的素有元素，将位于该位置及后面的元素移动，这个新元素将按照指      * 定集合的迭代器顺序排列
     *
     * 先校验添加的索引，小于0 大于size，报越界错误，然后如上面的addall一样，常规操作
     *
     * @param index 索引，在此插入指定元素的第一个元素
     * @param c 包含所有要添加进列表的集合
     * @return {@code true} 如果列表调用而更改
     * @throws IndexOutOfBoundsException {@inheritDoc} 索引越界
     * @throws NullPointerException  如果集合为空
     */
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        modCount++;
        int numNew = a.length;
        if (numNew == 0)
            return false;
        Object[] elementData;
        final int s;
        if (numNew > (elementData = this.elementData).length - (s = size))
            elementData = grow(s + numNew);

        int numMoved = s - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index,
                             elementData, index + numNew,
                             numMoved);
        System.arraycopy(a, 0, elementData, index, numNew);
        size = s + numNew;
        return true;
    }

    /**
     * 移除列表中从formIndex到toIndex的元素，把随后的元素往左移(减去索引)，这个调用
     * 将缩短 toIndex - formIndex个元素列表，如果两者相等则无影响
     * 
     * formIndex > toIndex 越界错误，修改次数+1，移动(核心还是system.arrayCopy)
     *
     *
     * @throws IndexOutOfBoundsException if {@code fromIndex} or
     *         {@code toIndex} is out of range
     *         ({@code fromIndex < 0 ||
     *          toIndex > size() ||
     *          toIndex < fromIndex})
     */
    protected void removeRange(int fromIndex, int toIndex) {
        if (fromIndex > toIndex) {
            throw new IndexOutOfBoundsException(
                    outOfBoundsMsg(fromIndex, toIndex));
        }
        modCount++;
        shiftTailOverGap(elementData, fromIndex, toIndex);
    }

    /** 滑动元素 填补空隙 */
    private void shiftTailOverGap(Object[] es, int lo, int hi) {
        System.arraycopy(es, hi, es, lo, size - hi);
        for (int to = size, i = (size -= hi - lo); i < to; i++)
            es[i] = null;
    }

    /**
     * 检查add addAll是否越界的方法
     */
    private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    /**
     * 构建一个 索引越界异常的详细信息.
     * 在所有重构中，这个方法在服务器与VM里性能最好
     */
    private String outOfBoundsMsg(int index) {
        return "Index: "+index+", Size: "+size;
    }

    /**
     * 重载，用于检查removeRange
     */
    private static String outOfBoundsMsg(int fromIndex, int toIndex) {
        return "From Index: " + fromIndex + " > To Index: " + toIndex;
    }

    /**
     * 移除列表中 包含指定集合元素 的所有元素
     *
     * @param c 包含将要从列表中去除的元素集合
     * @return {@code true} 是否列表因调用而改变
     * @throws ClassCastException 如果列表元素与集合元素类不相同(optional)
     * @throws NullPointerException 如果列表包含null，且集合不允许null 或者指定的
     * 集合为null
     * 
     * 代码分析留在下方
     * @see Collection.contains(Object)
     */
    public boolean removeAll(Collection<?> c) {
        return batchRemove(c, false, 0, size);
    }

    /**
     * 仅保留此列表中包含在指定集合中的元素.
     * 换句话说，去除列表中所有不包含指定集合的元素
     *
     * @param c 包含要保留在此列表中的元素的集合
     * @return {@code true} 如果此列表因调用而更改
     * @throws ClassCastException 如果此列表元素的类与指定的集合不兼容
     * @throws NullPointerException 如果此列表包含 null 元素，并且指定的集合不允许 		 * null 元素（可选），或者指定的集合为 null
     * @see Collection.contains(Object)
     */
    public boolean retainAll(Collection<?> c) {
        return batchRemove(c, true, 0, size);
    }

    /**
     * param 集合c， boolean complement 是删还是保留，删false，保留true, form end
     * 
     * 首先检验是否为null，然后初始化赋值给es，为survivors进行初始化，form到end循环，
     * 看包不包含，用来截取第一位，也就是开始的位置
     * 符合complement的时候再进行赋新值，但这个时候也要保持与AbstractCollection的兼容性
     * 即使抛出异常
     * 接着增加结构修改次数，滑动补充空隙。
     */
    boolean batchRemove(Collection<?> c, boolean complement,
                        final int from, final int end) {
        Objects.requireNonNull(c);
        final Object[] es = elementData;
        int r;
        // Optimize for initial run of survivors
        for (r = from;; r++) {
            if (r == end)
                return false;
            if (c.contains(es[r]) != complement)
                break;
        }
        int w = r++;
        try {
            for (Object e; r < end; r++)
                if (c.contains(e = es[r]) == complement)
                    es[w++] = e;
        } catch (Throwable ex) {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            System.arraycopy(es, r, es, w, end - r);
            w += end - r;
            throw ex;
        } finally {
            modCount += end - w;
            shiftTailOverGap(es, w, end);
        }
        return true;
    }

    /**
     * 将ArrayList实例的状态保存到流中(序列化)
     *
     * 记录修改次数,元素数量,写入流中,把所有元素以正常顺序写入
     * 
     *
     * @param s the stream
     * @throws java.io.IOException if an I/O error occurs
     * @serialData The length of the array backing the {@code ArrayList}
     *             instance is emitted (int), followed by all of its elements
     *             (each an {@code Object}) in the proper order.
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioral compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    /**
     * 从流中重新构造ArrayList实例(反序列化)
     * @param s the stream
     * @throws ClassNotFoundException if the class of a serialized object
     *         could not be found
     * @throws java.io.IOException if an I/O error occurs
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // like clone(), allocate array based upon size not capacity
            SharedSecrets.getJavaObjectInputStreamAccess().checkArray(s, Object[].class, size);
            Object[] elements = new Object[size];

            // Read in all elements in the proper order.
            for (int i = 0; i < size; i++) {
                elements[i] = s.readObject();
            }

            elementData = elements;
        } else if (size == 0) {
            elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new java.io.InvalidObjectException("Invalid size: " + size);
        }
    }

    /**
     * 返回一个以正常顺序的列表迭代器, 从列表的指定位置开始
     * 这个指定索引表明了调用返回的第一个元素, 对next返回下一个，对previous返回上一个
     * 
     *
     * fail-fast快速检查机制 贴到md最开头了
     *
     * @throws IndexOutOfBoundsException {@inheritDoc} 越界异常
     */
    public ListIterator<E> listIterator(int index) {
        rangeCheckForAdd(index);
        return new ListItr(index);
    }

    /**
     * 返回此列表中元素的列表迭代器（按正确的顺序）。
     *
     * fail-fast
     *
     * @see #listIterator(int)
     */
    public ListIterator<E> listIterator() {
        return new ListItr(0);
    }

    /**
     * 按正确的顺序返回此列表中元素的迭代器。
     *
     * fail-fast
     *
     * @return an iterator over the elements in this list in proper sequence
     */
    public Iterator<E> iterator() {
        return new Itr();
    }

    /**
     * AbstractList.Itr优化版本
     */
    private class Itr implements Iterator<E> {
        int cursor;       // 下一个要返回的索引
        int lastRet = -1; // 上一个返回的索引，初始化在0的前面 lastRet=-1表示此位置无元素，lastRet表示此位						 //	置有元素
        int expectedModCount = modCount; // expected 记录Iterator的操作次数

        // 无参构造
        Itr() {}

        public boolean hasNext() {
            return cursor != size;
        }

        /**
         * i作为要返回的索引，大于等于size 显示 NoSuchElement报错
         * 然后赋值element, 再进行一次比较，cursor+1 返回元素
         * lastRet = i 即游标分为上一个跟下一个
         */
        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        /**
         * 移除 如果最后一个元素索引<0 报错，检查修改次数
         * 然后list remove这个元素,cursor = 元素索引, 期间越界报错
         */
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        /**
         * 定义size为元素数量,i = 下一个返回索引，照常校验,结束时更新一次减少堆写入
         */
        @Override
        public void forEachRemaining(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i < size) {
                final Object[] es = elementData;
                if (i >= es.length)
                    throw new ConcurrentModificationException();
                for (; i < size && modCount == expectedModCount; i++)
                    action.accept(elementAt(es, i));
                // update once at end to reduce heap write traffic
                cursor = i;
                lastRet = i - 1;
                checkForComodification();
            }
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }

    /**
     * 大致与上面一样 摆了
     */
    private class ListItr extends Itr implements ListIterator<E> {
        ListItr(int index) {
            super();
            cursor = index;
        }

        public boolean hasPrevious() {
            return cursor != 0;
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor - 1;
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            checkForComodification();
            int i = cursor - 1;
            if (i < 0)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i;
            return (E) elementData[lastRet = i];
        }

        public void set(E e) {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.set(lastRet, e);
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        public void add(E e) {
            checkForComodification();

            try {
                int i = cursor;
                ArrayList.this.add(i, e);
                cursor = i + 1;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    }

    /**
   	 * 返回一个指定元素之间的(fromIndex->toIndex)之间的列表切片,如果这俩索引相等则返回空
     * 这个返回的列表由本身的列表支持,所以没有什么结构上的改变，反之亦然，返回的列表也支持原来列表
     * 的操作(不确定对不对)
     * 这个方法消除了对显式范围的需要(类似数组的 [1],[2],[3]等 直接拿数据)
     * 任何需要列表的操作都可以传递子列表视图 而不是整个列表来范围操作 
     * 例如下例，去除了一部分元素
     * 
     * list.subList(from, to).clear();
     * 
     * 可以为indexOf和lastIndexof构建相同的idioms，所有函数 list能用的 子list也能用
     *
     * 如果支持列表(原列表)以返回的列表以外的任何方式进行结构修正，那么将变得undefined(不能进行结构上的更改)
     * 结构更改意思是 比如更改大小，或者其他方式干扰，可能会导致出错
     *
     * @throws IndexOutOfBoundsException {@inheritDoc}
     * @throws IllegalArgumentException {@inheritDoc}
     */
    public List<E> subList(int fromIndex, int toIndex) {
        subListRangeCheck(fromIndex, toIndex, size);
        return new SubList<>(this, fromIndex, toIndex);
    }

    // subList的定义
    private static class SubList<E> extends AbstractList<E> implements RandomAccess {
        private final ArrayList<E> root;
        private final SubList<E> parent;
        private final int offset;
        private int size;

        /**
         * 由一个任意的arrayList创建subList
         */
        public SubList(ArrayList<E> root, int fromIndex, int toIndex) {
            this.root = root;
            this.parent = null;
            this.offset = fromIndex;
            this.size = toIndex - fromIndex;
            this.modCount = root.modCount;
        }

        /**
         * 由一个subList构建另一个subList
         */
        private SubList(SubList<E> parent, int fromIndex, int toIndex) {
            this.root = parent.root;
            this.parent = parent;
            this.offset = parent.offset + fromIndex;
            this.size = toIndex - fromIndex;
            this.modCount = root.modCount;
        }

        //赋值 先校验size 校验修改次数 offset就是最开始list分割subList的index，这一步是为了把list修改
        public E set(int index, E element) {
            Objects.checkIndex(index, size);
            checkForComodification();
            E oldValue = root.elementData(offset + index);
            root.elementData[offset + index] = element;
            return oldValue;
        }
		
        //依然是查老列表
        public E get(int index) {
            Objects.checkIndex(index, size);
            checkForComodification();
            return root.elementData(offset + index);
        }

        //返回size toIndex-formIndex
        public int size() {
            checkForComodification();
            return size;
        }

        //add 仍然是老list加一个，并且更新size
        public void add(int index, E element) {
            rangeCheckForAdd(index);
            checkForComodification();
            root.add(offset + index, element);
            updateSizeAndModCount(1);
        }

        //移除。。跟add相反
        public E remove(int index) {
            Objects.checkIndex(index, size);
            checkForComodification();
            E result = root.remove(offset + index);
            updateSizeAndModCount(-1);
            return result;
        }

        //removeRange 也是正常操作 
        protected void removeRange(int fromIndex, int toIndex) {
            checkForComodification();
            root.removeRange(offset + fromIndex, offset + toIndex);
            updateSizeAndModCount(fromIndex - toIndex);
        }

      
        public boolean addAll(Collection<? extends E> c) {
            return addAll(this.size, c);
        }

        //先检查index,然后正常addAll
        public boolean addAll(int index, Collection<? extends E> c) {
            rangeCheckForAdd(index);
            int cSize = c.size();
            if (cSize==0)
                return false;
            checkForComodification();
            root.addAll(offset + index, c);
            updateSizeAndModCount(cSize);
            return true;
        }

        //参考如上replaceAllRange
        public void replaceAll(UnaryOperator<E> operator) {
            root.replaceAllRange(operator, offset, offset + size);
        }

        //remove于retain相同 只是参数true/false的区别 
        public boolean removeAll(Collection<?> c) {
            return batchRemove(c, false);
        }

        public boolean retainAll(Collection<?> c) {
            return batchRemove(c, true);
        }

        private boolean batchRemove(Collection<?> c, boolean complement) {
            checkForComodification();
            int oldSize = root.size;
            boolean modified =
                root.batchRemove(c, complement, offset, offset + size);
            if (modified)
                updateSizeAndModCount(root.size - oldSize);
            return modified;
        }

        public boolean removeIf(Predicate<? super E> filter) {
            checkForComodification();
            int oldSize = root.size;
            boolean modified = root.removeIf(filter, offset, offset + size);
            if (modified)
                updateSizeAndModCount(root.size - oldSize);
            return modified;
        }

        //依然是对原list操作
        public Object[] toArray() {
            checkForComodification();
            return Arrays.copyOfRange(root.elementData, offset, offset + size);
        }

        @SuppressWarnings("unchecked")
        public <T> T[] toArray(T[] a) {
            checkForComodification();
            if (a.length < size)
                return (T[]) Arrays.copyOfRange(
                        root.elementData, offset, offset + size, a.getClass());
            System.arraycopy(root.elementData, offset, a, 0, size);
            if (a.length > size)
                a[size] = null;
            return a;
        }

        public boolean equals(Object o) {
            if (o == this) {
                return true;
            }

            if (!(o instanceof List)) {
                return false;
            }

            boolean equal = root.equalsRange((List<?>)o, offset, offset + size);
            checkForComodification();
            return equal;
        }

        public int hashCode() {
            int hash = root.hashCodeRange(offset, offset + size);
            checkForComodification();
            return hash;
        }

        public int indexOf(Object o) {
            int index = root.indexOfRange(o, offset, offset + size);
            checkForComodification();
            return index >= 0 ? index - offset : -1;
        }

        public int lastIndexOf(Object o) {
            int index = root.lastIndexOfRange(o, offset, offset + size);
            checkForComodification();
            return index >= 0 ? index - offset : -1;
        }

        public boolean contains(Object o) {
            return indexOf(o) >= 0;
        }

        public Iterator<E> iterator() {
            return listIterator();
        }

        //listIterator也是，内容并无太大变化，因为都是对原List的操作
        public ListIterator<E> listIterator(int index) {
            checkForComodification();
            rangeCheckForAdd(index);

            return new ListIterator<E>() {
                int cursor = index;
                int lastRet = -1;
                int expectedModCount = root.modCount;

                public boolean hasNext() {
                    return cursor != SubList.this.size;
                }

                @SuppressWarnings("unchecked")
                public E next() {
                    checkForComodification();
                    int i = cursor;
                    if (i >= SubList.this.size)
                        throw new NoSuchElementException();
                    Object[] elementData = root.elementData;
                    if (offset + i >= elementData.length)
                        throw new ConcurrentModificationException();
                    cursor = i + 1;
                    return (E) elementData[offset + (lastRet = i)];
                }

                public boolean hasPrevious() {
                    return cursor != 0;
                }

                @SuppressWarnings("unchecked")
                public E previous() {
                    checkForComodification();
                    int i = cursor - 1;
                    if (i < 0)
                        throw new NoSuchElementException();
                    Object[] elementData = root.elementData;
                    if (offset + i >= elementData.length)
                        throw new ConcurrentModificationException();
                    cursor = i;
                    return (E) elementData[offset + (lastRet = i)];
                }

                public void forEachRemaining(Consumer<? super E> action) {
                    Objects.requireNonNull(action);
                    final int size = SubList.this.size;
                    int i = cursor;
                    if (i < size) {
                        final Object[] es = root.elementData;
                        if (offset + i >= es.length)
                            throw new ConcurrentModificationException();
                        for (; i < size && modCount == expectedModCount; i++)
                            action.accept(elementAt(es, offset + i));
                        // update once at end to reduce heap write traffic
                        cursor = i;
                        lastRet = i - 1;
                        checkForComodification();
                    }
                }

                public int nextIndex() {
                    return cursor;
                }

                public int previousIndex() {
                    return cursor - 1;
                }

                public void remove() {
                    if (lastRet < 0)
                        throw new IllegalStateException();
                    checkForComodification();

                    try {
                        SubList.this.remove(lastRet);
                        cursor = lastRet;
                        lastRet = -1;
                        expectedModCount = root.modCount;
                    } catch (IndexOutOfBoundsException ex) {
                        throw new ConcurrentModificationException();
                    }
                }

                public void set(E e) {
                    if (lastRet < 0)
                        throw new IllegalStateException();
                    checkForComodification();

                    try {
                        root.set(offset + lastRet, e);
                    } catch (IndexOutOfBoundsException ex) {
                        throw new ConcurrentModificationException();
                    }
                }

                public void add(E e) {
                    checkForComodification();

                    try {
                        int i = cursor;
                        SubList.this.add(i, e);
                        cursor = i + 1;
                        lastRet = -1;
                        expectedModCount = root.modCount;
                    } catch (IndexOutOfBoundsException ex) {
                        throw new ConcurrentModificationException();
                    }
                }

                final void checkForComodification() {
                    if (root.modCount != expectedModCount)
                        throw new ConcurrentModificationException();
                }
            };
        }

        public List<E> subList(int fromIndex, int toIndex) {
            subListRangeCheck(fromIndex, toIndex, size);
            return new SubList<>(this, fromIndex, toIndex);
        }

        private void rangeCheckForAdd(int index) {
            if (index < 0 || index > this.size)
                throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
        }

        private String outOfBoundsMsg(int index) {
            return "Index: "+index+", Size: "+this.size;
        }

        private void checkForComodification() {
            if (root.modCount != modCount)
                throw new ConcurrentModificationException();
        }

        private void updateSizeAndModCount(int sizeChange) {
            SubList<E> slist = this;
            do {
                slist.size += sizeChange;
                slist.modCount = root.modCount;
                slist = slist.parent;
            } while (slist != null);
        }

        /**
         * java8新加的接口，目的是为了stream流进行更小粒度的划分
         */
        public Spliterator<E> spliterator() {
            checkForComodification();

            // ArrayListSpliterator not used here due to late-binding
            return new Spliterator<E>() {
                private int index = offset; // 当前索引, modified on advance/split
                private int fence = -1; // 前一个索引，初始化为-1 即index(0)的前一个; then one past last 									   //index
                private int expectedModCount; // 初始化修改次数 initialized when fence set

                private int getFence() { // 首次使用时初始化
                    int hi; // (forEach方法中出现一个专门的变体(?))
                    if ((hi = fence) < 0) { // -1即第一次使用的时候
                        expectedModCount = modCount;
                        hi = fence = offset + size;
                    }
                    return hi;
                }

                public ArrayList<E>.ArrayListSpliterator trySplit() {
                    //逻辑右移，无论该数为正还是为负，右移之后左边都是补上0
                    int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
                    // ArrayListSpliterator可以在这用 source已经bound 
                    return (lo >= mid) ? null : // 区分两半，除非粒度已经太小了
                        root.new ArrayListSpliterator(lo, index = mid, expectedModCount);
                }

                /**
                 * 用来判断当前是否还有剩余元素
                 * 参数 Comsumer接口的实例
                 */
                public boolean tryAdvance(Consumer<? super E> action) {
                    Objects.requireNonNull(action);
                    int hi = getFence(), i = index;
                    if (i < hi) {
                        index = i + 1;
                        @SuppressWarnings("unchecked") E e = (E)root.elementData[i];
                        action.accept(e);
                        if (root.modCount != expectedModCount)
                            throw new ConcurrentModificationException();
                        return true;
                    }
                    return false;
                }

                /**
                 * 对每一个元素都执行Consumer的action行为
                 * 首先判断原list是否为空，然后是否第一次调用 fence是否为-1，是的话更新hi 不然mc
                 * 赋值为已经改变了的modifyCount，然后判断当前index是否在list范围中,执行操作 否则跳出
                 */
                public void forEachRemaining(Consumer<? super E> action) {
                    Objects.requireNonNull(action);
                    int i, hi, mc; // hoist accesses and checks from loop
                    ArrayList<E> lst = root;
                    Object[] a;
                    if ((a = lst.elementData) != null) {
                        if ((hi = fence) < 0) {
                            mc = modCount;
                            hi = offset + size;
                        }
                        else
                            mc = expectedModCount;
                        if ((i = index) >= 0 && (index = hi) <= a.length) {
                            for (; i < hi; ++i) {
                                @SuppressWarnings("unchecked") E e = (E) a[i];
                                action.accept(e);
                            }
                            if (lst.modCount == mc)
                                return;
                        }
                    }
                    throw new ConcurrentModificationException();
                }

                //还有多少元素要处理
                public long estimateSize() {
                    return getFence() - index;
                }
				
                //返回类型
                public int characteristics() {
                    return Spliterator.ORDERED | Spliterator.SIZED | Spliterator.SUBSIZED;
                }
            };
        }
    }

    /**
     * 抛出空指针异常 forEach循环，执行action 
     * @throws NullPointerException {@inheritDoc}
     */
    @Override
    public void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        final int expectedModCount = modCount;
        final Object[] es = elementData;
        final int size = this.size;
        for (int i = 0; modCount == expectedModCount && i < size; i++)
            action.accept(elementAt(es, i));
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }

    /**
     * 在列表中创建一个late-binding于fail-fast Spliterator
     *
     * <p>The {@code Spliterator} reports {@link Spliterator#SIZED},
     * {@link Spliterator#SUBSIZED}, and {@link Spliterator#ORDERED}.
     * Overriding implementations should document the reporting of additional
     * characteristic values.
     *
     * @return a {@code Spliterator} over the elements in this list
     * @since 1.8
     */
    @Override
    public Spliterator<E> spliterator() {
        return new ArrayListSpliterator(0, -1, 0);
    }

    /** Index-based split-by-two, 延迟初始化的 Spliterator  */
    final class ArrayListSpliterator implements Spliterator<E> {

        /*
         * 如果列表是不可变的, 或者结构上是不可变的,我们可以用Arrays.spliterator实现拆分
         * 然后尽可能多的检测，而不会牺牲太多性能。
         * 主要依靠modCounts。
         * 这些并不能检测并发冲突，有时候对线程干扰过于保守，但多检测一点是好的
         * 为了实现，我们首先懒加载 fence和expectedModCount，知道我们需要提交到检查状态的最新点
         * 提高精度(不适用于subList，他们是non-lazy value)
         * 我们执行一个ConcurrentModificationException在forEach结束的时候(性能最敏感)
         * 当使用forEach(与迭代器相反)，我们在操作之后检测，而非之前
         * 进一步的ConcurrentModificationException检测适用于其他违反假设的情况，例如null或者比给定size()
         * 小很多的elementDate数组，由于干扰而发生(这块其实没太懂)
         * 这允许forEach内部循环无需近一步检查执行，简化了lambda解析
         * 虽然这确实需要一些检查，但 list.stream().forEach(a) 常见情况下，除了forEach本身，不会在其他地方
         * 检查或计算
         * 其他不太常用的就用不到了
         */

        private int index; // 当前 index, modified on advance/split
        private int fence; // 不用为-1，否则为前一个索引
        private int expectedModCount; // set fence的时候初始化

        /** 给定的range创建一个新spliterator*/
        ArrayListSpliterator(int origin, int fence, int expectedModCount) {
            this.index = origin;
            this.fence = fence;
            this.expectedModCount = expectedModCount;
        }

        //与之前的基本相同
        private int getFence() { // initialize fence to size on first use
            int hi; // (a specialized variant appears in method forEach)
            if ((hi = fence) < 0) {
                expectedModCount = modCount;
                hi = fence = size;
            }
            return hi;
        }

        public ArrayListSpliterator trySplit() {
            int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
            return (lo >= mid) ? null : // divide range in half unless too small
                new ArrayListSpliterator(lo, index = mid, expectedModCount);
        }

        public boolean tryAdvance(Consumer<? super E> action) {
            if (action == null)
                throw new NullPointerException();
            int hi = getFence(), i = index;
            if (i < hi) {
                index = i + 1;
                @SuppressWarnings("unchecked") E e = (E)elementData[i];
                action.accept(e);
                if (modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                return true;
            }
            return false;
        }

        public void forEachRemaining(Consumer<? super E> action) {
            int i, hi, mc; // hoist accesses and checks from loop
            Object[] a;
            if (action == null)
                throw new NullPointerException();
            if ((a = elementData) != null) {
                if ((hi = fence) < 0) {
                    mc = modCount;
                    hi = size;
                }
                else
                    mc = expectedModCount;
                if ((i = index) >= 0 && (index = hi) <= a.length) {
                    for (; i < hi; ++i) {
                        @SuppressWarnings("unchecked") E e = (E) a[i];
                        action.accept(e);
                    }
                    if (modCount == mc)
                        return;
                }
            }
            throw new ConcurrentModificationException();
        }

        public long estimateSize() {
            return getFence() - index;
        }

        public int characteristics() {
            return Spliterator.ORDERED | Spliterator.SIZED | Spliterator.SUBSIZED;
        }
    }

    // 一些bit实现

    // 1-64内的返回1
    private static long[] nBits(int n) {
        return new long[((n - 1) >> 6) + 1];
    }
    /**
     * 用 long 类型数组，一个 long 64 位，就存64个位置的状态。
     * 看removeIf代码，符合条件的都会setBit，传入的条件i-beg 就是满足删除条件的索引与第一个满足索引的隔了多少
     * 元素 进入setBit 先看这个元素位于哪，bits[]
     * 就比如deathRow = 1L 那么为0000000000........00001
     * setBit后进行|=, 比如索引是5 第一个满足条件的是2 那么变成了0000000000........00101 这样64位直接存储
     * 那些需要被删的
     */
   
    private static void setBit(long[] bits, int i) {
        bits[i >> 6] |= 1L << i;
    }
   
    private static boolean isClear(long[] bits, int i) {
        return (bits[i >> 6] & (1L << i)) == 0;
    }

    /**
     * @throws NullPointerException {@inheritDoc}
     */
    @Override
    public boolean removeIf(Predicate<? super E> filter) {
        return removeIf(filter, 0, size);
    }

    /**
     * 移除满足filter的元素
     * i (inclusive) to index end (exclusive).
     */
    boolean removeIf(Predicate<? super E> filter, int i, final int end) {
        //校验是否为空
        Objects.requireNonNull(filter);
        int expectedModCount = modCount;
        final Object[] es = elementData;
        // 找到第一个符合条件的
        for (; i < end && !filter.test(elementAt(es, i)); i++)
            ;
        // Tolerate predicates that reentrantly access the collection for
        // read (but writers still get CME), so traverse once to find
        // elements to delete, a second pass to physically expunge.
        if (i < end) {
            final int beg = i;
            //找到满足条件的，deathRow进行存储
            final long[] deathRow = nBits(end - beg);
            deathRow[0] = 1L;   // set bit 0
            for (i = beg + 1; i < end; i++)
                //找到所有符合条件的
                if (filter.test(elementAt(es, i)))
                    setBit(deathRow, i - beg);
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            modCount++;
            //删除工作
            int w = beg;
            for (i = beg; i < end; i++)
                if (isClear(deathRow, i - beg))
                    es[w++] = es[i];
            shiftTailOverGap(es, w, end);
            return true;
        } else {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            return false;
        }
    }

    @Override
    public void replaceAll(UnaryOperator<E> operator) {
        replaceAllRange(operator, 0, size);
        modCount++;
    }

    private void replaceAllRange(UnaryOperator<E> operator, int i, int end) {
        Objects.requireNonNull(operator);
        final int expectedModCount = modCount;
        final Object[] es = elementData;
        for (; modCount == expectedModCount && i < end; i++)
            es[i] = operator.apply(elementAt(es, i));
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }

    @Override
    @SuppressWarnings("unchecked")
    public void sort(Comparator<? super E> c) {
        final int expectedModCount = modCount;
        Arrays.sort((E[]) elementData, 0, size, c);
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        modCount++;
    }

    void checkInvariants() {
        // assert size >= 0;
        // assert size == elementData.length || elementData[size] == null;
    }
    
    // 9.20更新完了 没参考其他blog
    // 看一遍他人写的再结合自己写的 https://juejin.cn/post/7019088552990343198
}
```