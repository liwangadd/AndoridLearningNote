上一篇带大家初步了解了EventBus的使用方式，详见：Android EventBus实战 没听过你就out了，本篇博客将解析EventBus的源码，相信能够让大家深入理解该框架的实现，也能解决很多在使用中的疑问：为什么可以这么做？为什么这么做不好呢？
1、概述
一般使用EventBus的组件类，类似下面这种方式：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
public class SampleComponent extends Fragment  
{  
  
    @Override  
    public void onCreate(Bundle savedInstanceState)  
    {  
        super.onCreate(savedInstanceState);  
        EventBus.getDefault().register(this);  
    }  
  
    public void onEventMainThread(param)  
    {  
    }  
      
    public void onEventPostThread(param)  
    {  
          
    }  
      
    public void onEventBackgroundThread(param)  
    {  
          
    }  
      
    public void onEventAsync(param)  
    {  
          
    }  
      
    @Override  
    public void onDestroy()  
    {  
        super.onDestroy();  
        EventBus.getDefault().unregister(this);  
    }  
      
}  

大多情况下，都会在onCreate中进行register，在onDestory中进行unregister ；
看完代码大家或许会有一些疑问：
1、代码中还有一些以onEvent开头的方法，这些方法是干嘛的呢？
在回答这个问题之前，我有一个问题，你咋不问register（this）是干嘛的呢？其实register(this)就是去当前类，遍历所有的方法，找到onEvent开头的然后进行存储。现在知道onEvent开头的方法是干嘛的了吧。
2、那onEvent后面的那些MainThread应该是什么标志吧？
嗯，是的，onEvent后面可以写四种，也就是上面出现的四个方法，决定了当前的方法最终在什么线程运行，怎么运行，可以参考上一篇博客或者细细往下看。

既然register了，那么肯定得说怎么调用是吧。
[java] view plaincopy在CODE上查看代码片派生到我的代码片
EventBus.getDefault().post(param);  

调用很简单，一句话，你也可以叫发布，只要把这个param发布出去，EventBus会在它内部存储的方法中，进行扫描，找到参数匹配的，就使用反射进行调用。
现在有没有觉得，撇开专业术语：其实EventBus就是在内部存储了一堆onEvent开头的方法，然后post的时候，根据post传入的参数，去找到匹配的方法，反射调用之。
那么，我告诉你，它内部使用了Map进行存储，键就是参数的Class类型。知道是这个类型，那么你觉得根据post传入的参数进行查找还是个事么？

下面我们就去看看EventBus的register和post真面目。
2、register
EventBus.getDefault().register(this);
首先：
EventBus.getDefault()其实就是个单例，和我们传统的getInstance一个意思：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
/** Convenience singleton for apps using a process-wide EventBus instance. */  
   public static EventBus getDefault() {  
       if (defaultInstance == null) {  
           synchronized (EventBus.class) {  
               if (defaultInstance == null) {  
                   defaultInstance = new EventBus();  
               }  
           }  
       }  
       return defaultInstance;  
   }  

使用了双重判断的方式，防止并发的问题，还能极大的提高效率。
然后register应该是一个普通的方法，我们去看看：
register公布给我们使用的有4个：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
 public void register(Object subscriber) {  
        register(subscriber, DEFAULT_METHOD_NAME, false, 0);  
    }  
 public void register(Object subscriber, int priority) {  
        register(subscriber, DEFAULT_METHOD_NAME, false, priority);  
    }  
public void registerSticky(Object subscriber) {  
        register(subscriber, DEFAULT_METHOD_NAME, true, 0);  
    }  
public void registerSticky(Object subscriber, int priority) {  
        register(subscriber, DEFAULT_METHOD_NAME, true, priority);  
    }  

本质上就调用了同一个：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
private synchronized void register(Object subscriber, String methodName, boolean sticky, int priority) {  
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriber.getClass(),  
                methodName);  
        for (SubscriberMethod subscriberMethod : subscriberMethods) {  
            subscribe(subscriber, subscriberMethod, sticky, priority);  
        }  
    }  

