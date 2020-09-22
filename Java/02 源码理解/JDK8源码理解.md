# JDK源码学习

## 1 Object

常用方法说明

<img src="..\00 基础文件\images\Object方法图示.png" alt="image-20200921173032258" style="zoom: 80%;" />

1. **registerNatives()**

   Object对象初始化注册本地方法，native 关键字是 JNI（Java Native Interface）的重要体现，JNI 是Java调用其他语言（c，c++） 的一种机制。

   ```java
   private static native void registerNatives();
   static {
       registerNatives();
   }
   ```

2. **equals()**

   Object的equals()比较的是对象的地址

3. **hashCode()**

   返回对象的hashCode值

4. **toString()**

   返回全类名+'@'+16进制hashCode

   ```java
   public String toString() {
   	return getClass().getName() + "@" + Integer.toHexString(hashCode());
   }
   ```

5. **getClass()**

   native方法，调用此方法，返回运行时类名

6. **clone()**

   native方法，表示对对象的深复制。

7. **finalize()**

   垃圾回收器在认为该对象是垃圾对象的时候会调用该方法。子类可以通过重写该方法来达到资源释放的目的。 在方法调用过程中出现的异常会被忽略且方法调用会被终止，可能会导致内存泄漏无法得到回收，建议没有特殊需求不要重写该方法。

   **任何对象的该方法只会被调用一次**

8. **notify()、notifyAll()**

   多用于同步代码块、同步方法中。用于唤醒wait()的线程。

9. **wait()，wait(long timeout)，wait(long timeout, int nanos)**

   wait()线程需要先抛出中断异常，wait(long timeout, int nanos)timout毫秒单位，nanos纳秒，但是在JVM中只判断有值时，timeout加一毫秒。

   wait(0)表示一直wait(),直至被notify。

   ```java
       public final void wait(long timeout, int nanos) throws InterruptedException {
           if (timeout < 0) {
               throw new IllegalArgumentException("timeout value is negative");
           }
   
           if (nanos < 0 || nanos > 999999) {
               throw new IllegalArgumentException(
                                   "nanosecond timeout value out of range");
           }
   
           if (nanos > 0) {
               timeout++;
           }
   
           wait(timeout);
       }
   ```

**总结，Object类中，equal()、toString()、finalize()可以被重写，equal()重写时一般都会对hashCode修改。**



## 2 String

<img src="..\00 基础文件\images\String实现类.png" alt="image-20200921174616792" style="zoom:80%;" />

String类定义：

1. 被final修饰，表示String类不能被继承。
2. String类实现了Serializable、Comparable、CharSequence接口。
3. String底层是使用字符数组存储

```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    private final char value[];
}
```

源码理解：

1. **isEmpty()**

   通过比较字符串的长度来判断是否为空`value.length == 0`

2. **compareTo(String anotherString)**

   ```java
       public int compareTo(String anotherString) {
           int len1 = value.length;
           int len2 = anotherString.value.length;
           // 获取字符串中最短的字符长度
           int lim = Math.min(len1, len2);
           char v1[] = value;
           char v2[] = anotherString.value;
   
           int k = 0;
           while (k < lim) {
               char c1 = v1[k];
               char c2 = v2[k];
               // 判断字符串相同位置上的字符是否相同，如果相同比较下一位置
               if (c1 != c2) {
                   return c1 - c2;
               }
               k++;
           }
           //如果仍无法比较，则通过字符串长度进行比较
           return len1 - len2;
       }
   ```

3. **startsWith(String prefix, int toffset)**

   ```java
       public boolean startsWith(String prefix, int toffset) {
           char ta[] = value;
           int to = toffset;
           char pa[] = prefix.value;
           int po = 0;
           int pc = prefix.value.length;
           // Note: toffset might be near -1>>>1.
           // 判断offset值是否存在越界
           if ((toffset < 0) || (toffset > value.length - pc)) {
               return false;
           }
           // 比较次数是prefix的长度
           while (--pc >= 0) {
               // 从源字符串的index为toffset值开始与prefix第一个进行比较，依次向后比较，如果都相同则返回true，否则为false
               if (ta[to++] != pa[po++]) {
                   return false;
               }
           }
           return true;
       }
   ```

