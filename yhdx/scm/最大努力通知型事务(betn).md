1、背景

ERP与WMS有各自的库存模型（两个系统技术异构），为了使两边的库存数据一致，同时降低引入技术复杂度，采用最大努力通知型事务来保证两边的数据一致。鉴于最大努力通知型事务本身只能提供比较弱的事务一致性，当两边数据不一致窗口变大时，需要人工方式补偿。



2、设计

系统架构如下：

![betn0](C:\myws\work\github\work\yhdx\scm\betn0.png)

（图片：https://github.com/april11640/work/blob/master/yhdx/scm/betn0.png）

**消息的同步过程：**

（1）、对所有要同步的消息做本地记录，消息与业务数据都存储在同一库中，保证消息的保存与业务在同一本地事务下。

（2）、消息的生产方发送消息的操作，要放在事务提交之后。这样保证只有本地的业务和记录消息的操作都成功之后，才真正的同步数据给消费方。发送消息的操作即使失败了，也没关系，后面会有消息的重发-确认机制来保证消息的送达。

（3）、消费方收到消息后，先做消息的消费幂等检验，如果消息已消费过，则直接对消息的消费做确认。如果未消费过，则插入消费日志，并进行业务操作，消费日志的插入操作和业务操作要在同一本地事务下。之后的确认消息要在本地的事务成功提交之后再进行，即使确认消息发送失败，也没关系，后面会有生产方和消费方之间的消息的重发-确认机制来保证消息的送达和确认。

（4）、生产方收到消费方发来的确认消息后，更新消息的状态为已确认。同理，更新消息状态为已确认失败，也没关系，由重发-确认机制来保证。

（5）、生产方开启一个任务计划，不断的扫面那些超过时间阈值且处于未确认状态的消息，进行重发。因为最大努力通知型事务的特点决定，系统并不能从业务的角度完全去解决生产方和消费方数据的一致性，因此重发是设置了最大尝试次数的，超过最大尝试次数，消息状态进入冻结状态，不再重发。对于数据最终一致性的修复，只能通过人工手段来解决了。任务的时间计划，可以做成多种策略，例如按照时间的阶梯计算，前3次任务执行的时间间隔是5秒，接下来的3次任务执行的时间间隔是10秒，依次类推，直到达到最大尝试次数停止对某些消息的重发。

（6）、消费方根据业务数据特点，开启一个反查任务计划，对那些状态未更新且超出时间阈值的数据进行反查。如果查到生产方有最新状态的更新消息记录（未确认状态），则更新本地的数据状态，并对消息进行消费确认。

（7）、正常情况下，大部分消息都能第一次就同步达到一致性。而生产方和消费方开启任务计划，都是尽量使得生产方和消费方数据不一致的时间窗口缩小，同时也是为了保证消息的努力送达，达到两边数据的最终一致性。

**系统架构的设计要点：**

（1）、生产方的发送消息和消费方的确认消息，都是在各自的本地事务提交成功之后执行，且成功与否都不要求（由消息的重发-确认机制来保证消息同步的最终一致性），因此这2个操作都可以做成异步的，批量的执行。

（2）、生产方消息的记录都是在本地事务保护下保存的，为了提高消息重发、确认的效率，可以把消息放入缓存（例如Redis）。这样消息的发送或重发都基于缓存查询、更新发送状态的，然后再批量更新到数据库中去；之后消息的确认，也是先更新缓存，然后再批量的更新数据库。

（3）、当系统业务量瞬间激增时，或者网络问题、消费方暂时消费慢，生产方的系统会产生大量的要同步的消息，如果都放在本地内存或缓存中等待发送，会对生产方是一个极大的压力考验。所以生产方要有消息堆积的处理能力，在生产方消息发送这一环节，加入消息队列（如Kafka）。消息推入队列，和从队列拉去消费消费的过程中，消息可能会丢失，但是有重放-确认的机制，消息的丢失不是问题。

（4）、同理，消费方的消费幂等检验，也可以基于缓存和队列来优化。

（5）、为了支持生产方和消费方之间通信手段的多样化，如http, mq, rpc等（如果仅仅是mq，那么队列的使用就是顺理成章了，建议Kafka，消息的堆积能力强；rabbitmq的消息堆积能力有点差；如果生产方和消费方全是java栈，也可以考虑rocketmq），减少业务代码的编程量，可以通过反射或者动态代理的手段，去支持各种通信方式。