四个参数
subscriber 是我们扫描类的对象，也就是我们代码中常见的this;
methodName 这个是写死的：“onEvent”，用于确定扫描什么开头的方法，可见我们的类中都是以这个开头。
sticky 这个参数，解释源码的时候解释，暂时不用管
priority 优先级，优先级越高，在调用的时候会越先调用。
下面开始看代码：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriber.getClass(),  
                methodName);  

调用内部类SubscriberMethodFinder的findSubscriberMethods方法，传入了subscriber 的class，以及methodName，返回一个List<SubscriberMethod>。
那么不用说，肯定是去遍历该类内部所有方法，然后根据methodName去匹配，匹配成功的封装成SubscriberMethod，最后返回一个List。下面看代码：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass, String eventMethodName) {  
        String key = subscriberClass.getName() + '.' + eventMethodName;  
        List<SubscriberMethod> subscriberMethods;  
        synchronized (methodCache) {  
            subscriberMethods = methodCache.get(key);  
        }  
        if (subscriberMethods != null) {  
            return subscriberMethods;  
        }  
        subscriberMethods = new ArrayList<SubscriberMethod>();  
        Class<?> clazz = subscriberClass;  
        HashSet<String> eventTypesFound = new HashSet<String>();  
        StringBuilder methodKeyBuilder = new StringBuilder();  
        while (clazz != null) {  
            String name = clazz.getName();  
            if (name.startsWith("java.") || name.startsWith("javax.") || name.startsWith("android.")) {  
                // Skip system classes, this just degrades performance  
                break;  
            }  
  
            // Starting with EventBus 2.2 we enforced methods to be public (might change with annotations again)  
            Method[] methods = clazz.getMethods();  
            for (Method method : methods) {  
                String methodName = method.getName();  
                if (methodName.startsWith(eventMethodName)) {  
                    int modifiers = method.getModifiers();  
                    if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {  
                        Class<?>[] parameterTypes = method.getParameterTypes();  
                        if (parameterTypes.length == 1) {  
                            String modifierString = methodName.substring(eventMethodName.length());  
                            ThreadMode threadMode;  
                            if (modifierString.length() == 0) {  
                                threadMode = ThreadMode.PostThread;  
                            } else if (modifierString.equals("MainThread")) {  
                                threadMode = ThreadMode.MainThread;  
                            } else if (modifierString.equals("BackgroundThread")) {  
                                threadMode = ThreadMode.BackgroundThread;  
                            } else if (modifierString.equals("Async")) {  
                                threadMode = ThreadMode.Async;  
                            } else {  
                                if (skipMethodVerificationForClasses.containsKey(clazz)) {  
                                    continue;  
                                } else {  
                                    throw new EventBusException("Illegal onEvent method, check for typos: " + method);  
                                }  
                            }  
                            Class<?> eventType = parameterTypes[0];  
                            methodKeyBuilder.setLength(0);  
                            methodKeyBuilder.append(methodName);  
                            methodKeyBuilder.append('>').append(eventType.getName());  
                            String methodKey = methodKeyBuilder.toString();  
                            if (eventTypesFound.add(methodKey)) {  
                                // Only add if not already found in a sub class  
                                subscriberMethods.add(new SubscriberMethod(method, threadMode, eventType));  
                            }  
                        }  
                    } else if (!skipMethodVerificationForClasses.containsKey(clazz)) {  
                        Log.d(EventBus.TAG, "Skipping method (not public, static or abstract): " + clazz + "."  
                                + methodName);  
                    }  
                }  
            }  
            clazz = clazz.getSuperclass();  
        }  
        if (subscriberMethods.isEmpty()) {  
            throw new EventBusException("Subscriber " + subscriberClass + " has no public methods called "  
                    + eventMethodName);  
        } else {  
            synchronized (methodCache) {  
                methodCache.put(key, subscriberMethods);  
            }  
            return subscriberMethods;  
        }  
    }  
