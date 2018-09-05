---
layout: post
title: Event Bus Library 분석 - 내부 코드(subscribe())
tag: [EventBus, Java, Android]
---

지난 포스팅에서는 EventBus가 이벤트를 구독할 객체에서 register()함수를 호출할 때 어떤 방식으로 객체에서 구독 메서드를 찾는지에 대해서 살펴보았습니다. 이번 포스팅에서는 register() 메서드의 마지막 코드부인 subscirbe() 메서드의 동작 방식에 대해서 살펴보겠습니다.
register() 메서드는 이벤트 구독 객체와 구독 메서드들을 찾으면 내부적으로 구독 메서드 갯수만큼 반복문을 돌려 subscribe() 메서드를 호출합니다. subscribe() 메서드는 구독 객체와 구독 메서드를 인자로 받으며 메서드가 인자로 받고 있는 `eventType`, 구독자와 구독메서드를 묶어놓은 `newSubscription`, [eventType : subscriptions] 을 [key : value] 형태로 가지고 있는 Map에서 현재 구독 메서드의 eventType을 키로 가지고 있는 subscriptions를 가져오는 `subscriptions` 이렇게 3개의 필드를 초기화 시킵니다. `eventType`의 `subscriptions` 필드가 존재하지 않을 경우 CopyOnWriteArrayList를 생성하여 위에서 언급한 맵에 새로운 엔트리로 추가합니다.
구독 메서드의 `eventType`을 가진 `subscriptions`를 찾았거나 생성한 다음에는 `subscriptions`의 사이즈 만큼 반복문을 돌면서 추가할 구독 메서드가 현재 `subscriptions.subscriberMethod` 보다 우선순위가 높을 경우 해당 메서드 앞에 추가가 되며 없을 경우 List의 맨 뒤로 추가가 됩니다.
그 후 이벤트 구독 객체가 가지고 있는 이벤트 클래스 List를 EventBus 내부에서 구독 객체와 이벤트 클래스 List를 [key : value] 형식으로 가지고 있는 Map에서 가져옵니다. 위의 설명 부분과 비슷한 로직으로 작성되어 있고 인자로 들어온 구독 객체가 기존 List의 key로 존재하지 않으면 구독 객체를 key로 하는 새로운 ArrayList를 생성하여 Map의 엔트리에 추가합니다. 엔트리가 존재할 경우 바로 엔트리의 값인 이벤트 클래스 List에 eventType을 추가합니다.
여기까지의 코드는 다음과 같습니다.
```java
// Must be called in synchronized block
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        //CopyOnWriteArrayList는 ArrayList의 한 종류로서 List의 입출력 과정에서 Thread-Safe를 보장합니다.
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);
```

여기까지가 기본적인 subscribe() 메서드가 동작하는 방식입니다. Event별로 구독 객체, 객체 내부의 구독 메서드를 추가하고 구독 객체 별로 Event클래스들의 List를 저장하여 여러개의 이벤트가 여러 구독 객체들에게 전송될 수 있도록 관리를 하고 있습니다.
기본 구독 로직 수행후에는 EventBus에서 정의한 Sticky Event에 관한 코드들이 동작합니다. EventBus 라이브러리를 제작한 greenrobot은 Sticky Event를 다음과 같이 정의하고 있습니다.
*"Some events carry information that is of interest after the event is posted. For example, an event signals that some initialization is complete. Or if you have some sensor or location data and you want to hold on the most recent values. Instead of implementing your own caching, you can use sticky events. So EventBus keeps the last sticky event of a certain type in memory. Then the sticky event can be delivered to subscribers or queried explicitly. Thus, you don’t need any special logic to consider already available data."*
위 설명에 언급되어있는 대로 Sticky Event가 발생하면 Sticky Event를 구독하고 있는 객체들은 가입과 동시에 가장 최근에 발생한 Sticky Event의 결과를 추가적인 로직 구현 없이 사용할 수 있게 됩니다(Ex. 사용자의 위치값, 센서 측정 값). subscribe() 메서드는 앞서 설명한 과정들을 완료한 후 구독 메서드가 Sticky Event를 구독하는지 확인합니다. 그리고 boolean타입의 `eventInheritance `변수의 true, false여부에 따라 다른 로직이 수행되는데 true일 경우 EventBus 객체 내부에 Sticky Event를 저장해놓은 ConcurrentHashMap타입의 `stickyEvents` 변수의 엔트리들을 Set으로 저장 후 엔트리 갯수만큼 반복하며 `eventType` 변수가 `stickyEvents`의 key와 동일한 클래스거나 상위 클래스일 경우 이를 생성한 Object인 value를 checkPostStickyEventToSubscription()메서드의 인자로 넣어 Sticky Event를 발생시킵니다. false일 경우 `eventType` 와 동일한 key에 있는 Object를 가져와 동일한 메서드를 실행시킵니다.
ConcurrentHashMap 또한 CopyOnWriteArrayList와 같이 Thread-Safe한 HashMap 구조입니다. 업로드 한 포스팅에는 보이지않지만 EventBus가 처음 초기화될시 해당 Map을 ConcurrentHashMap 형으로 초기화 시키고 있습니다. Thread-Safe에 관한 내용은 [제 포스팅](https://hyeokjaechoi.github.io/2018/09/05/About-Thread-Safe.html)을 참고하시면 도움이 될 것 같습니다.
greenrobot에서는 `eventInheritance` 변수를 `default=true`로 명시해놓고 있으며 Event 클래스의 계층을 고려한다고 명시해놓고 있습니다. 따라서 이 변수를 조작하지 않고 기본적으로 이벤트가 발생할 경우 `SuperEventClass`를 상속하는 `SubEventClass1`과 `SubEventClass2`가 있고 `SuperEventClass` 가 Sticky Event로 발송이 되면 `SubEventClass1`과 `SubEventClass2`를 sticky=true 상태로 구독하는 객체의 메서드에서는 각각 상속받은 이벤트 클래스의 로직이 register()가 실행된 후 동작하게 됩니다.
여기까지의 코드는 설명에 비해 간단합니다. 다음과 같습니다.
```java
if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
```
코드를 살펴보면 앞서 설명한 두 로직 전부다 checkPostStickyEventToSubscription() 메서드를 호출합니다. 이 메서드는 인자로 넘어온 `stickyEvent`가 null이 아닐 경우 postToSubscription() 메서드를 호출하는 간단한 로직을 수행합니다. postToSubscription() 메서드는 boolean 타입의 `isMainThread()` 인자가 추가되어 동작하는데 메서드가 동작하는 시점의 스레드가 메인 스레드인지 판별하여 true, false로 넘겨줍니다. 그 후 `subscription.subscriberMethod.threadMode`를 호출하여 현재 구독 객체 메서드의 스레드 모드를 확인하여 기본 값인 `threadMode.Posting`을 제외한 나머지 모드는 현재 구독 객체 메서드의 스레드가 설정한 스레드 모드랑 다를 시 해당하는 `Poster`의  `Poster.enqueue()` 메서드를 호출하여 작업 큐에 넣어놓는 작업을 수행합니다.
실행한 메서드 순서대로 코드는 다음과 같습니다.
```java
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
            // --> Strange corner case, which we don't take care of here.
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }
```
```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```
subscribe() 메서드의 동작이 완료되었습니다. register()가 실행하는 가입 로직을 subscirbe() 메서드가 구독 객체의 구독 메서드 만큼 실행하여 실제 가입을 시키는 것이 주된 특징으로 보입니다. 다음 포스팅에는 가입을 해지하는 unregister()와 이벤트를 발생시키는 post() 메서드에 대해 알아보겠습니다.