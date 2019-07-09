1、背景

redis的访问，除了基本的操作之外，系统会有一些通用的功能需求：

（1）、redis是一个键值数据库，通过键访问数据，所以要保证数据的键是全局唯一的。

（2）、存储到redis的数据可能会涉及到对象序列化。

（3）、数据访问操作的调试、追踪、日志记录。

（4）、键和值的校验。

（5）、批量操作的统一处理。

（6）、大键的处理。

（7）、空值处理。

（8）、分布式锁。



2、设计

在基本的操作之上，如get, set, mget, mset等，增加一些额外的功能。最先想到就是通过模板方法去设计解决。Spring提供了RedisTemplate和StringRedisTemplate，在此基础上进一步的封装。



3、代码

**模板工具**

```java
/**
 * 访问redis的模板工具
 */
public interface OperationTemplate {

    /**
     * 获取应用空间
     * @return
     */
    String getNamespace();

    /**
     * 设置应用空间
     *
     * @param value
     */
    void setNamespace(String value);

    /**
     * 获取业务领域名称
     * @return
     */
    String getDomain();

    /**
     * 设置业务领域名称
     * @param value
     */
    void setDomain(String value);

    /**
     * 获取批量操作的记录数
     *
     * @return
     */
    int getMultiOperationBatchSize();

    /**
     * 设置批量操作的记录数
     *
     * @param value
     */
    void setMultiOperationBatchSize(int value);

    /**
     * 是否开启键过期时间的上下浮动特性
     *
     * @return
     */
    boolean isEnableKeyExpiredVolatility();

    /**
     * 设置开启键过期时间的上下浮动特性
     *
     * @return
     */
    void setEnableKeyExpiredVolatility(boolean value);
}

/**
 * 访问redis的抽象模板工具
 */
public abstract class AbstractOperationTemplate<K, V> implements OperationTemplate{

    /**
     * 键的类型
     */
    protected final Class<K> keyType;
    /**
     * 值的类型
     */
    protected final Class<V> valueType;
    /**
     * redis访问工具
     */
    protected final StringRedisTemplate stringRedisTemplate;
    /**
     * 应用空间
     */
    protected String namespace;
    /**
     * 数据的领域名称，例如订单、商品等
     */
    protected String domain;
    /**
     * 批量操作的数据记录条数
     */
    protected int multiOperationBatchSize = 100;
    /**
     * 开启键过期时间的上下浮动特性
     */
    protected boolean enableKeyExpiredVolatility = true;
    /**
     * 对访问操作的包装，例如对于redis的访问操作增加计数、监控等行为
     */
    protected final Operations operations;
    /**
     * 数据的转换工具，例如json转换
     */
    protected final StringConverter<V> converter;

    protected final RedisSerializer<String> keySerializer;
    protected final RedisSerializer<String> valueSerializer;
    
    ......
        
    /**
     * 检验键的合法性
     *
     * @param key
     */
    protected void validateKey(K key) {
        Objects.requireNonNull(key, "The key is null.");
        KeyUtils.checkType(key.getClass());
    }

    /**
     * 检验键的合法性
     *
     * @param keys
     */
    protected void validateKey(Collection<K> keys) {
        if(CollectionUtils.isEmpty(keys)) {
            throw new NullPointerException("The keys is null or empty.");
        }
        for(K key : keys) {
            validateKey(key);
        }
    }

    /**
     * 包装键值，把key包装为"namespace:domain:key"的格式
     *
     * @param key
     * @return
     */
    protected final <TKey> String wrapKey(TKey key) {
        return KeyUtils.wrapKey(namespace, domain, key);
    }

    /**
     * 包装键值，把key包装为"namespace:domain:key"的格式
     *
     * @param keys
     * @return
     */
    protected final List<String> wrapKey(Collection<K> keys) {
        List<String> keyList = new ArrayList<>();
        for(K key : keys) {
            keyList.add(wrapKey(key));
        }
        return keyList;
    }
    
    ......
}
```

以上是访问redis的模板接口和抽象类，主要定义了构成键的namespace, domain，以及键值类型等变量。



