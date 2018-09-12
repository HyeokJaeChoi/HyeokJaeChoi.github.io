---
layout: post
title: Event Bus Library 활용 어플리케이션 제작
tags: EventBus, Android, Java
---

지난 주 금요일부터 3일동안 저번 포스팅들에서 분석했던 Event Bus Library를 사용하여 간단한 어플리케이션을 제작해보았습니다. UI 작업을 위주로 하였으며 메인 화면 상단에는 ViewPager를 사용하여 이미지 슬라이딩을 할 수 있는 영역이 있고 밑으로는 RecyclerView가 있어서 오른쪽 하단 플로팅 버튼을 누르면 나오는 색깔 입력 화면에서 입력한 것들이 표시가 됩니다. 색깔 입력 화면에서는 앱 구동간 입력한 색깔, 수량 정보들이 버튼으로 표시되어 있고 +버튼을 누르면 입력할 수 있는 폼이 생성이 되고 입력 후에는 Event Bus Library를 사용하여 메인 화면에 입력한 정보를 바탕으로 색깔이 생성될 수 있도록 이벤트를 전달하게 됩니다.
전체적인 동작 화면을 첨부해 드리니 참고하시면 되겠습니다
![video]({{ site.baseurl }}/assets/video/EventBusExample_video.mp4)


소스코드는 제 GitHub 저장소에 업로드 되어 있으니 참고하시면 되겠습니다.[https://github.com/HyeokJaeChoi/EventBusExample](https://github.com/HyeokJaeChoi/EventBusExample)