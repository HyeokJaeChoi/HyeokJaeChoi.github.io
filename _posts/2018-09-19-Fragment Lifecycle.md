---
layout: post
title: Android Fragment 생명주기
tags: Android, Fragment
---

Android 3.0 이후 버전에서는 태블릿과 같은 넓은 디스플레이에서 여러 화면을 분할하여 각각 독립적인 UI를 구성하기 위해 Fragement라는 개념이 나왔습니다. Android 공식 문서에서는 Fragment를 이렇게 정의하고 있습니다.
```
Fragment는 동작 또는 Activity 내에서 사용자 인터페이스의 일부를 나타냅니다.
여러 개의 프래그먼트를 하나의 액티비티에 조합하여 창이 여러 개인 UI를 구축할 수 있으며, 하나의 프래그먼트를 여러 액티비티에서 재사용할 수 있습니다.
프래그먼트는 자체 수명 주기를 가지고, 자체 입력 이벤트를 받으며, 액티비티 실행 중에 추가 및 제거가 가능한 액티비티의 모듈식 섹션이라고 생각하면 됩니다(다른 액티비티에 재사용할 수 있는 "하위 액티비티"와 같은 개념).
```
즉 Fragment는 Activity 에 의존하는 수명주기를 가진 UI Container & Manager 라고 볼 수 있겠습니다. 그러면 Fragment는 어떤 생명 주기를 가지고 동작하는지 살펴보겠습니다.

1. OnAttach(Context context)

Fragment가 Activity에 포함되어 속해졌을 때 호출됩니다. Context를 인자로 받아오는데 이를 사용하면 Activity에 있는 콜백 리스너 등을 가져와서 Activity <-> Fragment 통신이 가능하도록 하게 해줍니다.

2. OnCreate(Bundle saveInstanceState)

Fragment가 처음 생성된 후 onAttach()가 끝나면 이 메서드가 호출됩니다. Activity.onCreate()와 마찬가지로 Fragment의 초기 설정들을 해줄 수 있으나 주의해야 될 점은 View와 관련된 설정은 할 수 없으며 이는 다음에 나오는 onCreateView() 함수에서 설정해야 합니다. 또한 Fragment 객체가 소멸되지 않도록 하는 함수인 [setRetainInstance()](https://developer.android.com/reference/android/support/v4/app/Fragment.html#setRetainInstance(boolean))가 true로 설정되어 있을 때 또한 호출되지 않으므로 이 부분에서는 Fragment가 완전히 처음 생성되었을때 초기화 해야되는 것들로만 구성해야 합니다.

3. onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState)

Fragment에만 있는 생명주기 메서드로 Activity.onCreate() 처럼 View와 관련된 초기화를 해줄 수 있습니다. 주의해야할 점은 Activity 에서 View를 초기화 할때 쓰는 메서드인 findViewById()가 Fragment에는 없으므로 rootView = inflater.inflate() 메서드로 Fragment UI를 View형태로 만든 후 rootView.findViewById()로 View를 초기화 할 수 있습니다.

4. onViewCreated(View view, Bundle saveInstanceState)

onCreateView()가 호출된 후 바로 호출되며 Fragment 내부의 View들이 초기화된 다음 시점입니다. 해당 Fragment 내부에 이 Fragment의 View 계층을 알아야 하는 클래스가 있을 경우 여기서 view 인자를 통하여 정보를 획득 할 수 있습니다.

5. onActivityCreated(Bundle saveInstanceState)

Activity.onCreate()가 호출이 된 후 Activity에서 Fragment 인스턴스가 포함되면 호출되는 메서드이며 Fragment가 새로 만들어지거나 reAttach가 됐을 때만 호출되며 단순히 다른 Activity 밑으로 가려질 경우 호출되지 않습니다. Fragment에 저장해

6. onViewStateRestored(Bundle saveInstanceState)

onActivityCreated()가 호출된 후 onStart()로 넘어가기 전에 바로 호출되며 Fragment 소멸전 저장했던 상태들이 복원된 후 호출됩니다. 이 메서드는 onActivityCreated()가 호출되지 않으면 호출되지 않습니다.

7. onStart() ~ onStop()

Activity.onStart()가 실행되면 같이 호출됩니다. 여기서부터 onStop() 까지는 Activity와 생명주기를 같이하게 됩니다.

8. onSaveInstanceState(Bundle outstate)

Activity.onSaveInstanceState()처럼 Fragment 재생성 시 저장되어야할 Bundle 정보들을 여기서 저장할 수 있습니다. Android 공식 문서에서는 여기서 저장한 정보들을 onCreate(), onCreateView(), onActivityCreated() 등에서 복원할 수 있다고 하며 개인적인 추천은 Fragment가 Activity에 완전히 포함되어 인스턴스화 된 시점인 onActivityCreated()에서 복원할 것을 추천드립니다.

9. onDestroyView()

onCreateView()로 생성된 View들이 Fragment로부터 분리되고 해제될때 호출됩니다. onDestroy() 호출전에 호출됩니다.

10. onDestroy()

Fragment가 더이상 사용되지 않을 시 호출되며 onCreate()와 마찬가지로 setRetainInstance() 가 true로 설정되어 있으면 이 또한 호출되지 않습니다.

11. onDetach()

Activity로부터 Fragment가 완전히 분리되었을때 호출됩니다.

위 생명주기 메서드들의 동작을 확인하고 싶으면 생명주기 메서드마다 Log.d()를 호출하여 Fragment전환, Fragment가 활성화된 상태에서 새로운 Activity 생성, Configuration 변경 (Ex. 단말기 가로 / 세로 전환) 등을 확인할 수 있을 것 같습니다.
지금까지 Fragment의 전체적인 생명주기 흐름을 파악해 보았습니다. 기능이 추가된 때로부터 지금까지 많은 버그가 고쳐지면서 이제는 Activity와 똑같은 (Fragment만의 고유 생명주기 메서드를 제외한) 생명주기 흐름을 가지고 있다고 봐도 과언이 아닐 듯 합니다. 다음 포스팅에는 Fragment생성, 제거방법과 Fragment간의 통신방법, 기존 Fragment 데이터 유지 방법, Fragment 전환(Transition)에 대해서 소개해보겠습니다.