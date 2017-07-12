
## 泄露对象是什么
### 三种内存分配
 1. 静态存储区 : 静态数据、全局static数据和常量
 2. 栈区 : 在执行函数时，函数内局部变量的存储单元
 3. 堆区 : 程序运行 new 的内存 
```
    public class A{
       //成员变量全部存储在堆中（包括基本数据类型，引用和引用的对象实体） 
       //因为它们属于类，类对象终究是要被new出来使用的。
       int a = 1;   
       Obj f = new Obj();
       public void foo(){
           //局部变量的基本数据类型和引用存储于栈中
           int b = 2; 
           //new 出的对象实体存储于堆中
           Obj o = new Obj();
       }
    }
    // aObj 这个引用存在栈中
    // aObj new 出的对象存在堆中
    A aObj = new A();
```   
内存泄露只针对堆内存

## 内存为什么会泄露
### 对象可达性判断
如果一个对象具有一个或多个引用，称这个对象是可达的。
如上面的例子 ```A aObj = new A();``` JVM堆区 存储新创建的 A 的实例, 栈区存放 aObj（引用），存放的是 A 实例的地址。
此时，A 的实例是可达的，因为有 aObj 这个引用指向它。 
### 内存回收机制
Java的内存垃圾回收机制是从程序的主要运行对象(如静态对象/寄存器/栈上指向的堆内存对象等)开始检查引用链，当遍历一遍后得到上述这些可达对象组成无法回收的对象集合，而其他孤立的不可达对象就作为垃圾回收。
### 实例
```
    private static PopSharedPreferences instance = null;
    private static Context mContext = null;
    public static PopSharedPreferences getInstance(Context context) {
        if (instance == null) {
            instance = new PopSharedPreferences(context);
        }
        return instance;
    }
    
    private PopSharedPreferences(Context context) {
        this.mContext = context;
        ....
    }
```  
```
    public class PopAudioPlayer extends HomeAsBackActivity implements PopWindowParent {
        @Override
        protected void onCreate(Bundle icicle) {
            mPreferences = PopSharedPreferences.getInstance(this);
        }
    }
```  
GC ROOTS --> .... --> PopSharedPreferences --> mContext --> PopAudioPlayer  
PopAudioPlayer 销毁 ，但是 mContext 被 PopSharedPreferences 静态实例持有。 GC 无法回收 PopAudioPlayer 对象占用的堆内存。
mContext 改成 ApplicationContext 打断了 PopSharedPreferences 对象和 PopAudioPlayer 对象的引用关系。

## 强引用、软引用、弱引用和虚引用
1. 强引用 用的大部分都是强引用，new 一个对象等。 GC 不会回收。
2. 软引用 ```SoftReference``` 如果内存空间足够，垃圾回收器就不会回收它，如果内存空间不足了，就会回收这些对象的内存。
3. 弱引用 ```WeakReference``` 在垃圾回收器扫描过程中，一旦发现了只具有（仅有）弱引用的对象，不管当前内存空间是否足够都会被回收。
4. 虚引用 ```PhantomReference``` 如果一个对象仅持有虚引用，那么这个对象就如同没有任何引用一样，随时可能被GC回收。主要用来跟踪对象被GC回收的过程。
        
