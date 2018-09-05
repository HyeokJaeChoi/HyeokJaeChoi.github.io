---
layout: post
title: (Java) Reflection이란?
tags: [Java, Reflection]
---

EventBus 라이브러리 분석 중에 Reflection이란 개념이 나왔습니다. Java에서 사용하고 있는 기법 중 하나로 특정 클래스의 정확한 정보를 모르는 상태에서 해당 클래스의 메서드와 필드, 생성자 등 클래스 구성 요소들을 찾고 사용할 수 있습니다. 예시로 다음과 같은 코드는 java.util.Stack 클래스에 있는 메서드 리스트를 Reflection을 사용하여 얻습니다.

```java
import java.lang.reflect.*;
 
   public class DumpMethods {
      public static void main(String args[])
      {
         try {
            Class c = Class.forName(args[0]);
            Method m[] = c.getDeclaredMethods();
            for (int i = 0; i < m.length; i++)
            System.out.println(m[i].toString());
         }
         catch (Throwable e) {
            System.err.println(e);
         }
      }
   }
```
위 예제는 Oracle에서 Reflection에 대한 예제 코드입니다. `c` 변수가 `main()` 함수의 첫번째 인자를 이름으로 가지고 있는 클래스를 찾고 `m[]` 배열은 클래스의 정의된 메서드를 `getDeclaredMethods()`를 통해서 가져오고 메서드명을 출력하고 있습니다.

결과는 다음과 같습니다.
```shell
public java.lang.Object java.util.Stack.push(
    java.lang.Object)
   public synchronized 
     java.lang.Object java.util.Stack.pop()
   public synchronized
      java.lang.Object java.util.Stack.peek()
   public boolean java.util.Stack.empty()
   public synchronized 
     int java.util.Stack.search(java.lang.Object)
```
이처럼 특정 클래스 명을 넣을 경우 Java에 내장된 Reflection API를 사용하여 메서드, 필드, 메서드의 인자 타입 등을 알 수 있습니다. 
Reflection은 EventBus 라이브러리 분석 중에 새롭게 알게된 개념으로 EventBus 객체에서 해당 Activity나 Fragment가 이벤트를 구독할 수 있도록 가입시킬때 register()메서드를 사용하는데 이 메서드에서 내부적으로 가입하는 클래스에서 구독을 표시한 메서드를 찾을 때 메서드 이름, 인자 타입 등을 모르므로 Reflection을 활용하여 이를 구독 메서드에 추가시키는 기능을 합니다. 
Reflection을 잘 활용하면 Object 타입의 객체를 사용하여 동일한 작업을 수행하는 로직을 수행할 수 있을 것 같습니다. EventBus에서 Reflection을 적용한 코드는 [제 포스팅](https://hyeokjaechoi.github.io/2018/09/04/About-Event-Bus-Library-2.html)에 있으니 참고하시면 좋을 것 같습니다. 또한 Reflection 공식 문서는 [이 곳](https://www.oracle.com/technetwork/articles/java/javareflection-1536171.html)을 참고하시면 됩니다.