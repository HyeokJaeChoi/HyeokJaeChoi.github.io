---
layout: post
title: (Java) Thread-Safe란?
---

Java에서는 기본적으로 [멀티 스레딩](https://ko.wikipedia.org/wiki/%EB%A9%80%ED%8B%B0%EC%8A%A4%EB%A0%88%EB%94%A9)을 지원하며 Thread 클래스를 통해 스레드를 생성, 동작 시킬 수 있습니다. 스레드는 한 프로세스 내에서 여러개 생성이 가능하며 작업을 분산시켜 처리 속도를 증가시키는 장점이 있습니다.
하지만 공유되는 자원을 여러 스레드에서 접근시 예상하는 값과 호출되는 값이 다른 상황이 발생할 수 있습니다. 코드로 간단한 예를 들어보면
```java

class TestModel {
    int money = 100;

    public void spendMoney(){
        int spend = (int)(Math.random() * 3 + 1) * 10;
        if(money >= spend) {
            try {
                System.out.println("출금 전 잔액 : " + money + " " + Thread.currentThread().getName());
                Thread.sleep(500);
                this.money -= spend;
                System.out.println("출금 후 잔액 : " + money + " " + Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

class ThreadTest implements Runnable {
    TestModel testModel = new TestModel();

    @Override
    public void run() {
        while(testModel.money > 0) {
            testModel.spendMoney();
        }
    }
}

public class Test1 {
    public static void main(String[] args){
        ThreadTest threadTest = new ThreadTest();
        Thread th1 = new Thread(threadTest);
        Thread th2 = new Thread(threadTest);

        th1.start();
        th2.start();
    }
}

```
*참고코드 : [http://limkydev.tistory.com/64](http://limkydev.tistory.com/64)
위의 코드를 실행하면 잔액이 0원이 되면 실행을 멈춰야 합니다. 하지만 코드 실행 결과는 다음과 같습니다.
![Non-thread-safe]({{ site.baseurl }}/assets/img/pexels/Non-Thread-Safe.png)

이처럼 스레드 간 특정 자원을 공유해서 접근하여 수정할 경우 자원에 관한 상태가 부정확하여 보장받지 못하는 상태 (Non-Thread-Safe)가 발생합니다. 이런 경우를 가장 쉽게 해결하는 방법은 synchronized 키워드를 사용하여 한 스레드가 해당 작업을 수행 중에는 다른 스레드에서 접근을 하지 못하도록 블럭처리를 하는 방법이 있습니다. 코드는 다음과 같이 수정됩니다.
```java

class TestModel {
    int money = 100;

    public synchronized void spendMoney(){
        int spend = (int)(Math.random() * 3 + 1) * 10;
        if(money >= spend) {
            try {
                System.out.println("출금 전 잔액 : " + money + " " + Thread.currentThread().getName());
                Thread.sleep(500);
                this.money -= spend;
                System.out.println("출금 후 잔액 : " + money + " " + Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

class ThreadTest implements Runnable {
    TestModel testModel = new TestModel();

    @Override
    public void run() {
        while(testModel.money > 0) {
            testModel.spendMoney();
        }
    }
}

public class Test1 {
    public static void main(String[] args){
        ThreadTest threadTest = new ThreadTest();
        Thread th1 = new Thread(threadTest);
        Thread th2 = new Thread(threadTest);

        th1.start();
        th2.start();
    }
}

```

위의 코드에서는 스레드간 공유해서 사용하고 있는 money 필드를 수정하는 메서드 spendMoney() 메서드에 synchronized 키워드를 사용하여 특정 스레드가 메서드를 사용중일 때는 다른 스레드에서 접근을 하지 못하게 하는 block 처리를 하였습니다.
하지만 synchronized 키워드는 다른 스레드의 접근 자체를 블로킹 하기 때문에 공유자원을 사용하는 스레드가 늘어날수록 성능 저하가 심해집니다. Android에서는 대표적인 예로 특정 기능을 제공하는 싱글톤 객체 생성 시 싱글톤 객체가 synchronized 키워드를 사용해서 구현되어있으면 여러 스레드에서 접근 시 객체 생성 및 반환을 하는 동안 해당 스레드 이외에 다른 스레드는 싱글톤 객체를 얻을 수 없으므로 다른 방식으로 코드를 개선해야 합니다. 싱글톤 객체 디자인 패턴에 관한 내용은 [여기](https://medium.com/@joongwon/multi-thread-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C%EC%9D%98-%EC%98%AC%EB%B0%94%EB%A5%B8-singleton-578d9511fd42) 를 참고해보시면 좋을 것 같습니다.
Thread-Safe 개념은 최근에 EventBus 라이브러리를 분석하는 도중 EventBus 객체 또한 싱글톤 패턴이 적용되어 있었고 싱글톤 객체는 Thread-Safe 해야 접근 간 성능이 보장됨을 확인하여 알아보게 되었습니다. 멀티스레딩 프로그래밍이 필수인 최근 개발 트렌드에 따라 Thread-Safe 개념을 정립하고 가는 것도 좋을 것 같습니다.