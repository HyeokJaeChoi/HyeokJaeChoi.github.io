---
layout: post
title: Event Bus Library 분석 - 내부 코드(unregister(), post())
tags: [EventBus, Java, Android]
---

지난 포스팅에는 EventBus 내부의 subscribe() 메서드가 어떻게 구독 객체와 구독 메서드들을 하나씩 실제로 가입시키는지에 대해 확인해보았습니다. 이번 포스팅에는 가입을 해지하는 unregister()와 이벤트를 발송시키는 post() 메서드에 대해서 살펴보겠습니다.

unregister() 메서드는 register() 메서드에 비해 비교적 간단한 로직을 가지고 있습니다. 먼저 구독 객체별 구독 메서드 List 들을 저장하는 내부의 Map 타입의 `typesBySubscriber` 변수에서 가입을 해제할 객체를 key로 가지고 있는 메서드 List 들을 가져옵니다. 그 후 List의 끝에 도달할 때 까지 unsubscribeByEventType() 메서드를 호출하여 구독 객체에 있는 모든 구독 메서드들을 해지합니다. 모든 메서드가 구독해지가 완료되면 `typesBySubscriber.remove()` 메서드를 호출하여 해당 객체 이름의 [key:value] 엔트리를 삭제합니다. unsubscribeByEventType() 메서드도 간단한 로직으로 구현되어 있습니다. 이벤트 클래스와 해당 이벤트를 구독하는 객체와 메서드로 구성되어있는 `Subscription`이 [key:value]로 있는 Map 타입의 `subscriptionByEventType` 변수를 참조하여 해당 이벤트를 key로 가지고 있는 `subscriptions` List를 가져옵니다. 가져온 후에는 `subscriptions` List 내의 `subscription.subscriber`와 인자로 받은 구독 객체가 일치할 경우 `subscription.active = false` 를 실행하여 구독을 받지 않는 객체로 상태변경을 한 후 `subscriptions` List 에서 제거를 합니다. 코드는 다음과 같습니다.
```java
/* Unregisters the given subscriber from all event classes. */
    public synchronized void unregister(Object subscriber) {
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            typesBySubscriber.remove(subscriber);
        } else {	//register가 안되어있는 상태에서 unregister()를 호출할 경우 log를 찍어줍니다.
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```
```java
/* Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
    private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    //subscriptions가 CopyOnWriteArrayList타입, 즉 ArrayList 상태여서 remove 후 자동으로 인덱스 갱신이 이루어지지 않아 --연산을 통하여 해결해줍니다.
                    i--;
                    size--;
                }
            }
        }
    }
```

