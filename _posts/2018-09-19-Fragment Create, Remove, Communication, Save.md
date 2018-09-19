---
layout: post
title: Android Fragment 생성, 제거와 데이터 통신, 기존 데이터 저장 하는법
tags: Android, Fragment
---

지난 포스팅에서는 Fragment의 생명주기에 대해서 살펴보았습니다. 이번 포스팅에서는 Fragment간에 데이터를 서로 공유하는 방법과 Fragment생성 및 제거, Fragment가 다른 Fragment로 전환되어 제거된 후 다시 생성되었을 때 어떻게 재사용 할 수 있는지 살펴보겠습니다.

1. Fragment 생성 및 제거

Fragment를 생성, 제거하기 위해선 부모 Activity에서 FragmentManager와 FragmentTransaction을 생성해야합니다. FragmentManager는 getSupportFragmentManager() 메서드를 통해서 가져올 수 있고 FragmentTransaction은 FragmentManager.beginTransaction() 메서드를 통해서 가져올 수 있습니다.
위의 두개의 객체를 얻어오면 firstFragment = new FirstFragment(), secondFragment = new SecondFragment() 로 객체를 생성한 다음 FragmentTransaction.add(R.id.container, firstFragment)로 빈 Fragment에 추가할 수 있으며 FragmentTransaction.replace(R.id.container, secondFragment) 메서드로 교체할 수 있습니다. 제거할 경우에는 FragmentTransction.remove(secondFragment)메서드를 사용하여 Activity 상에서 제거할 수 있습니다.
### 중요한 점은 모든 작업을 한 후에는 반드시 Transaction.commit() 메서드를 호출하여 작업결과를 Activity에 반영해야 합니다.

2. Fragment간 데이터 통신 방법

