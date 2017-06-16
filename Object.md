# Object类
## equals
```java
public boolean equals(Object obj) {
        return (this == obj);
}
```

## hashCode
```java
public int hashCode() {
    int lockWord = shadow$_monitor_;
    final int lockWordStateMask = 0xC0000000;  // Top 2 bits.
    final int lockWordStateHash = 0x80000000;  // Top 2 bits are value 2 (kStateHash).
    final int lockWordHashMask = 0x0FFFFFFF;  // Low 28 bits.
    if ((lockWord & lockWordStateMask) == lockWordStateHash) {
        return lockWord & lockWordHashMask;
    }
    return System.identityHashCode(this);
}
```

## clone
```java
protected Object clone() throws CloneNotSupportedException {
    if (!(this instanceof Cloneable)) {
        throw new CloneNotSupportedException("Class " + getClass().getName() +
                                                 " doesn't implement Cloneable");
    }

    return internalClone();
}
```

（1）实现深拷贝较为困难，需要整个类继承系列的所有类都很好的实现clone方法。

（2）需要处理CloneNotSupportedException异常。Object类中的clone方法被声明为可能会抛出CloneNotSupportedException，因此在子类中，需要对这一异常进行处理。

《Effective Java》的作者Joshua Bloch建议我们不应该实现Cloneable接口，而应该使用拷贝构造器或者拷贝工厂。

## toString
```java
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```

## finalize
类似析构函数，当垃圾回收器确定不存在对该对象的更多引用时，由对象的垃圾回收器调用此方法，用于释放资源。
finalized方法在由GC调用，并且还允许一次再生(标记为finalized后该对象在这个线程中又可达了)，可能会拖慢GC速度导致频繁fullGC，甚至直接OOM。
finalize的执行是不确定的，既不确定由哪个线程执行，也不确定执行的顺序。
由于finalize方法可能影响GC，因此可以使用PhantomReference虚引用来替代finalize的功能。
effective java告诉我们，最好的做法是提供close()方法，并且告知上层应用在不需要该对象时一掉要调用这类接口。

### finalize方法缺陷

* 重写finalize方法对象的创建过程
    
    * 创建对象A实例
    
    * 创建java.lang.ref.Finalizer对象实例F1，F1指向A和一个ReferenceQueue,引用关系：F1—>A，F1—>ReferenceQueue(Finalizer类静态方法创建一个后台线程从queue中执行runFinalizer方法)
    
    * java.lang.ref.Finalizer的类对象引用F1，这样可以保持F1永远不会被回收，除非解除Finalizer的类对象对F1的引用。
    
    * 经过上述三个步骤，引用关系：java.lang.ref.Finalizer–>F1–>A，F1–>ReferenceQueue。

* 重写finalize方法对象的GC回收过程
    
    * 在发生minor gc时，即便一个对象A不被任何其他对象引用，只要它含有override finalize()，就会最终被java.lang.ref.Finalizer类的一个对象F1引用
    
    * 如果新生代的对象都含有override finalize()，那岂不是无法GC？没错，这就是finalize()的第一个风险所在；
    
    * minor gc会把所有活跃对象以及被java.lang.ref.Finalizer类对象引用的（实际）垃圾对象拷贝到下一个survivor区域，如果拷贝溢 出，就将溢出的数据晋升到老年代，极端情况下，老年代的容量会被迅速填满，于是让人头痛的频繁full gc就离我们不远了。
    
    * 当第一次minor gc中发现一个对象只被java.lang.ref.Finalizer类对象引用时，GC线程会把指向对象A的Finalizer对象F1塞入F1所引用的ReferenceQueue中；
    
    * java.lang.ref.Finalizer类对象中包含了一个运行级别很低的deamon线程 finalizer来异步地调用这些对象的finalize()方法，调用完之后，java.lang.ref.Finalizer类对象会清除自己对 F1的引用。这样GC线程就可以在下一次minor gc时将对象A回收掉。
    
    * 可见Finalizer对象的多少也会直接影响minor gc的快慢。
    
* finalizer方法的对象回收过程总结下来，有以下三个风险：
    
    * 如果随便一个finalize()抛出一个异常，finallize线程会终止，很快地会由于ReferenceQueue的不断增长导致OOM
    
    * finalizer线程运行级别很低，有可能出现finalize速度跟不上对象创建速度，最终可能还是会OOM，实际应用中一般会有富裕的CPU时间，所以这种OOM情况可能不太常出现
    
    * 含有override finalize()的对象至少要经历两次GC才能被回收，严重拖慢GC速度，运气不好的话直接晋升到老年代，可能会造成频繁的full gc，进而影响这个系统的性能和吞吐率。
