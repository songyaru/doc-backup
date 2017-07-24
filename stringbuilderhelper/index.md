### 背景

ES 加入新文件提醒功能模块废弃了使用 mediastore 来查找所有文件，而是基于 android 底层的文件系统自己实现了一套全盘扫描监控文件的方案。

这种方案可以更方便的处理和监控文件的变化，但由于涉及到全盘扫描，对于数据量庞大的文件来说，内存占用是很大的一个挑战。


#### android 的内存碎片和 OutOfMemoryError
OutOfMemoryError 不一定是内存不足！！！

ES 的文件扫描方案分配和使用了大量的对象，由于5.0以前的系统（dalvik 虚拟机）的内存回收算法会造成大量的内存碎片，经过一段时间的运行后出现内存不足的 Error。(android GC 见末尾的文章参考)


#### StringBuilderHelper 减少碎片产生
在做内存优化的时候用到了一个小功能函数解决一个大问题。
``` java
/**	
 * 重复利用 StringBuild 的内存区域，减少 enlargeBuffer 造成的内存碎片	
 */
public class StringBuilderHelper {
    private static final ThreadLocal<StringBuilder> threadLocalStringBuilder = new ThreadLocal<StringBuilder>() {
        @Override
        protected StringBuilder initialValue() {
            return new StringBuilder(256);
        }
    };

    public static StringBuilder getThreadLocalStringBuilder() {
        StringBuilder sb = threadLocalStringBuilder.get();
        sb.setLength(0);
        return sb;
    }

}
```
将所有 ``` StringBuilder sb = ew StringBuilder();``` 替换成
``` StringBuilder sb = StringBuilderHelper.getThreadLocalStringBuilder();``` 和 StringBuilder 一样使用即可。

在一台没有太多文件的手机上对比，StringBuilderHelper 减少了 300K 的内存占用。


### 参考
* [揭秘 ART 细节 ---- Garbage collection](http://www.cnblogs.com/jinkeep/p/3818180.html)
* [Android系统中的进程管理：内存的回收](http://qiangbo.space/2016-12-08/AndroidAnatomy_Process_Recycle/)
* [浅谈StringBuilder](http://www.jianshu.com/p/160c9be0b132)