하나의 Activity에 2개의 Fragment가 존재하고 있다면 이 두 Fragment들은 Activity를 통해서 데이터를 주고받을 수 있습니다. 대표적인 방법으로 데이터를 전달해줄 Fragment에서 static interface를 선언하여 interface내에 콜백 메서드 함수를 정의한 후 부모 Activity에서 interface를 상속하여 구현하는 방법이 있습니다.
코드로 예시를 들어보면
```java
public class FirstFragment extends Fragment {
	//...이전코드
	
    public static interface TextChangeCallback{
    	public void onTextChanged(String text);
    }
    
    public TextChangeCallback textChangeCallback;
    private final String message = "Send";
    
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        // Inflate the layout for this fragment
        Log.d(TAG, "onCreateView");
        View rootView = inflater.inflate(R.layout.fragment_second, container, false);
        increamentButton = rootView.findViewById(R.id.increase_number_second_fragment);
        increamentButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                textChangeCallback.onTextChanged(message);
            }
        });
        return rootView;
    }
    
    //...다음코드
}
```
이런 식으로 콜백 메서드를 호출할 수 있습니다. Activity는 이 콜백메서드가 있는 interface를 상속받아 다음과 같이 구현할 수 있습니다.
```java
public class MainActivity extends AppCompatActivity implements FirstFragment.TextChangeCallback{
	//...이전 코드
    
    @Override
    public void onTextChanged(String text){
    	FragmentManager fm = getSupportFragmentManager();
        SecondFragment secondFragment = (SecondFragment)fm.findFragmentById(R.id.second_fragment);
        secondFragment.receiveMessage(text);
    }
}
	//...다음 코드
```
```java
public class SecondFragment extends Fragment {
	//...이전 코드
    public void receiveMessage(String message){
    	textView.setText(message);
    }
    //...다음 코드
}
```
Activity에서는 콜백 메서드를 구현하여 secondFragment에 있는 receiveMessage()함수를 호출하며 실제로 secondFragment에 있는 textView의 값을 변경해주고 있습니다.
위 코드 처럼 interface를 사용하여 통신하는 경우에는 Fragment가 적을때는 괜찮겠지만 통신해야할 Fragment의 갯수가 많아질 경우 코드가 복잡해지는 단점이 있습니다. 따라서 [EventBus](https://github.com/greenrobot/EventBus) 같은 컴포넌트 간 통신을 용이하게 해주는 라이브러리를 사용하면 더 쉽게 Fragment간 데이터 전송을 할 수 있습니다. EventBus 라이브러리는 [제 포스팅](https://hyeokjaechoi.github.io/2018/09/03/About-Event-Bus-Library-1.html/)을 보시면 상세한 분석 내용과 사용방법이 나와있습니다.

3. Fragment 재사용

Android는 스마트폰의 back키를 누르면 현재 Activity가 종료되고 이전에 마지막으로 표시되었던 Activity로 돌아갈 수 있도록 Activity Stack을 시스템에서 관리하고 있습니다. Fragment는 이 스택을 부모 Activity에서 관리하며 FragmentTransaction.addToBackStack() 메서드를 사용하여 이 메서드 이전에 일어났던 모든 트랜잭션들을 하나의 트랜잭션으로 만들어 프래그먼트 트랜잭션 스택에 넣습니다. addToBackStack() 메서드 이후 commit()을 실행한 뒤 다른 Fragment로 전환되면 Fragment 생명주기 메서드가 onDestroyView() 까지만 호출되는 것을 확인할 수 있습니다. 또한 back키를 눌러 해당 Fragment로 복귀할 경우 onCreateView()부터 시작하는 것을 확인할 수 있습니다. 로그 사진은 다음과 같습니다.
![Fragment Backstack]({{ site.baseurl }}/assets/img/pexels/Fragment_BackStack_Added.png)
이를 살펴보면 Fragment가 처음 실행될 시에는 onAttach() 부터 실행이 되지만 addToBackStack()을 실행하고 commit()을 할 경우 위에서 설명한 것처럼 onCreateView() ~ onDestroyView() 까지의 생명주기를 가지는 것으로 확인이 됩니다.
이처럼 addToBackStack()을 할 경우 Fragment 객체가 보존은 되지만 다른 Fragment로 전환 후 다시 돌아올 경우 onCreateView() 부터 시작하기 때문에 이전에 변경한 View의 내용이 초기화 됩니다. 부모 Activity가 같은 Fragment끼리는 서로 전환할 경우 onSavedInstanceState() 메서드가 호출되지 않기 때문에 이 메서드 내의 Bundle을 사용하여 저장할 경우 데이터가 제대로 저장이 되지 않습니다. 따라서 onPause() 혹은 onStop() 메서드에서 저장을 해줘야 하는데 이 메서드들은 Bundle을 매개변수로 가지고 있지 않아서 그냥 저장할 수가 없습니다.
###### 이에 대한 해결책으로 Fragment가 가지고 있는 메서드인 getArguments()메서드를 사용하여 Bundle을 따로 가져와서 저장을 해야합니다.이 메서드에서 가져오는 Bundle은 Fragment 생명주기 내에서 인자로 위치하고 있는 savedInstanceState 매개변수가 아니므로 이점에 유의해서 사용하시기 바랍니다.
저는 Fragment 전환 시 onPause() 메서드에서 현재 TextView의 문자열을 저장하고 onActivityCreated() 메서드에서 Bundle을 가져와서 저장되어있는 문자열로 복원시키고 있습니다. 코드는 다음과 같습니다.
```java
@Override
    public void onPause() {
        String savedText = textView.getText().toString();
        getArguments().putString("FirstTextView", savedText);
        Log.d(TAG, "onPause" + " " + savedText);
        super.onPause();
    }
```
```java
@Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        String savedText = getArguments().getString("FirstTextView");
        if(getArguments().containsKey("FirstTextView")){
            textView.setText(savedText);
        }
        Log.d(TAG, "onActivityCreated" + " " + savedText);
        super.onActivityCreated(savedInstanceState);
    }
```
코드에서 보시다시피 onPause()에서는 getArguments.putString() 메서드를 사용하여 key-value 타입으로 현재 TextView의 문자열 값을 저장하고 재생성시 호출되는 onActivityCreated() 메서드에서 저장한 값이 있는 키가 있으면 이를 TextView에 넣어주고 있습니다.

지금까지 Fragment 생성 및 제거, Fragment간 통신하는 방법, Fragment 객체 및 내부 데이터 저장, 복원 하는 방법 등에 대해서 살펴보았습니다. 다음 포스팅에서는 마지막으로 Fragment 전환 애니메이션 적용 방법 및 기본 설정들에 대해서 알아보겠습니다.