3、代码解读

```java
@EventuallySend
@Transactional
public void saveDummyOrderThenSend() {
    ......
    DummyOrder dummyOrder = new DummyOrder();
	......
	collectionTemplate.collect(DummyManager.class, "syncDummyOrder",
			NotificationBuilder.create(new PayloadWrapper<>(dummyOrder))
					.setType(10)
					.setAction(1000)
					.setBizId(String.valueOf(dummyOrder.getId()));
    ......
}
```

@EventuallySend注解与@Transactional放在一起，说明当前方法业务逻辑产生的数据库写操作，与要被同步数据消息的保存，在同一个本地事务下。CollectionTemplate.collect是收集同步数据的操作，DummyManager.syncDummyOrder方法就是本地事务成功提交后，要执行的发送同步消息的方法。



**EventuallySendAspect**

```java
NotificationHandler handler = ......
Object result = null;
try {  
    handler.clear();   
    //业务逻辑（proceed）在本地事务的范围内执行   
    result = pjp.proceed();   
    //没有异常，则发送通知；或如果业务逻辑的返回值是Result类型，则只有结果为ok的情况下，才发送通知   
    boolean flag = true;  
    if((result instanceof Result) && !((Result<?>)result).wasOk()) {    
        flag = false;   
    }   
    if(flag) {  
        //发送同步数据的消息
        handler.emit();   
    }
} finally {   
    handler.clear();
}
return result;
```

以上代码是根据@EventuallySend注解的aop拦截处理逻辑。



**NotificationHandlerImpl**

```java
public void clear() {   
	PendingSendNotificationBufferHolder.clear();
}

public void emit() {
  emitService.emit(PendingSendNotificationBufferHolder.getPendingSendNotificationBuffer(), true);
}
```

NotificationHandler实现的部分逻辑，clear()清空上下文中残留的待发送消息，emit()负责把上下文中待发送的消息发送出去。



**PendingSendNotificationBufferHolder**

```java
public class PendingSendNotificationBufferHolder {    
    
    private static ThreadLocal<PendingSendNotificationBuffer> holder       = new InheritableThreadLocal<PendingSendNotificationBuffer>();  
   
    public static PendingSendNotificationBuffer getPendingSendNotificationBuffer() {  
        PendingSendNotificationBuffer buffer = holder.get();      
        if(buffer == null) {         
            buffer = new PendingSendNotificationBuffer();       
            holder.set(buffer);     
        }     
        return buffer;    
    }   
    
    public static void clear() {    
        holder.remove(); 
    }
    
}
```

PendingSendNotificationBufferHolder就是从ThreadLocal中维护待发送消息。



**PendingSendNotificationBuffer**

```java
public class PendingSendNotificationBuffer {     
    private Map<String, List<Notification>> buffer = new HashMap<>();
    ......
}
```

PendingSendNotificationBuffer其实就是一个HashMap，用来缓存待发送的消息。



**EmitServiceImpl**

