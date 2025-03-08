---
layout: single
title: cocursor 기획
categories: cocursor
---

코드잇 스프린트에서 팀의 일정 관리를 도와주는 [Planit](https://planit-xi.vercel.app/)을 개발했었다.
해당 프로젝트는 다수의 인원이 하나의 페이지에서 수정할 수 있는 서비스였다.

개발 전에 "만약 같은 일정을 동시에 수정하면 어떻게 되지?"" 와 같은 의문이 들었고,
피그마, 노션과 같은 실시간 업데이트 웹 구현을 통해 이를 해결하고 싶었다.

노션의 경우는 글자를 입력할 때마다 `syncRecordValuesSpace`라는 api 명으로 매번 fetch를 하고 있었고,
![alt text](/images/2025-02-01-cocursor-plan/image.png)
피그마의 경우는 웹소켓 통신을 통해 Binary Message를 주고 받아 커서의 위치를 실시간으로 업데이트 하고 있었다.
![alt text](/images/2025-02-01-cocursor-plan/image2.png)

웹소켓 통신 방식을 사용해 Planit에서는 [Pusher](https://pusher.com/)라는 써드파티 라이브러리를 통해 이벤트에 따른 데이터를 주고 받으며 새로고침하지 않아도 데이터가 바로바로 업데이트 되도록 구현했었다.

<video controls style="width: 100%; max-width: 800px;">
  <source src="/images/2025-02-01-cocursor-plan/video.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

하지만 돌아보면, 단순히 알림으로만 표시하는 것은 다소 부족하다는 느낌이 들었다.
피그마에서는 커서 움직임을 실시간으로 보여줌으로써, 사용자가 무엇을 확인하고 있으며, 어떤 행동을 할 예정인지 다른 사용자들이 예측할 수 있도록 돕는다.

**이처럼 웹 페이지에서도 동일한 경험을 제공할 수는 없을까?** 라는 생각이 들었다.
또한, Pusher처럼 npm 모듈로 제공하여, 간편하게 이 기능을 구현할 수 있도록 만들고 싶다는 아이디어가 떠올랐고, 이를 계기로 이 프로젝트를 시작하게 되었다.
