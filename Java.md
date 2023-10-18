#### Q：HashSet和TreeSet有什么区别？

HashSet是由一个hash表来实现的，因此，它的元素是无序的。add()，remove()，contains()方法的时间复杂度是O(1)。
另一方面，TreeSet是由一个树形的结构来实现的，它里面的元素是有序的。因此，add()，remove()，contains()方法的时间复杂度是O(logn)。

#### Q：Java中Exception和Error有什么区别？
Exception和Error都是Throwable的子类。Exception用于用户程序可以捕获的异常情况。Error定义了不期望被用户程序捕获的异常。

#### HashMap和Hashtable的区别。
HashMap是Hashtable的轻量级实现（非线程安全的实现），他们都完成了Map接口，主要区 别在于HashMap允许空（null）键值（key）,由于非线程安全，效率上可能高于Hashtable。HashMap允许将null作为一个entry的key或者value，而Hashtable不允许。HashMap把Hashtable的contains方法去掉了，改成containsvalue和containsKey。因为contains方法容 易让人引起误解。Hashtable继承自Dictionary类，而HashMap是Java1.2引进的Map interface的一个实现。最大的不同是，Hashtable 的方法是Synchronize的，而HashMap不是，在多个线程访问Hashtable时，不需要自己为它的方法实现同步，而HashMap 就必须为之 提供外同步。Hashtable和HashMap采用的hash/rehash算法都大概一样，所以性能不会有很大的差异。

