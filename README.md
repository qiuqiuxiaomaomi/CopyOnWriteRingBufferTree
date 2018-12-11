# CopyOnWriteRingBufferTree
探讨CopyOnWrite容器,和Disruptor队列的RingBuffer


<pre>
CopyOnWrite并发容器

           CopyOnWrite并发容器用于读多写少的并发场景，比如白名单，黑名单，商品类目的访问
       和更新场景。
</pre>

<pre>
什么是CopyOnWrite并发容器

      核心思想：
           
             如果有多个调用者同时要求相同的资源，他们会共同获取相同的指针指向相同的资源，
       直到某个调用者试图修改资源内容时，系统才会真正复制一份专用副本给该调用者，而其他调用
       者锁见到的最初的资源任然保持不变。这个过程对其他调用者是透明的，此做法的主要目的是
       如果调用者没有修改资源，就不会有副本被创建，因此多个调用者只是读取操作时可以共享同
       一份副本。   
</pre>

<pre>
CopyOnWriteArrayList并发容器类：

        底层实现是数组Object[]，声明：volatile
        加锁的方式是ReentrantLock
        copy的方法是Arrays.copyOf

public E set(int index, E element) {
        final ReentrantLock lock = this.lock;
        lock.lock();//获得锁
        try {
            Object[] elements = getArray();//得到目前容器数组的一个副本
            E oldValue = get(elements, index);//获得index位置对应元素目前的值

            if (oldValue != element) {
                int len = elements.length;
                //创建一个新的数组newElements,将elements复制过去
                Object[] newElements = Arrays.copyOf(elements, len);
                //将新数组中index位置的元素替换为element
                newElements[index] = element;
                //这一步是关键，作用是将容器中array的引用指向修改之后的数组，即newElements
                setArray(newElements);
            } else {
                //index位置元素的值与element相等，故不对容器数组进行修改
                setArray(elements);
            }
            return oldValue;
        } finally {
            lock.unlock();//解除锁定
        }
    }

   在set方法中，首先是获得了当前数组的一个拷贝获得一个新的数组，然后在这个新的数组上完成
   想要的操作。当操作完成之后，再把原有数组的引用指向新的数组。并且在此过程中，只拥有一个事
   实不可变对象，即容器中的array。这样一来就很巧妙地体现了CopyOnWrite思想。
 
   其实这也是读写分离的一种体现。当线程在对线程进行读或者写的操作时，其实操作的是不同的容
   器。这么一来我们可以对容器进行并发的读，而不需要加锁。实际上就是这么做的。


CopyOnWrite容器的不足：

      1）存在内存占用的问题，因为每次对容器结构进行修改的时候都要对容器进行复制，这么一来
         就有了旧有对象和新入的对象，会占用两份内存，如果对象占用的内存过大，就会引发频繁
         的GC，降低性能。

      2）CopyOnWrite只能保证数据的最终一致性，不能保证数据的实时一致性。
</pre>

<pre>
CopyOnWrite的使用场景：
 
      适用于读多写少的场景。

           1）缓存
           2）Kafka消息
</pre>