呵，代码还真长；不过我们直接看核心部分：
22行：看到没，clazz.getMethods();去得到所有的方法：
23-62行：就开始遍历每一个方法了，去匹配封装了。
25-29行：分别判断了是否以onEvent开头，是否是public且非static和abstract方法，是否是一个参数。如果都复合，才进入封装的部分。
32-45行：也比较简单，根据方法的后缀，来确定threadMode，threadMode是个枚举类型：就四种情况。
最后在54行：将method, threadMode, eventType传入构造了：new SubscriberMethod(method, threadMode, eventType)。添加到List，最终放回。
注意下63行：clazz = clazz.getSuperclass();可以看到，会扫描所有的父类，不仅仅是当前类。
继续回到register:
[java] view plaincopy在CODE上查看代码片派生到我的代码片
for (SubscriberMethod subscriberMethod : subscriberMethods) {  
            subscribe(subscriber, subscriberMethod, sticky, priority);  
        }  

for循环扫描到的方法，然后去调用suscribe方法。
[java] view plaincopy在CODE上查看代码片派生到我的代码片
// Must be called in synchronized block  
   private void subscribe(Object subscriber, SubscriberMethod subscriberMethod, boolean sticky, int priority) {  
       subscribed = true;  
       Class<?> eventType = subscriberMethod.eventType;  
       CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);  
       Subscription newSubscription = new Subscription(subscriber, subscriberMethod, priority);  
       if (subscriptions == null) {  
           subscriptions = new CopyOnWriteArrayList<Subscription>();  
           subscriptionsByEventType.put(eventType, subscriptions);  
       } else {  
           for (Subscription subscription : subscriptions) {  
               if (subscription.equals(newSubscription)) {  
                   throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "  
                           + eventType);  
               }  
           }  
       }  
  
       // Starting with EventBus 2.2 we enforced methods to be public (might change with annotations again)  
       // subscriberMethod.method.setAccessible(true);  
  
       int size = subscriptions.size();  
       for (int i = 0; i <= size; i++) {  
           if (i == size || newSubscription.priority > subscriptions.get(i).priority) {  
               subscriptions.add(i, newSubscription);  
               break;  
           }  
       }  
  
       List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);  
       if (subscribedEvents == null) {  
           subscribedEvents = new ArrayList<Class<?>>();  
           typesBySubscriber.put(subscriber, subscribedEvents);  
       }  
       subscribedEvents.add(eventType);  
  
       if (sticky) {  
           Object stickyEvent;  
           synchronized (stickyEvents) {  
               stickyEvent = stickyEvents.get(eventType);  
           }  
           if (stickyEvent != null) {  
               // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)  
               // --> Strange corner case, which we don't take care of here.  
               postToSubscription(newSubscription, stickyEvent, Looper.getMainLooper() == Looper.myLooper());  
           }  
       }  
   }  
我们的subscriberMethod中保存了method, threadMode, eventType，上面已经说了；
4-17行：根据subscriberMethod.eventType，去subscriptionsByEventType去查找一个CopyOnWriteArrayList<Subscription> ，如果没有则创建。
顺便把我们的传入的参数封装成了一个：Subscription（subscriber, subscriberMethod, priority）；
这里的subscriptionsByEventType是个Map，key：eventType ； value：CopyOnWriteArrayList<Subscription> ； 这个Map其实就是EventBus存储方法的地方，一定要记住！
22-28行：实际上，就是添加newSubscription；并且是按照优先级添加的。可以看到，优先级越高，会插到在当前List的前面。
30-35行：根据subscriber存储它所有的eventType ； 依然是map；key：subscriber ，value：List<eventType> ;知道就行，非核心代码，主要用于isRegister的判断。
37-47行：判断sticky；如果为true，从stickyEvents中根据eventType去查找有没有stickyEvent，如果有则立即发布去执行。stickyEvent其实就是我们post时的参数。
postToSubscription这个方法，我们在post的时候会介绍。

