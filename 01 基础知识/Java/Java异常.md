# Java异常

Java从Throwable直接派生出Exception和Error。其中Exception是可以抛出的基本类型，在Java类库、方法以及运行时故障中都可能抛出Exception型异常。Exception表示可以恢复的异常，是编译器可以捕捉到的；Error表示编译时和系统错误，表示系统在运行期间出现了严重的错误，属于不可恢复的错误，由于这属于JVM层次的严重错误，因此这种错误会导致程序终止执行。Exception又分为检查异常和运行时异常。

**典型的RuntimeException(运行时异常)**包括NullPointerException, ClassCastException(类型转换异常)，IndexOutOfBoundsException(越界异常), IllegalArgumentException(非法参数异常),ArrayStoreException(数组存储异常),AruthmeticException(算术异常),BufferOverflowException(缓冲区溢出异常)等；

**非RuntimeException(检查异常)**包括IOException, SQLException,InterruptedException(中断异常-调用线程睡眠时候),NumberFormatException(数字格式化异常)等。

而按照编译器检查方式划分，异常又可以分为检查型异常（CheckedException）和非检查型异常 （UncheckedException）。Error和RuntimeException合起来称为UncheckedException，之所以这么 称呼，是因为编译器不检查方法是否处理或者抛出这两种类型的异常，因此编译期间出现这种类型的异常也不会报错，默认由虚拟机提供处理方式。除了Error 和RuntimeException这两种类型的异常外，其它的异常都称为Checked异常。

一般会选择继承Exception和RuntimeException，如果不要求调用者一定要处理抛出的异常，就继承RuntimeException。

CustomException继承自Exception或RuntimeException，就属于自定义异常了。

一般来说，自定义异常的作用有以下情形：

1)、将检查型异常转换为非检查型异常。

2)、在产生异常时封装上下文信息、定义异常码、收集环境对象，有利于信息的传递。



关于finally需要注意的几点：

1)、finally中的代码总是会被执行，除非在执行try或者catch语句时虚拟机退出（System.exit(1))。

2)、finally块可以做一些资源清理工作，如关闭文件、关闭游标等操作。

3)、finally块不是必须的。

另外，如果在try和finally块中都执行了return语句，最终返回的将是finally中的return值