두개의 내부 변수에서 이벤트 구독 현황 관리를 진행하고 있고 두 변수 모두에서 구독 객체와 구독 메서드 삭제를 해주고 있습니다.
이벤트 수신부가 동작 시키는 모든 코드는 전부 둘러보았습니다. 이벤트 수신 객체가 register(), unregister()로 가입을 하고 @Subscribe 어노테이션도 구독 메서드에 달아야 하는 수고가 있었다면 이벤트 발생은 정말 간단합니다. 이벤트를 발생시킬 클래스 내부의 어떤 함수부, 어떤 코드부에서든지 `EventBus.getDafault.post()` 메서드를 호출하여 post()함수에 발생시킬 이벤트 클래스를 생성하여 넘겨주기만 하면 여태까지 설명한 수신 관련 코드들이 동작하게 됩니다. 그럼 post() 메서드는 어떤 방식으로 이벤트를 전송하는지 살펴보겠습니다.
post() 메서드는 생성시킬 이벤트 객체를 단일 인자로 받아 내부에 `ThreadLocal<PostingThreadState>` 타입의 `postingState` 변수를 얻고 `postingState.eventQueue` 를 호출하여 이벤트 객체 인자를 넣어줍니다. ThreadLocal은 앞서 설명했던 CopyOnWriteArrayList, ConcurrentHashMap 타입과 같이 Thread-Safe한 입출력을 보장하기 위해 쓰는 객체타입입니다. ThreadLocal 타입으로 선언된 변수가 여러 스레드간 공유하는 자원이 될 경우 각 스레드간 접근 시 해당 자원을 스레드의 로컬 변수로 복사하여 작업 스레드 고유의 값으로 사용할 수 있도록 합니다. ThreadLocal의 설명과 예제, 메서드는 [오라클 공식 doc](https://docs.oracle.com/javase/7/docs/api/java/lang/ThreadLocal.html/)을 참고하시면 되겠습니다.
이벤트 큐에 객체를 추가한 후에는 큐가 비워질때까지 큐의 앞에서부터 차례대로 내부 메서드인 postSingleEvent()를 호출하여 이벤트를 보내는 로직을 수행하게 하고 이벤트 전송이 완료된 경우 `postingState.isPosting` 을 false로 설정하여 다시 이벤트 전송이 가능한 상태로 만들어 줍니다. 코드는 다음과 같습니다.
```java
public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = isMainThread();
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
```

postSingleEvent() 메서드는 이벤트 객체 인자와 `postingState` 변수를 인자로 받아 이벤트 객체의 클래스 타입과 이벤트를 보낼 `subscription`을 찾았는지 확인하는 `subscriptionFound` 를 false로 초기화합니다. 그후 앞서 봤던 이벤트를 상위 클래스까지 전파할지 여부를 결정하는 `eventInheritance` 변수가 true일시 내부 static 함수인 lookupAllEventTypes를 호출하여 인자로 받은 객체의 상위 이벤트 클래스와 인터페이스를 전부 얻어와서 `subscriptionFound |= postSingleEventForEventType()` 연산을 수행합니다. false일 시는 인자로 받은 객체를 바로 postSingleEnvetForEventType()의 결과를 `subscriptionFound` 변수에 대입합니다. 이벤트를 보낼 `subscription`을 찾지 못한 경우 해당 이벤트를 구독한 객체가 없다는 에러 메세지를 출력해 줍니다. 코드는 다음과 같습니다.
```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```

코드에서 볼 수 있듯이 post() 메서드가 실패했을 경우의 예외처리를 이 메서드에서 수행하고 있으며 이벤트 전송 로직은 postSingleEventForEventTypes() 메서드가 담당하고 있는 것을 확인할 수 있었습니다.

postSingleEventForEventTypes() 메서드는 앞서 확인했던 이벤트 클래스 별 `subscriptions` 들을 저장해놓는 `subscriptionsByEventType` 에서 `subscriptions`를 가지고 옵니다. 그리고 `subscriptions` List가 끝날때까지 앞서 Sticky Event 분석 부분에서 봤던 postToSubscription() 메서드를 호출하여 실제 post() 메서드가 수행하는 이벤트 발송 로직을 수행합니다. 해당 이벤트를 구독하는 `subscriptions`가 null 상태이거나 List가 비어있는 경우에는 false를 반환하여 postSingleEvent() 메서드의 예외 처리 관련 로직을 수행하게 합니다. 코드는 다음과 같습니다.
```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
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
            return true;
        }
        return false;
    }
```

EventBus에서 단순하게 이벤트를 발생시키게 제공해준 post() 메서드가 어떤 과정을 거쳐 작동하는지 확인해보았습니다. 오픈소스 라이브러리들이 단순하게 제공해주는 메서드들이 실제로는 작성 언어가 가진 기본 자료구조들을 최대한 효율적으로 활용하여 작성한 것을 확인하고 나니 작동 방식을 알고 쓰는 거소가 모르고 쓰는 것은 버그를 고치고 디버깅 하는 과정에서 큰 차이가 있을 것을 느꼈습니다. EventBus 라이브러리 분석은 여기까지이고 다음 포스팅에는 또다른 오픈소스 라이브러리들을 들고 와보겠습니다.