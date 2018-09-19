---
layout: post
title: Fragment Transition Animation(전환 애니메이션)
tags: Android, Fragment, Transition
---

지난 포스팅에서는 Fragment의 생성과 제거, Fragment간 통신 방법, Fragment 객체와 내부 데이터 저장 및 복원 하는 방법에 대해서 포스팅하였습니다. 지난 포스팅과 Fragment Lifecycle을 정리해놓은 포스팅을 살펴보면 Fragment의 대략적인 개념에 대해 이해하실 수 있을겁니다.

이번 포스팅에서는 Fragment가 전환되면 기존 Fragment가 없어지고 새로운 Fragment가 생기는 그 과정에 애니메이션 효과를 적용할 수 있는 방법에 대해서 알아보겠습니다.
[Android Fragment 공식문서](https://developer.android.com/guide/components/fragments?hl=ko#Transactions)에서는 Fragment 전환 간에 애니메이션을 적용해야되는 경우 commit() 메서드 호출 전에 setTransition() 메서드를 호출하라고 명시되어있습니다.
```
팁: 각 프래그먼트 트랜잭션에 대해 전환 애니메이션을 적용하려면 커밋하기 전에 setTransition()을 호출하면 됩니다.
//Android Fragment 공식 문서
```
setTransition() 메서드는 transit 이라는 int 타입의 값을 받고있는데 이 값으로 FragmentTransaction 클래스 내부에 있는 상수값을 사용하여 기본적인 전환 효과를 사용할 수 있습니다. 만약 Custom Animation을 사용하고 싶은 경우 setTransitionStyle() 메서드를 호출하여 인자값으로 R.anim.custom_animation 타입으로 넣어주면 됩니다.
그럼 기본 값으로는 어떤 효과들을 사용할 수 있는지 살펴보겠습니다.

`TRANSIT_FRAGMENT_OPEN`
전환되어지는 Fragment가 가로, 세로로 살짝 줄어드는 효과와 함께 다음 Fragment가 표시됩니다.
`TRANSIT_FRAGMENT_CLOSE`
전환되어지는 Fragment가 사라지며 새로운 Fragment의 가로, 세로가 넓어지는 효과와 함께 표시됩니다.
`TRANSIT_FRAGMENT_FADE`
전환되어지는 Fragment가 어둡게 사라지면서 새로운 Fragment가 전체 영역에 서서히 나타나는 Fade In, Out 효과와 함께 표시됩니다.
`TRANSIT_NONE`
전환 애니메이션 효과를 주지 않습니다.

위의 기본 애니메이션 효과만 사용해도 Fragment 전환이 좀 더 깔끔하고 역동적으로 변하는 것을 확인할 수 있습니다.
이번 포스팅에서는 Fragment의 전환 애니메이션에 대해서 살펴보았습니다. 다음 포스팅에서는 그동안 소개했던 Fragment들을 활용하는 방법들을 사용하여 간단한 어플리케이션을 만들어 본 후 기능 소개 및 동작 영상을 첨부해 보겠습니다.