4. **endsWith(String suffix)**

   ```java
       public boolean endsWith(String suffix) {
           // 此次调用startsWith方法进行比较：后缀比较转换成前缀比较
           // 方法中的第二个参数就是用于计算出应该从哪个位置开始比较的索引
           return startsWith(suffix, value.length - suffix.value.length);
    
       }
   ```
5. **replace(char oldChar, char newChar)**
    ```java
    public String replace(char oldChar, char newChar) {
        // 首先判断要替换的新旧字符是否相同，如果相同直接返回
        if (oldChar != newChar) {
            // 获取源字符串长度
            int len = value.length;
            int i = -1;
            char[] val = value; /* avoid getfield opcode */
            // 判断游标是否越界
            while (++i < len) {
                // 判断是否找到需要替换的字符，如果找到，跳出循环，当前i的值为第一个需要替换的字符下标位置
                if (val[i] == oldChar) {
                    break;
                }
            }
            if (i < len) {
                // 开辟新的字符数组空间用来存放替换后的新字符串
                char buf[] = new char[len];
                // 将原来的字符串数组复制到新的字符串数组中
                for (int j = 0; j < i; j++) {
                    buf[j] = val[j];
                }
                // 从找到的第一个下标开始向下查找替换旧字符，直到全部查找完毕
                while (i < len) {
                    char c = val[i];
                    buf[i] = (c == oldChar) ? newChar : c;
                    i++;
                }
                return new String(buf, true);
            }
        }
        return this;
    }
    ```

## 3 StringBuilder

源码理解：

1. 对象实例化

   ```java
   StringBuilder sb = new StringBuilder();
   
   // 实例化过程
   public StringBuilder() {
   	super(16);
   }
   
   // 调用父类AbstractStringBuilder的AbstractStringBuilder(int capacity)，capacity初始值为16
   AbstractStringBuilder(int capacity) {
   	value = new char[capacity];
   }
   ```

2. append(String str)

   ```java
       public AbstractStringBuilder append(String str) {
           // 判断追加的字符串是否为null，如果为空则在追后追加“null”字符串，否则继续
           if (str == null)
               return appendNull();
           // 获取追加的字符串长度
           int len = str.length();
           // 确认是否需要扩容
           ensureCapacityInternal(count + len);
           // 追加字符串到最后
           str.getChars(0, len, value, count);
           count += len;
           return this;
       }
   
       private void ensureCapacityInternal(int minimumCapacity) {
           // overflow-conscious code
           if (minimumCapacity - value.length > 0) {
               value = Arrays.copyOf(value, newCapacity(minimumCapacity));
           }
       }
   
       private int newCapacity(int minCapacity) {
           // overflow-conscious code
           // 扩容为当前容量的2倍再加2
           int newCapacity = (value.length << 1) + 2;
           if (newCapacity - minCapacity < 0) {
               newCapacity = minCapacity;
           }
           return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
               ? hugeCapacity(minCapacity)
               : newCapacity;
       }
   
   ```

## 4 StringBuffer

源码理解：

1. append(String str)

```java
	// 直接在方法上添加synchronized来保证多线程安全，其他和StringBuilder操作一致
	public synchronized StringBuffer append(String str) {
        toStringCache = null;
        super.append(str);
        return this;
    }
```



## 5 Integer

源码理解：

