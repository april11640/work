

1、背景

一般应用代码分层包括：外部接口层（facade层），web层（处理与页面请求的一些逻辑），业务逻辑层（service层），通用逻辑层（manager层），数据访问层（dao层）等（见《阿里巴巴Java开发手册》）。层与层之间是通过对象的方法接口调用来传递参数和返回值的。在一次请求响应过程中，一般系统会存在一些通用的信息，需要在各层之间传递，例如用户的登录信息、数据库分片信息等。如果定义为方法参数来传递，则会对方法的代码严重“污染”，因为每一个方法除了定义必要的入参之外，还要定义额外的参数；而且如果增加了新的各层要共享的数据，则会导致方法接口的修改。为了解决这个问题，一种是定义一个请求生命周期的上下文对象context，作为一个数据root，负责所有层之间数据的捆绑传递，但是不够优雅，因为仍然要作为方法的一个入参，而且也会被新的需求影响而不断修改。另一种，就是利用处理当前请求的线程本地变量来完成，即ThreadLocal。为什么用ThreadLocal，因为它可以做到在各层中（实际上是各对象调用之间）隐式的传递或共享数据，而不用破坏方法接口。而且在需求功能的扩展方面，通过aop等手段，不会对既有的代码进行修改，只用代码扩展的方式就可以搞定。



2、原理及内存泄漏的问题

ThreadLocal是通过ThreadLocalMap数据结构来存储Thread的本地变量的。

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. 
 */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

TheadLocalMap的条目Entry扩展了WeakReference，且使用ThreadLocal作为key。

```java
static class ThreadLocalMap {    
/**     
 * The entries in this hash map extend WeakReference, using     
 * its main ref field as the key (which is always a     
 * ThreadLocal object).  Note that null keys (i.e. entry.get()     
 * == null) mean that the key is no longer referenced, so the     
 * entry can be expunged from table.  Such entries are referred to    
 * as "stale entries" in the code that follows.     
 */    
    static class Entry extends WeakReference<ThreadLocal<?>> {        
        /** The value associated with this ThreadLocal. */       
        Object value;        
        Entry(ThreadLocal<?> k, Object v) {          
            super(k);           
            value = v;       
        }   
    }
    
    ......
}
```

因为Thread的theadLocals变量就是ThreadLocalMap类型，而应用中通常用的都是线程池；线程使用完毕后，是放回池中，不会被垃圾回收的，那么线程的变量也是不会被垃圾回收的。因此一般在ThreadLocal使用完毕后，要手动清除ThreadLocal变量。否则，ThreadLocalMap中的条目因为是WeakReference（其key就是ThreadLocal本身），当每次GC后，WekReference的key就为null了，那么map中其对应的value就无从查询。而value是强引用，线程池的Thread一直反复使用的话，那么value就不会被回收，就可能造成内存泄漏的问题。



3、InheritableThreadLocal的陷阱

InheritableThreadLocal继承了ThreadLocal，提供了一些父子线程间传递数据的功能。

```java
public class UserContext {      
    private final static ThreadLocal<Long> currentWarehouseId = new InheritableThreadLocal<Long>();     
    private final static ThreadLocal<String> currentWarehouseName = new InheritableThreadLocal<String>();   
    private final static ThreadLocal<Long> currentUserId = new InheritableThreadLocal<Long>();   
    private final static ThreadLocal<String> currentUserName = new InheritableThreadLocal<String>();  
    ......
}
```

这里用到了InheritableThreadLocal，而不是ThreadLocal。因为在有些功能的开发中，用到了异步操作，需要在线程间传递数据。但是InheritableThreadLocal的使用方式有一个很大的坑。具体分析代码来了解过程：

Thread.init方法

```java
......
if (inheritThreadLocals && parent.inheritableThreadLocals != null)    this.inheritableThreadLocals =        ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
......
```

在线程创建时，初始化方法init中，如果父线程的inheritableThreadLocals不为null，则创建赋值给子线程的inheritableThreadLocals变量。一般应用中使用的都是线程池，也就是父子线程之间的数据传递时发生在子线程创建的时候，而之后的子线程反复的使用，inheritableThreadLocals的数据并没有被重新创建赋值，那么就可能产生了”脏读“的问题。例如子线程创建的时候，inheritableThreadLocals赋值了A用户的登录信息，而后子线程的重复使用，可能是B用户的请求，从子线程inheritableThreadLocals取到的却是A用户的数据。

解决这个问题，使用https://github.com/whaon/transmittable-thread-local来代替。



4、代码示例：

```java
public class UserSessionInterceptor extends HandlerInterceptorAdapter {
    
    ......
        
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws
            Exception {
        ......
        //此处做了清空
        UserContext.clear();
        ......
        UserContext.setWarehouseId(userSession.getWarehouseId());
        UserContext.setWarehouseName(userSession.getWarehouseName());
        UserContext.setUserId(userSession.getUserId());
        UserContext.setUserName(userSession.getUserName());
		......
    }
    
    //TODO 源码中没有重写afterCompletion，存在风险
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {
        UserContext.clear();
    }
    
}
```

UserSessionInterceptor的代码，没重写afterCompletion方法，是个潜在的风险。



5、FastThreadLocal

ThreadLocalMap的hash冲突解决采用的是线性探测方案，如下set的部分代码所示（get类似）：

```java
Entry[] tab = table;
int len = tab.length;
int i = key.threadLocalHashCode & (len-1);

for (Entry e = tab[i];
     e != null;
     e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();

    if (k == key) {
        e.value = value;
        return;
    }

    ......
```

如果key发生了冲突后，就不断的尝试table表中是否有对应的key，效率比较低。netty是采用FastThreadLocal来解决的，dubbo中也有类似的机制。一般在ThreadLocal使用中，通过减少ThreadLocal变量个数的使用，以减少hash的冲突，比如UserContext改为以下代码（这个是凌指导写的）：

```java
public class CurrentContextHolder {    
    private static ThreadLocal<Map<String, Object>> PARAM = new ThreadLocal<Map<String, Object>>();    
    public static void put(String key, Object value) {        
        Map<String, Object> map = PARAM.get();        
        if (map == null) {           
            map = new HashMap<>();           
            PARAM.set(map);       
        }        
        map.put(key, value);   
    }    
    public static Object get(String key) {       
        Map<String, Object> map = PARAM.get();       
        if (map == null) {           
            return null;       
        }        
        return map.get(key);   
    }    
    public static <T> T get(String key, Class<T> clazz) {     
        T value = (T) get(key);       
        return value;  
    }   
    public static void remove(String key) {       
        Map<String, Object> map = PARAM.get();      
        if (map != null) {            
            map.remove(key);       
        }   
    }    
    public static void clear() {       
        PARAM.remove();   
    }
}
```



6、参考：

（1）、JDK源码

（2）、**Java多线程——ThreadLocal源码解析**  https://yq.aliyun.com/articles/648286

（3）、**ThreadLocal内存泄漏真因探究**  https://www.jianshu.com/p/a1cd61fa22da

（4）、**FastThreadLocal**  https://www.jianshu.com/p/17e6989d647a