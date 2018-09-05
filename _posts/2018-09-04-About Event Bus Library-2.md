---
layout: post
title: Event Bus Library 분석 - 내부 코드(getDefault(), register())
---

지난 포스팅에는 EventBus 라이브러리가 하는 역할과 사용 방법에 대해서 살펴보았습니다. EventBus의 주요 특징으로 register(), unregister() 메서드로 이벤트를 구독과 해지하며 @Subscribe 어노테이션이 명시된 메서드에서 이벤트 수신처리 및 관련 코드를 작성하게 됩니다. 그러면 EventBus의 핵심 기능이라 할 수 있는 register()와 unregister()는 어떤 과정을 통해서 동작하는지 한번 살펴보겠습니다.

먼저 위의 메서드들은 전부 EventBus의 싱글톤 객체를 통해서 접근할 수 있습니다. EventBus객체는 EventBus.getDefault() 메서드를 통해서 접근할 수 있으며 getDefault() 메서드는 내부의 instance가 null 상태일때 synchronized 블록을 사용하여 호출하는 시점에서의 쓰레드로 실행을 보장하는 Thread-Safe 상태를 만들어주고 다시 instance를 체크하여 null일 경우 생성자를 통해 생성하고 이를 반환합니다. Thread-Safe 관련 글은 [제 포스팅](https://hyeokjaechoi.github.io/2018/09/05/About-Thread-Safe.html)을 참고하시면 도움이 될 것 같습이다.
코드는 다음과 같습니다.
```java
/** Convenience singleton for apps using a process-wide EventBus instance. */
    public static EventBus getDefault() {
        EventBus instance = defaultInstance;
        if (instance == null) {
            synchronized (EventBus.class) {
                instance = EventBus.defaultInstance;
                if (instance == null) {
                    instance = EventBus.defaultInstance = new EventBus();
                }
            }
        }
        return instance;
    }
```

이렇게 싱글톤 객체에 접근한 후 register(), unregister() 메서드를 호출할 수 있게 됩니다.

앞서 명시했던 것처럼 register() 메서드는 이벤트를 수신할 클래스를 구독자로 만들어주는 역할을 합니다. 인자로는 Object를 인자로 받으며 이벤트 수신 객체 자체를 넘겨받습니다. Object를 인자로 받기 때문에 Activity, Fragment 처럼 모든 타입의 객체를 제한없이 받을 수 있습니다.
register() 메서드는 실행이 되면 인자로 받은 Object의 클래스 타입과 구독하고 있는 메서드들을 가져오고 synchronized 블록을 통해 Thread-Safe 상태를 만든 후 메서드의 개수만큼 내부의 subscribe() 메서드를 수행합니다. 코드는 다음과 같습니다.

```java
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

register() 메서드에서 가입 객체의 구독 메서드를 찾을때 subscriberMethodFinder.findSubscriberMethods() 메서드를 사용하는데, 이 메서드가 실행되면 객체 이름과 구독 메서드를 캐싱해놓은 Map을 먼저 참조하여 해당 객체와 메서드 리스트가 존재하면 이를 반환하고 존재하지 않을시 EventBus의 boolean타입의 ignoreGenaratedIndex의 값에 따라 true일 경우 findUsingReflection(), false일 경우 findUsingInfo() 메서드를 호출하여 구독 메서드를 찾습니다. findUsingReflection()와 findUsingInfo() 메서드는 register()함수를 호출한 클래스가 최상위 클래스에 도달할때 까지 findUsingReflectionInSinleClass() 를 호출하여 @Subscribe 어노테이션이 있는 메서드를 찾고 이를 subscriberMethod 타입의 List에 저장하고 반환하여 subscriberMethodFinder의 캐시 Map에 저장한 후 최종적으로 List를 반환하여 subscribe() 메서드의 인자로 넘어가게 됩니다.

findSubscriberMethods()의 코드는 다음과 같습니다.
```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

앞에서 설명한 캐시 담당 맵인 METHOD_CACHE가 구독 메서드 리스트를 최초로 받아올 시 저장 후 캐싱하고 있습니다.

findUsingInfo()와 findUsingReflection()의 코드는 다음과 같습니다.
```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

두 메서드의 차이점은 findUsingInfo() 메서드는 기존에 subscriberInfo가 null이 아닐 경우 내부의 getSubscriberMethods를 호출하여 subscriberInfo가 가지고 있는 구독 메서드들을 가져옵니다.

findUsingReflectionInSingleClass()의 코드는 다음과 같습니다.
```java
private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```
findUsingReflectionInSingleClass() 메서드는 Java의 Reflection 이라는 개념을 사용하였습니다. Reflection관련 글은 [제 포스팅](#)을 참고하시면 도움이 될 것 같습니다.
EventBus 라이브러리에서 요구하는 구독 메서드는 @Subscribe 어노테이션과 Event Class 단일 인자를 가지고 있어야 한다는 것을 보여주는 코드입니다. 

처음 register() 메서드가 실행될 때 구독 메서드를 찾는 부분을 살펴보았습니다. 오픈소스 라이브러리 분석은 이번에 처음 해보는 분야이지만 확실히 라이브러리가 사용하고 있는 비즈니스 로직을 분석하는 것은 본인의 실력에 많은 도움이 된다고 생각합니다. 게시글을 보시는 분들도 자신이 자주 쓰는 오픈소스 라이브러리들을 한번 분석해보는 것도 좋은 경험이 될거라 생각합니다. 다음 게시글에서는 register() 메서드가 구독 메서드들을 찾은 후 내부적으로 구독을 시키는 subscribe() 함수에 대해 알아보겠습니다.