1. valueOf

   ```java
       public static void main(String[] args) {
           Integer i1 = 66;// 通过反编译可以知道 Integer i = 66; 这种形式声明的变量是通过 java.lang.Integer#valueOf(int) 来构造 Integer 对象的
           Integer i2 = 66;
           Integer i3 = 128;
           Integer i4 = 128;
           System.out.println(i1 == i2);// true，在[-128,127]内时，会直接从 IntegerCache 中获取 Integer
           System.out.println(i3 == i4);// false，属于两个对象
       }
   
   
   	// 源码分析
   	public static Integer valueOf(int i) {
           // IntegerCache中预先存储了一个范围为：[-128, 127]对象，如果值在这个范围内就不需要重新实例化，所以在这个范围内的只要值相同，对象的地址就相同
           if (i >= IntegerCache.low && i <= IntegerCache.high)
               return IntegerCache.cache[i + (-IntegerCache.low)];
           return new Integer(i);
       }
   
   // IntegerCache源码
   private static class IntegerCache {
           static final int low = -128;
           static final int high;
           static final Integer cache[];
   
           static {
               // high value may be configured by property
               int h = 127;
               String integerCacheHighPropValue =
                   sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
               if (integerCacheHighPropValue != null) {
                   try {
                       int i = parseInt(integerCacheHighPropValue);
                       i = Math.max(i, 127);
                       // Maximum array size is Integer.MAX_VALUE
                       h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                   } catch( NumberFormatException nfe) {
                       // If the property cannot be parsed into an int, ignore it.
                   }
               }
               high = h;
   
               cache = new Integer[(high - low) + 1];
               int j = low;
               for(int k = 0; k < cache.length; k++)
                   cache[k] = new Integer(j++);
   
               // range [-128, 127] must be interned (JLS7 5.1.7)
               assert IntegerCache.high >= 127;
           }
   
           private IntegerCache() {}
       }
   ```

## 6 ThreadLocal

代码测试：

```java
public class ThreadTest {
    public static void main(String[] args) throws InterruptedException {
        ThreadLocal<Integer> count = new ThreadLocal<>();
        count.set(1);
        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                count.set(9999);
                System.out.println("我正在执行任务");
                System.out.println("当前线程:"+Thread.currentThread()+",count:"+count.get());
            }
        });
        t.start();
        t.join();
        System.out.println("当前线程:"+Thread.currentThread()+",count:"+count.get());
    }
}

运行结果：

我正在执行任务
当前线程:Thread[Thread-0,5,main],count:9999
当前线程:Thread[main,5,main],count:1
    
总结：
	根据结果我们发现ThreadLocal是针对当前线程有效，线程之间操作赋值不会影响
```

源码理解：

1. **set(T value)**

   ```java
   public void set(T value) {
       // 获取当前线程
       Thread t = Thread.currentThread();
       // 获取当前线程对象threadLocals属性
       ThreadLocalMap map = getMap(t);
       if (map != null)
           // this代表的是调用threadLoal对象，value就是对象的值
           map.set(this, value);
       else
           // 如果map为空，创建ThreadLocalMap对象，并设置，这是一种懒加载思想
           createMap(t, value);
   }
   ```

2. **get()**

   ```java
   public T get() {
       // 获取当前线程
       Thread t = Thread.currentThread();
       // 获取当前线程对象threadLocals属性
       ThreadLocalMap map = getMap(t);
       if (map != null) {
           //获取当前线程下对应threadLocal对象的Entry， Entry采用WeakReference弱引用，来解决内存泄漏问题
           ThreadLocalMap.Entry e = map.getEntry(this);
           if (e != null) {
               // 获取当前线程下的threadLocal对象的值
               @SuppressWarnings("unchecked")
               T result = (T)e.value;
               return result;
           }
       }
       // 如果不存在的话，则先创建一个值为null的localThread对象进行存储在ThreadLocalMap中，给后续使用
       return setInitialValue();
   }
   ```



## 7 ClassLoader

源码理解：

