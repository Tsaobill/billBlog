### 开启线程的方式

1. 继承Thread类，重写run方法，然后调用start方法。
2. 实现Runnable接口，实现run方法，传给Thread类对象。



### 停止线程的方法

1. stop方法。已过时，不推荐。因为这个方法太过于暴力，立即销毁线程，可能会导致脏数据。
2. 自行给重现了run方法的类添加一个boolean类型的volatile字段和一个自定义stop方法，在run方法中进行工作前判断该字段是否被改变，然后就可以在适当的时候调用自定义的stop方法改变字段即可。

### 线程中断

线程中断是Java多线程中一种重要的线程协作机制。

> `public void Thread.interrupt() ` // 中断线程
>
> `public boolean Thread.isInterrupted()` // 判断线程是否被中断
>
> `public static boolean Thread.interrupted() ` // 判断是否被中断，并清楚中断状态标志

上述前两个方法配合来进行停止线程，比自定义的停止方法更强劲，因为在线程调用sleep方法后我们不能及时退出，而使用这两个方法可以，因为sleep时调用interrupt方法会抛出异常，我们可以在捕获到异常的时候，退出线程，或者做一下操作维护数据一致性和完整性，然后再调用interrupt方法置上中断标志位，然后在线程下次执行操作之前通过isInterrupted方法判断后再退出。

> 注意：`Thread.sleep()`方法由于中断而抛出异常，此时， 它会清除中断标记，如果不加处理，那么在下一次循环开始时， 就无法捕获这个中断，故在异常处理中，再次设置中断标记位。

