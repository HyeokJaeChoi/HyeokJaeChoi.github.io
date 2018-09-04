---
layout: post
title: Event Bus Library 분석 - 역할 및 동작 방식
---
Android에서는 Activity <-> Activity, Activity <-> Fragment 같은 컴포넌트들끼리 통신을 하는 방법으로 Intent등을 통하여 서로의 값을 공유하는 방식이 있습니다. 하지만 통신해야할 컴포넌트의 갯수가 늘어날수록 코드가 복잡해지며 송수신 주체를 한번에 알아보기 어려운 단점이 있습니다. 이를 해결하기 위한 라이브러리로 greenrobot의 [EventBus](https://github.com/greenrobot/EventBus) 라이브러리가 있으며 이를 활용하면 어느 코드에서든지 특정 이벤트를 발생시킬 수 있으며 이를 수신할 클래스에서는 이벤트를 구독할 메서드를 정의하여 이벤트 발생 시 해당 메서드를 바로 실행할 수 있습니다.

![Image of a glass on a book]({{ site.baseurl }}/assets/img/pexels/EventBus-Publish-Subscribe.png)

Android 기준 EventBus를 사용하려면 AndroidProject의 app수준 build.gradle에 다음과 같은 의존성을 추가해야합니다.
```gradle
implementation 'org.greenrobot:eventbus:3.1.1'
```

EventBus에서는 POJO 형태의 어디에도 의존하지 않는 Event 클래스를 선언하여 발생할 이벤트를 정의하고 있습니다. 아래는 예시로 작성한 SendMessageEvent 클래스 입니다.
```java
public static class SendMessageEvent {
	private String message;
    
    public SendMessageEvent(String message){
    	this.message = message;
    }
    
    public String getMessage(){
    	return this.message;
    }
}
```

Event 클래스를 정의하고 나면 해당 이벤트를 받을 클래스에서 이벤트를 구독할 메서드를 정의해야 합니다. EventBus에서는 @Subscribe 어노테이션을 선언하여 이벤트 구독 메서드임을 나타냅니다. 추가로 @Subscribe 어노테이션을 선언할 때 메서드가 실행될 쓰레드를 설정할 수 있으며 threadMode = ThreadMode.MAIN 과 같은 Value 타입으로 선언할 수 있습니다. 선언하지 않을 시 이벤트를 보내는 메서드와 같은 쓰레드에서 호출되는 ThreadMode.POSTING으로 설정됩니다.

```java
//Android UI Component를 조작하므로 MainThread(UI Thread)에서 실행
@Subscribe(threadMode = ThreadMode.MAIN)
public void onSendMessageEvent(SendMessageEvent event){
	mTextView.setText(event.getMessage());
}
```

EventBus의 이벤트를 받기 위해서는 @Subscribe 어노테이션 설정 뿐만 아니라 해당 구독자를 이벤트에 가입시켜 이벤트를 구독중임을 알려야 합니다. EventBus의 공식 레퍼런스에서는 Activity, Fragment의 생명주기에 맞춰서 가입과 해지를 할 것을 명시하고 있습니다. 가입과 해지는 EventBus 싱글톤 객체의 register()와 unregister() 메서드로 실행할 수 있으며 Object형식의 객체를 받습니다.

```java
@Override
public void onStart(){
	super.onStart();
    EventBus.getDefault().register(this);
}

@Override
public void onStop(){
	EventBus.getDefault().unregister(this);
    super.onStart();
}
```

위와 같이 onStart(), onStop() 에서 가입과 해지를 선언할 경우 Activity, Fragment가 사라지기 전까지 이벤트를 수신할 수 있습니다.

여기까지 작성이 완료되면 이벤트 수신부는 정의가 완료되었습니다. 이제 이벤트를 보낼 다른 클래스에서 이벤트를 호출하는 메서드를 실행하면 이벤트를 구독하고 있는 메서드로 전송이 됩니다. 이벤트 호출은 register(), unregister() 메서드와 마찬가지로 EventBus 싱글톤 객체안에 있는 post() 메서드로 실행하며 동일하게 Object형식의 객체를 받습니다.

```java
public void sendMessage(String message){
	EventBus.getDefault().post(new SendMessageEvent(message));
}
```

EventBus를 활용하면 이와 같이 Android Component간의 통신이 훨씬 간결해지고 명확해지는 장점이 있습니다. 다음 포스팅에는 EventBus 라이브러리 내부를 살펴보면서 자세한 동작 정보를 확인해보겠습니다.