```java
public void emit(PendingSendNotificationBuffer pendingSendNotificationBuffer, boolean scheduled) {  
    NextPlayTimeStrategy nextPlayTimeStrategy = null;   
    if(scheduled) {   
        //设置消息重发的策略
        String nextPlayTimeStrategyName = 							                				NextPlayTimeStrategyHelper.getNextPlayTimeStrategy(configManager); 
        nextPlayTimeStrategy = applicationContext.getBean(      
            nextPlayTimeStrategyName, NextPlayTimeStrategy.class); 
    }   
    Collection<List<Notification>> collection = 		            		  	     	     	pendingSendNotificationBuffer.listNotificationList();   
    for(List<Notification> notificationList : collection) {   
        for(Notification notification : notificationList) {     
            if(scheduled) {   
                //设置消息下一次的重发时间
                nextPlayTimeStrategy.setNextPlayTime(notification);      
            }       
        } 
        //异步发送消息
        threadPoolHelper.execute(() -> {      
            emit(notificationList, scheduled);    
        });  
    }
}

private void emit(List<Notification> notificationList, boolean scheduled) {
    for(Notification notification : notificationList) {
        if(invoke(notification)) {
            notification.setPlayTime(new Date());
        }
    }
    CollectionUtils.batchVisit(notificationList, 													CollectionUtils.DEFAULT_BATCH_VISIT_SIZE, 
            	(e) -> {
                	if(scheduled) {
                    	notificationManager.updateAllNotificationForPlayAndSchedule(e);
                    } else {
                        notificationManager.updateAllNotificationForPlay(e);
                    }
    });
}

private boolean invoke(Notification notification) {
    Metadata metadata = notification.getMetadata();
    Method method = metadata.getMethod();
    Class<?> parameterType = metadata.getParameterType();
    Object bean = null;
    Object payload = null;
    try {
        if(metadata.isContainerManaged()) {
            if(StringUtils.hasText(metadata.getContainerBeanId())) {
                bean = applicationContext.getBean(metadata.getContainerBeanId());
            } else {
                bean = applicationContext.getBean(metadata.getClazz());
            }
        } else {
            bean = metadata.getClazz().newInstance();
        }
        payload = notification.getPayload();
        if((payload instanceof String) && !Objects.equals(String.class, parameterType)) {
            payload = JsonUtils.parse(payload, parameterType);
        }
    } catch(Exception e) {
        String errorMessage = "反射创建对象或反序列化负载数据失败";
        logger.error(LogUtils.message(errorMessage, new Object[] {metadata}));
        throw new BetnException(errorMessage);
    }
    try {
        method.invoke(bean, payload);
        return true;
    } catch (IllegalAccessException | IllegalArgumentException | InvocationTargetException e) {
        String errorMessage = "反射调用方法失败：" + e.getMessage();
        logger.error(LogUtils.message(errorMessage, new Object[] {metadata}), e);
        throw new BetnException(errorMessage);
    } catch(Throwable t) {
        logger.error(LogUtils.message("方法执行异常：" + t.getMessage(), 
                                      new Object[] {metadata, payload}), t);
    }

    return false;
}
```

以上是EmitService的部分逻辑，实际上数据的同步，最后是通过反射实现的。



```java
@EventuallyAck
@Transactional
public void receiveDummyOrderThenUpdate(PayloadWrapper<DummyOrder> wrapper) {  
	if (!idempotencyTemplate.isAllowed()) {   
        return;   
    }
    ......
}
```

@EventuallyAck注解与@Transactional放在一起，保证消息幂等检验的消费日志的保存要与业务逻辑在同一本地事务下。



**EventuallyAckAspect**

```java
Objects.requireNonNull(arguments);
if(arguments.length < 1) {   
    throw new NullPointerException();
}
Object acknowledgement = arguments[0];
AcknowledgementHandler handler = ......
Object result = null;
try {   
    handler.reset(acknowledgement);   
    //消费通知的业务逻辑（proceed）在本地事务的范围内执行   
    pjp.proceed();   
    //没有异常，则发送回执；或如果业务逻辑的返回值是Result类型，则只有结果为ok的情况下，才发送回执   
    boolean flag = true;   
    if((result instanceof Result) && !((Result<?>)result).wasOk()) {   
        flag = false;  
    }   
    flag = flag & AcknowledgementHolder.getFlag();  
    if(flag) {      
        handler.ack();   
    }
} finally {   
    handler.clear();
}
return result;
```

以上代码是根据@EventuallyAck注解的aop拦截处理逻辑。



**IdempotencyCheckTemplate**

```java
AcknowledgementHandler handler = ......
    
public boolean isAllowed() {   
    return handler.checkIdempotent();
}
```

幂等性检验模板。



**AcknowledgementHandlerImpl**