到此，我们register就介绍完了。
你只要记得一件事：扫描了所有的方法，把匹配的方法最终保存在subscriptionsByEventType（Map，key：eventType ； value：CopyOnWriteArrayList<Subscription> ）中；
eventType是我们方法参数的Class，Subscription中则保存着subscriber, subscriberMethod（method, threadMode, eventType）, priority；包含了执行改方法所需的一切。

3、post
register完毕，知道了EventBus如何存储我们的方法了，下面看看post它又是如何调用我们的方法的。
再看源码之前，我们猜测下：register时，把方法存在subscriptionsByEventType；那么post肯定会去subscriptionsByEventType去取方法，然后调用。
下面看源码：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
/** Posts the given event to the event bus. */  
   public void post(Object event) {  
       PostingThreadState postingState = currentPostingThreadState.get();  
       List<Object> eventQueue = postingState.eventQueue;  
       eventQueue.add(event);  
  
       if (postingState.isPosting) {  
           return;  
       } else {  
           postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();  
           postingState.isPosting = true;  
           if (postingState.canceled) {  
               throw new EventBusException("Internal error. Abort state was not reset");  
           }  
           try {  
               while (!eventQueue.isEmpty()) {  
                   postSingleEvent(eventQueue.remove(0), postingState);  
               }  
           } finally {  
               postingState.isPosting = false;  
               postingState.isMainThread = false;  
           }  
       }  
   }  
currentPostingThreadState是一个ThreadLocal类型的，里面存储了PostingThreadState；PostingThreadState包含了一个eventQueue和一些标志位。
[java] view plaincopy在CODE上查看代码片派生到我的代码片
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {  
       @Override  
       protected PostingThreadState initialValue() {  
           return new PostingThreadState();  
       }  
   }  

把我们传入的event，保存到了当前线程中的一个变量PostingThreadState的eventQueue中。
10行：判断当前是否是UI线程。
16-18行：遍历队列中的所有的event，调用postSingleEvent（eventQueue.remove(0), postingState）方法。
这里大家会不会有疑问，每次post都会去调用整个队列么，那么不会造成方法多次调用么？
可以看到第7-8行，有个判断，就是防止该问题的，isPosting=true了，就不会往下走了。

下面看postSingleEvent
[java] view plaincopy在CODE上查看代码片派生到我的代码片
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {  
        Class<? extends Object> eventClass = event.getClass();  
        List<Class<?>> eventTypes = findEventTypes(eventClass);  
        boolean subscriptionFound = false;  
        int countTypes = eventTypes.size();  
        for (int h = 0; h < countTypes; h++) {  
            Class<?> clazz = eventTypes.get(h);  
            CopyOnWriteArrayList<Subscription> subscriptions;  
            synchronized (this) {  
                subscriptions = subscriptionsByEventType.get(clazz);  
            }  
            if (subscriptions != null && !subscriptions.isEmpty()) {  
                for (Subscription subscription : subscriptions) {  
                    postingState.event = event;  
                    postingState.subscription = subscription;  
                    boolean aborted = false;  
                    try {  
                        postToSubscription(subscription, event, postingState.isMainThread);  
                        aborted = postingState.canceled;  
                    } finally {  
                        postingState.event = null;  
                        postingState.subscription = null;  
                        postingState.canceled = false;  
                    }  
                    if (aborted) {  
                        break;  
                    }  
                }  
                subscriptionFound = true;  
            }  
        }  
        if (!subscriptionFound) {  
            Log.d(TAG, "No subscribers registered for event " + eventClass);  
            if (eventClass != NoSubscriberEvent.class && eventClass != SubscriberExceptionEvent.class) {  
                post(new NoSubscriberEvent(this, event));  
            }  
        }  
    }  