1. **loadClass(String name, boolean resolve)**

   ```java
       protected Class<?> loadClass(String name, boolean resolve)
           throws ClassNotFoundException
       {
           synchronized (getClassLoadingLock(name)) {
               // First, check if the class has already been loaded
               // 首先，检查这个类是否已经被加载
               Class<?> c = findLoadedClass(name);
               if (c == null) {
                   long t0 = System.nanoTime();
                   try {
                       // 如果父类加载器不为空，则使用父类加载器进行加载，否则使用BootstrapClass类加载器加载
                       if (parent != null) {
                           c = parent.loadClass(name, false);
                       } else {
                           c = findBootstrapClassOrNull(name);
                       }
                   } catch (ClassNotFoundException e) {
                       // ClassNotFoundException thrown if class not found
                       // from the non-null parent class loader
                   }
                   // 如果所有父类加载器都没有加载成功，BootstrapClass类也没有加载成功，则进行下面操作
                   if (c == null) {
                       // If still not found, then invoke findClass in order
                       // to find the class.
                       long t1 = System.nanoTime();
                       c = findClass(name);
   
                       // this is the defining class loader; record the stats
                       sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                       sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                       sun.misc.PerfCounter.getFindClasses().increment();
                   }
               }
               if (resolve) {
                   resolveClass(c);
               }
               return c;
           }
       }
      
   总结：类加载顺序，会优先由父类加载器进行加载，不断递归向上，最后由BootstrapClass加载器进行加载，从下到上查缓存，从上到下进行加载。
   ```



## 8 ArrayList

继承关系：

![](..\00 基础文件\images\ArrayList.png)

类定义：

```java
// 继承AbstractList并且实现了RandomAccess接口
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
	// 初始默认容量为10
    private static final int DEFAULT_CAPACITY = 10;
    // 底层存储结构是Object数组
    transient Object[] elementData
    // 数组的大小
    private int size;
    
    
    // 带有初始容量的构造函数
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            // 容量大于0的情况下，实例化一个指定容量的Object数组
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            // 容量大于0的情况下，构造一个空{}
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
}
```

源码理解：

1. **grow(int minCapacity)**

   ```java
   // 列表扩容
   private void grow(int minCapacity) {
       // overflow-conscious code
       int oldCapacity = elementData.length;
       // 得到扩容后的新数组容量：当前数组容量的1/2+当前数组的容量，也就是1.5倍的当前容量
       int newCapacity = oldCapacity + (oldCapacity >> 1);
       if (newCapacity - minCapacity < 0)
           newCapacity = minCapacity;
       if (newCapacity - MAX_ARRAY_SIZE > 0)
           newCapacity = hugeCapacity(minCapacity);
       // minCapacity is usually close to size, so this is a win:
       elementData = Arrays.copyOf(elementData, newCapacity);
   }
   ```

2. **add(E e)**

   ```java
   public boolean add(E e) {
       // 添加元素之前检查是否需要扩容，每次add操作都会导致modCount加1，来保证安全
       ensureCapacityInternal(size + 1);  // Increments modCount!!
       // 添加新增元素
       elementData[size++] = e;
       return true;
   }
   ```

3. **iterator()**

   ```java
       	int cursor;       // index of next element to return
       	int lastRet = -1; // index of last element returned; -1 if no such
       	// 将modCount值赋值给expectedModCount，用于检查一致性
       	int expectedModCount = modCount;
   		public E next() {
               // 检查modCount和expectedModCount是否相同，如果相同则没有被修改过，如果不相同，则抛出ConcurrentModificationException异常
               checkForComodification();
               // 刷新当前索引
               int i = cursor;
               if (i >= size)
                   throw new NoSuchElementException();
               Object[] elementData = ArrayList.this.elementData;
               if (i >= elementData.length)
                   throw new ConcurrentModificationException();
               // 索引指向下一个
               cursor = i + 1;
               // 返回当前索引的元素
               return (E) elementData[lastRet = i];
           }
   
           public void remove() {
               if (lastRet < 0)
                   throw new IllegalStateException();
               checkForComodification();
   
               try {
                   // remove元素后，modCount增加1
                   ArrayList.this.remove(lastRet);
                   // 刷新游标索引
                   cursor = lastRet;
                   // 重新设置索引
                   lastRet = -1;
                   // 重新赋值expectedModCount，保证不抛异常
                   expectedModCount = modCount;
               } catch (IndexOutOfBoundsException ex) {
                   throw new ConcurrentModificationException();
               }
           }
   ```