```java
public void clear() {   
    AcknowledgementHolder.clear();
}

public void reset(Object bean) {
    Objects.requireNonNull(bean);
    AcknowledgementHolder.clear();
    AcknowledgementHolder.setAcknowledgement(convertToAcknowledgement(bean, true));
}

public boolean checkIdempotent() {
    Acknowledgement acknowledgment = AcknowledgementHolder.getAcknowledgement();
    if(acknowledgment == null) {
        throw new BetnException("因为方法未标注EventuallyAck注解，所以上下文没有找到回执对象，则无法检验数据的幂等性");
    }
    return checkIdempotent0(acknowledgment, true);
}

private boolean checkIdempotent0(Acknowledgement acknowledgment, boolean context) {
    String uniqueCode = acknowledgment.getUniqueCode();
    if(!StringUtils.hasText(uniqueCode)) {
        throw new BetnException("因为回执对象未设置唯一标识码uniqueCode，所以无法检验数据的幂等性");
    }
    IdempotencyLog idempotencyLog = new IdempotencyLog();
    idempotencyLog.setUniqueCode(uniqueCode);
    idempotencyLog.setCreateTime(new Date());
    int affectedRows = idempotencyLogManager.saveIdempotencyLog(idempotencyLog);
    if(context) {
        AcknowledgementHolder.setFlag(true);
    }
    return (affectedRows > 0);
}

public void ack() {
    Acknowledgement acknowledgement = AcknowledgementHolder.getAcknowledgement();
    if(acknowledgement != null) {
        echoService.ack(acknowledgement);
    }
}

private Acknowledgement convertToAcknowledgement(Object bean, boolean context) {
    Acknowledgement acknowledgement = null;
    if(bean instanceof Acknowledgement) {
        acknowledgement = (Acknowledgement)bean;
    } else {
        Class<?> clazz = bean.getClass();
        Method notifyUrlGetMethod = null;
        if(context) {
            notifyUrlGetMethod = ReflectionUtils.findMethod(clazz, "getNotifyUrl");
            if(notifyUrlGetMethod == null) {
                String message = "检验幂等性的对象必须包含getNotifyUrl方法";
                logger.error(LogUtils.message(message, new Object[] {bean, clazz}));
                throw new BetnException(message);
            }
        }
        Method uniqueCodeGetMethod = ReflectionUtils.findMethod(clazz, "getUniqueCode");
        if(uniqueCodeGetMethod == null) {
            String message = "检验幂等性的对象必须包含getUniqueCode方法";
            logger.error(LogUtils.message(message, new Object[] {bean, clazz}));
            throw new BetnException(message);
        }
        try {
            acknowledgement = new PayloadWrapper<>();
            if(context) {
                acknowledgement.setNotifyUrl((String)notifyUrlGetMethod.invoke(bean));
            }
            acknowledgement.setUniqueCode((String)uniqueCodeGetMethod.invoke(bean));
        } catch (IllegalAccessException | IllegalArgumentException | InvocationTargetException e) {
            String message = "访问检验幂等性对象的getNotifyUrl或getUniqueCode方法失败";
            logger.error(LogUtils.message(message, new Object[] {bean}));
            throw new BetnException(message, e);
        }
    }
    if(context && !StringUtils.hasText(acknowledgement.getNotifyUrl())) {
        String message = "检验幂等性的对象未设置notifyUrl的值";
        logger.error(LogUtils.message(message, new Object[] {bean}));
        throw new BetnException(message);
    }
    if(!StringUtils.hasText(acknowledgement.getUniqueCode())) {
        String message = "检验幂等性的对象未设置uniqueCode的值";
        logger.error(LogUtils.message(message, new Object[] {bean}));
        throw new BetnException(message);
    }
    return acknowledgement;
}
```

消息确认的主要处理逻辑。



**AcknowledgementHolder**

```java
public class AcknowledgementHolder {   
    
    private static ThreadLocal<Acknowledgement> acknowledgement       = new InheritableThreadLocal<Acknowledgement>();  
    
    private static ThreadLocal<Boolean> flag = new ThreadLocal<Boolean>(); 
    
    public static Acknowledgement getAcknowledgement() {    
        return acknowledgement.get();  
    }   
    
    public static void setAcknowledgement(Acknowledgement value) {   
        acknowledgement.set(value);  
    }      
    
    public static boolean getFlag() {    
        Boolean value = flag.get();    
        return value != null ? value.booleanValue() : false;  
    }  
    
    public static void setFlag(boolean value) {    
        flag.set(value);   
    }  
    
    public static void clear() {      
        acknowledgement.remove();     
        flag.remove();  
    }
    
}
```

存储确认消息的上下文。



**EchoServiceImpl**

```java
public void ack(Acknowledgement acknowledgment) {   
    threadPoolHelper.execute(() -> {      
        acknowledgementManager.send(acknowledgment);   
    });
}
```

消息的确认是异步的。