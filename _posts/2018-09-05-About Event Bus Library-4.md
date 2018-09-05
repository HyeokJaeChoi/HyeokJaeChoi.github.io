---
layout: post
titls: Event Bus Library 분석 - 내부 코드(unregister(), post())
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
*post() 메서드 설명 추가해야됨!*