```java
public class StringOperationTemplate<K, V> extends AbstractOperationTemplate<K, V> {
    ......
    
    public void set(K key, V value) {
        set(key, value, 0);
    }

    public void set(K key, V value, long seconds) {
        validateKey(key);
        String value0 = converter.serialize(value);
        setString(wrapKey(key), value0, seconds);
    }

    protected final void setString(String key, String value, long seconds) {
        operations.execute(SET, getKeyPrefix(), Arrays.asList(key),
                () -> {
                    if(seconds < 1) {
                        valueOps.set(key, value);
                    } else {
                        valueOps.set(key, value,
                                seconds + getExpiredSecondsVolatility(seconds),
                                TimeUnit.SECONDS);
                    }
                }, value, seconds);
    }    
        
    ......
}
```

以上是StringOperationTemplate模板的set的相关方法的逻辑。

所有的模板工具如下：

StringOperationTemplate：字符串的基本操作访问。

StringListOperationTemplate：在字符串的基础上，存储java的List对象的操作访问。

HashOperationTemplate：hash的基本操作访问。

HashListOperationTemplate：在hash的基础上，值是java的List对象的操作访问。

HashBucketOperationTemplate：在hash的基础上，把不同的值hash到不同的bucket中（每个bucket就是一个hash）。

ListOperationTemplate：list的基本操作访问。

SetOperationTemplate：set的基本操作访问。

ZSetOperationTemplate：zset的基本操作访问。



**一个简单的分布式锁的实现**

```java
.......
    
public boolean tryLock(String object, long timeoutMillis) {
    Objects.requireNonNull(object);
    String key = getKey(object);
    String taskKey = getTaskKey(object);
    long timeout = timeoutMillis;
    synchronized (lockHolder.getLockObject(object)) {
        for(;;) {
            String expireTime = String.valueOf(System.currentTimeMillis() + expireMillis + 1);
            if(stringTemplate.setIfAbsent(key, expireTime)) {
                //成功设置key的值为一个时间戳，表示成功得到锁，则直接返回锁
                stringTemplate.set(taskKey, getTaskId());
                return true;
            }
            // 没有拿到锁，则获取key的时间戳，判断其是否小于当前系统时间？如果是，说明锁已经过期，则通过redis
            // 的getset方法设置新的时间戳，并返回旧的时间戳；如果旧的时间戳小于当前系统时间，则成功得到锁，
            // 否则可能是其它线程先获取了锁并设置了新的时间戳；在并发的情况下，当前线程即使没有成功获取锁，但是
            // 修改了时间戳且多计数了，考虑到并发情况下先后时间比较短，可忽略不考虑。
            String expireTimeFound = stringTemplate.get(key);
            if(expireTimeFound == null || Long.parseLong(expireTimeFound) < System.currentTimeMillis()) {
                expireTime = String.valueOf(System.currentTimeMillis() + expireMillis + 1);
                String oldExpireTime = stringTemplate.getAndSet(key, expireTime);
                if(oldExpireTime == null || Long.parseLong(oldExpireTime) < System.currentTimeMillis()) {
                    stringTemplate.set(taskKey, getTaskId());
                    return true;
                }
            }
            long delta = DEFAULT_ACQUIRE_RESOLUTION_MILLIS - random.nextInt(DEFAULT_ACQUIRE_RESOLUTION_MILLIS);
            timeout -= delta;
            if(timeout < 1) {
                break;
            }
            //休眠一定时间，继续尝试获取锁
            try {
                Thread.sleep(delta);
            } catch (InterruptedException e) {
                //do nothing
            }
        }
        return false;
    }
}

public void unlock(String object) {
    Objects.requireNonNull(object);
    String key = getKey(object);
    String taskKey = getTaskKey(object);
    synchronized (lockHolder.getLockObject(object)) {
        //为了防止误删别的任务的锁，需要根据任务标识来判断锁是否是当前任务的。
        String taskId = stringTemplate.get(taskKey);
        if (Objects.equals(taskId, getTaskId())) {
            stringTemplate.delete(key);
            stringTemplate.delete(taskKey);
        }
    }
}
    
.......
```