将我们的event，即post传入的实参；以及postingState传入到postSingleEvent中。
2-3行：根据event的Class，去得到一个List<Class<?>>；其实就是得到event当前对象的Class，以及父类和接口的Class类型；主要用于匹配，比如你传入Dog extends Dog，他会把Animal也装到该List中。
6-31行：遍历所有的Class，到subscriptionsByEventType去查找subscriptions；哈哈，熟不熟悉，还记得我们register里面把方法存哪了不？
是不是就是这个Map;
12-30行：遍历每个subscription,依次去调用postToSubscription(subscription, event, postingState.isMainThread);
这个方法就是去反射执行方法了，大家还记得在register，if(sticky)时，也会去执行这个方法。
下面看它如何反射执行：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {  
        switch (subscription.subscriberMethod.threadMode) {  
        case PostThread:  
            invokeSubscriber(subscription, event);  
            break;  
        case MainThread:  
            if (isMainThread) {  
                invokeSubscriber(subscription, event);  
            } else {  
                mainThreadPoster.enqueue(subscription, event);  
            }  
            break;  
        case BackgroundThread:  
            if (isMainThread) {  
                backgroundPoster.enqueue(subscription, event);  
            } else {  
                invokeSubscriber(subscription, event);  
            }  
            break;  
        case Async:  
            asyncPoster.enqueue(subscription, event);  
            break;  
        default:  
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);  
        }  
    }  
前面已经说过subscription包含了所有执行需要的东西，大致有:subscriber, subscriberMethod（method, threadMode, eventType）, priority；
那么这个方法：第一步根据threadMode去判断应该在哪个线程去执行该方法；
case PostThread:
[java] view plaincopy在CODE上查看代码片派生到我的代码片
void invokeSubscriber(Subscription subscription, Object event) throws Error {  
          subscription.subscriberMethod.method.invoke(subscription.subscriber, event);  
｝  

直接反射调用；也就是说在当前的线程直接调用该方法；
case MainThread:
首先去判断当前如果是UI线程，则直接调用；否则： mainThreadPoster.enqueue(subscription, event);把当前的方法加入到队列，然后直接通过handler去发送一个消息，在handler的handleMessage中，去执行我们的方法。说白了就是通过Handler去发送消息，然后执行的。
 case BackgroundThread:
如果当前非UI线程，则直接调用；如果是UI线程，则将任务加入到后台的一个队列，最终由Eventbus中的一个线程池去调用
executorService = Executors.newCachedThreadPool();。
 case Async:将任务加入到后台的一个队列，最终由Eventbus中的一个线程池去调用；线程池与BackgroundThread用的是同一个。
这么说BackgroundThread和Async有什么区别呢？
BackgroundThread中的任务，一个接着一个去调用，中间使用了一个布尔型变量handlerActive进行的控制。
Async则会动态控制并发。

到此，我们完整的源码分析就结束了，总结一下：register会把当前类中匹配的方法，存入一个map，而post会根据实参去map查找进行反射调用。分析这么久，一句话就说完了~~
其实不用发布者，订阅者，事件，总线这几个词或许更好理解，以后大家问了EventBus，可以说，就是在一个单例内部维持着一个map对象存储了一堆的方法；post无非就是根据参数去查找方法，进行反射调用。

4、其余方法
介绍了register和post；大家获取还能想到一个词sticky，在register中，如何sticky为true，会去stickyEvents去查找事件，然后立即去post；
那么这个stickyEvents何时进行保存事件呢？
其实evevntbus中，除了post发布事件，还有一个方法也可以：
[java] view plaincopy在CODE上查看代码片派生到我的代码片
public void postSticky(Object event) {  
       synchronized (stickyEvents) {  
           stickyEvents.put(event.getClass(), event);  
       }  
       // Should be posted after it is putted, in case the subscriber wants to remove immediately  
       post(event);  
   }  

和post功能类似，但是会把方法存储到stickyEvents中去；
大家再去看看EventBus中所有的public方法，无非都是一些状态判断，获取事件，移除事件的方法；没什么好介绍的，基本见名知意。

好了，到此我们的源码解析就结束了，希望大家不仅能够了解这些优秀框架的内部机理，更能够体会到这些框架的很多细节之处，并发的处理，很多地方，为什么它这么做等等。