## 9 HashMap

继承关系：

![](..\00 基础文件\images\HashMap.png)

类定义：

```java
// 底层数据结构为Node数组+链表/红黑树的形式存储
// 继承AbstractMap，实现Map接口
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    // 默认初始化容量为16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    // 默认扩容因子为0.75
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //  链表转成红黑树的阈值为8
    static final int TREEIFY_THRESHOLD = 8;
    // 黑树转为链表的阈值为6
    static final int UNTREEIFY_THRESHOLD = 6;
    // 最小树行化阀值为64，当哈希表中的容量 > 该值时，才允许树形化链表，为了避免进行扩容、树形化选择的冲突，这个值不能小于 4 * TREEIFY_THRESHOLD
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    
}
```



源码理解：

1.  **put(K key, V value)**

   ```java
   // 添加元素
   public V put(K key, V value) {
       return putVal(hash(key), key, value, false, true);
   }
   
      final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                      boolean evict) {
           Node<K,V>[] tab; Node<K,V> p; int n, i;
           if ((tab = table) == null || (n = tab.length) == 0)
               // 初始化容量或者检查扩容
               n = (tab = resize()).length;
           if ((p = tab[i = (n - 1) & hash]) == null)
               // 当前位置为空，直接插入
               tab[i] = newNode(hash, key, value, null);
           else {
               Node<K,V> e; K k;
               if (p.hash == hash &&
                   ((k = p.key) == key || (key != null && key.equals(k))))
                   // 当前位置存在相同的key，直接覆盖
                   e = p;
               else if (p instanceof TreeNode)
                   // 当前结构为树，写入到树
                   e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
               else {
                   for (int binCount = 0; ; ++binCount) {
                       if ((e = p.next) == null) {
                           // 将新数据写入到链表最后
                           p.next = newNode(hash, key, value, null);
                           // 检查链表长度是否需要转化成树
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
               // 如果存在值，覆盖并返回旧值
               if (e != null) { // existing mapping for key
                   V oldValue = e.value;
                   if (!onlyIfAbsent || oldValue == null)
                       e.value = value;
                   afterNodeAccess(e);
                   return oldValue;
               }
           }
           // modCount加一，防止添加元素过程中存在遍历行为
           ++modCount;
           // 判断是否满足扩容条件
           if (++size > threshold)
               resize();
           afterNodeInsertion(evict);
           return null;
       }
   
   ```

