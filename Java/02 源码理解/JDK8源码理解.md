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

常用方法说明：

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