2. **resize()**

   ```java
   final Node<K,V>[] resize() {
       Node<K,V>[] oldTab = table;
       // 得到当前容量
       int oldCap = (oldTab == null) ? 0 : oldTab.length;
       // 得到下一次需要扩容的阀值
       int oldThr = threshold;
       // 初始化新的容量和新的扩容阀值
       int newCap, newThr = 0;
       // 判断是否是第一次进入
       if (oldCap > 0) {
           if (oldCap >= MAXIMUM_CAPACITY) {
               threshold = Integer.MAX_VALUE;
               return oldTab;
           }
           else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
               // 扩容后容量为当前容量的2倍，扩容阀值扩大2倍
               newThr = oldThr << 1; // double threshold
       }
       else if (oldThr > 0) // initial capacity was placed in threshold
           newCap = oldThr;
       else {               // zero initial threshold signifies using defaults
           newCap = DEFAULT_INITIAL_CAPACITY;
           newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
       }
       if (newThr == 0) {
           float ft = (float)newCap * loadFactor;
           newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                     (int)ft : Integer.MAX_VALUE);
       }
       // 重置下一次扩容阀值
       threshold = newThr;
       @SuppressWarnings({"rawtypes","unchecked"})
       Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
       table = newTab;
       // 判断resize之前oldTab是否为空，如果是空，说明第一次初始化，还没有数据，直接返回就行，如果有数据，则要把之前的数据复制到新的table中去。
       if (oldTab != null) {
           // 遍历旧的table元素
           for (int j = 0; j < oldCap; ++j) {
               Node<K,V> e;
               if ((e = oldTab[j]) != null) {
                   // 重置老的位置为空，便于回收
                   oldTab[j] = null;
                   if (e.next == null)
                       // 位置没有数据，直接写入
                       newTab[e.hash & (newCap - 1)] = e;
                   else if (e instanceof TreeNode)
                       // 当前位置是树，则通过树的方式写入
                       ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                   else { // preserve order
                       // 当前位置是链表，则通过链表的方式写入
                       Node<K,V> loHead = null, loTail = null;
                       Node<K,V> hiHead = null, hiTail = null;
                       Node<K,V> next;
                       do {
                           next = e.next;
                           // 判断当前位置是否需要移动
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

3. **get(Object key)**

   ```java
       public V get(Object key) {
           Node<K,V> e;
           return (e = getNode(hash(key), key)) == null ? null : e.value;
       }
   
       final Node<K,V> getNode(int hash, Object key) {
           Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
           if ((tab = table) != null && (n = tab.length) > 0 &&
               (first = tab[(n - 1) & hash]) != null) {
               // 检查数组的第一个位置是否满足要求，如果满足直接返回，如果不满足，则遍历链表或者红黑树进行查找
               if (first.hash == hash && // always check first node
                   ((k = first.key) == key || (key != null && key.equals(k))))
                   return first;
               if ((e = first.next) != null) {
                   // 如果当前位置为树节点，则进行递归查找符合的键值对
                   if (first instanceof TreeNode)
                       return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                   // 如果当前节点为链表，则进行遍历查找符合的键值对
                   do {
                       if (e.hash == hash &&
                           ((k = e.key) == key || (key != null && key.equals(k))))
                           return e;
                   } while ((e = e.next) != null);
               }
           }
           return null;
       }
       
   总结：key相同：hash()相同，equals()相同
   ```

## 10 ConcurrentHashMap

源码理解：

1. **put(K key, V value)**

   ```java
   final V putVal(K key, V value, boolean onlyIfAbsent) {
           if (key == null || value == null) throw new NullPointerException();// key和value不允许null
           int hash = spread(key.hashCode());//两次hash，减少hash冲突，可以均匀分布
           int binCount = 0;//i处结点标志，0: 未加入新结点, 2: TreeBin或链表结点数, 其它：链表结点数。主要用于每次加入结点后查看是否要由链表转为红黑树
           for (Node<K, V>[] tab = table; ; ) {//CAS经典写法，不成功无限重试
               Node<K, V> f;
               int n, i, fh;
               //检查是否初始化了，如果没有，则初始化
               if (tab == null || (n = tab.length) == 0)
                   tab = initTable();
               /**
                * i=(n-1)&hash 等价于i=hash%n(前提是n为2的幂次方).即取出table中位置的节点用f表示。 有如下两种情况：
                * 1、如果table[i]==null(即该位置的节点为空，没有发生碰撞)，则利用CAS操作直接存储在该位置， 如果CAS操作成功则退出死循环。
                * 2、如果table[i]!=null(即该位置已经有其它节点，发生碰撞)
                */
               else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                   if (casTabAt(tab, i, null,
                           new Node<K, V>(hash, key, value, null)))
                       break;                   // no lock when adding to empty bin
               } else if ((fh = f.hash) == MOVED)//检查table[i]的节点的hash是否等于MOVED，如果等于，则检测到正在扩容，则帮助其扩容
                   tab = helpTransfer(tab, f);
               else {//table[i]的节点的hash值不等于MOVED。
                   V oldVal = null;
                   // 针对首个节点进行加锁操作，而不是segment，进一步减少线程冲突
                   synchronized (f) {
                       if (tabAt(tab, i) == f) {
                           if (fh >= 0) {
                               binCount = 1;
                               for (Node<K, V> e = f; ; ++binCount) {
                                   K ek;
                                   // 如果在链表中找到值为key的节点e，直接设置e.val = value即可
                                   if (e.hash == hash &&
                                           ((ek = e.key) == key ||
                                                   (ek != null && key.equals(ek)))) {
                                       oldVal = e.val;
                                       if (!onlyIfAbsent)
                                           e.val = value;
                                       break;
                                   }
                                   // 如果没有找到值为key的节点，直接新建Node并加入链表即可
                                   Node<K, V> pred = e;
                                   if ((e = e.next) == null) {//插入到链表末尾并跳出循环
                                       pred.next = new Node<K, V>(hash, key,
                                               value, null);
                                       break;
                                   }
                               }
                           } else if (f instanceof TreeBin) {// 如果首节点为TreeBin类型，说明为红黑树结构，执行putTreeVal操作
                               Node<K, V> p;
                               binCount = 2;
                               if ((p = ((TreeBin<K, V>) f).putTreeVal(hash, key,
                                       value)) != null) {
                                   oldVal = p.val;
                                   if (!onlyIfAbsent)
                                       p.val = value;
                               }
                           }
                       }
                   }
                   if (binCount != 0) {
                       // 如果节点数>＝8，那么转换链表结构为红黑树结构
                       if (binCount >= TREEIFY_THRESHOLD)
                           treeifyBin(tab, i);//若length<64,直接tryPresize,两倍table.length;不转红黑树
                       if (oldVal != null)
                           return oldVal;
                       break;
                   }
               }
           }
           // 计数增加1，有可能触发transfer操作(扩容)
           addCount(1L, binCount);
           return null;
       }
   
   总结：线程安全的HashMap，是通过Cas+Synchronized来保证的，如果没有发生hash冲突，通过cas循环操作，如果发生冲突，则通过synchronized锁住node首节点，进行安全写入。
   ```

## 11 ThreadPoolExecutor

源码理解：

1. execute()

   ```java
       public void execute(Runnable command) {
           // 若任务为空，则抛 NPE，不能执行空任务
           if (command == null) {
               throw new NullPointerException();
           }
           int c = ctl.get();
           // 若工作线程数小于核心线程数，则创建新的线程，并把当前任务 command 作为这个线程的第一个任务
           if (workerCountOf(c) < corePoolSize) {
               if (addWorker(command, true)) {
                   return;
               }
               c = ctl.get();
           }
           /**
            * 至此，有以下两种情况：
            * 1.当前工作线程数大于等于核心线程数
            * 2.新建线程失败
            * 此时会尝试将任务添加到阻塞队列 workQueue
            */
           // 若线程池处于 RUNNING 状态，将任务添加到阻塞队列 workQueue 中
           if (isRunning(c) && workQueue.offer(command)) {
               // 再次检查线程池标记
               int recheck = ctl.get();
               // 如果线程池已不处于 RUNNING 状态，那么移除已入队的任务，并且执行拒绝策略
               if (!isRunning(recheck) && remove(command)) {
                   // 任务添加到阻塞队列失败，执行拒绝策略
                   reject(command);
               }
               // 如果线程池还是 RUNNING 的，并且线程数为 0，那么开启新的线程
               else if (workerCountOf(recheck) == 0) {
                   addWorker(null, false);
               }
           }
           /**
            * 至此，有以下两种情况：
            * 1.线程池处于非运行状态，线程池不再接受新的线程
            * 2.线程处于运行状态，但是阻塞队列已满，无法加入到阻塞队列
            * 此时会尝试以最大线程数为限制创建新的工作线程
            */
           else if (!addWorker(command, false)) {
               // 任务进入线程池失败，执行拒绝策略
               reject(command);
           }
       }
   ```

## 12 AtomicInteger

源码理解：

1. **getAndAddInt(Object var1, long var2, int var4)**

   ```java
   public final int getAndAddInt(Object var1, long var2, int var4) {
       int var5;
       do {
           var5 = this.getIntVolatile(var1, var2);
       } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
   
       return var5;
   }
   ```