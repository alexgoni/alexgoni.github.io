---
layout: single
title: cocursor 초기 모델
categories: cocursor
excerpt: Vanila JS & 로컬 서버로 구현하기
---

## Server

먼저 Node.js를 사용해 간단한 웹소켓 서버를 개발했다.  
웹소켓 통신을 위해 `ws` 라이브러리를 사용했으며,  
특정 클라이언트로부터 메시지가 오면, 이를 연결된 모든 클라이언트에게 전달하는 구조로 설계했다.  
단, 메시지를 보낸 클라이언트에게는 다시 전달되지 않도록 처리했다.

```js
const WebSocket = require("ws");

const wss = new WebSocket.Server({ port: 8080 });

wss.on("connection", (ws) => {
  console.log("새로운 사용자가 연결됨");

  // 수신
  ws.on("message", (data) => {
    const cursorData = JSON.parse(data);

    wss.clients.forEach((client) => {
      // 메시지를 보낸 주체에게는 다시 전달 X
      if (client !== ws && client.readyState === WebSocket.OPEN) {
        // 송신
        client.send(JSON.stringify(cursorData));
      }
    });
  });

  ws.on("close", () => {
    console.log("사용자가 연결을 종료함");
  });
});

console.log("웹소켓 서버가 8080번 포트에서 실행 중...");
```

## Client

만들려고 하는 npm 모듈은 리액트를 대상으로 제공할 예정이지만, 우선 가능성을 확인하기 위해 Vanilla JS로 구현하였다.  
(후에 React용 npm 모듈이 개발 완료되면, 특정 프레임워크에 종속되지 않고 모든 환경에서 사용할 수 있도록 만들면 좋을 것 같다.)

### 1. 웹소켓 연결

브라우저에서는 Web API에서 제공하는 WebSocket 객체를 사용하여 웹소켓 서버와 연결하고, 연결을 관리할 수 있다.  
이 객체를 통해 데이터 송수신을 관리할 것이다.

```js
// Handshake
const ws = new WebSocket("ws://localhost:8080");
```

**주요 메서드 및 프로퍼티**

```js
// 1. 웹소켓 연결이 되었을 때
ws.onopen = () => {
  console.log("서버에 연결됨");
};

// 2. 웹소켓 연결이 끊어졌을 때
ws.onclose = () => {
  console.log("서버와 연결 끊김");
};

// 3. 웹소켓으로부터 메시지가 왔을 때
ws.onmessage = (event) => {
  // 메시지 처리 로직...
};

// 4. 웹소켓으로 메시지 보낼 때
ws.send(JSON.stringify(cursorData));

// 5. 웹소켓 연결을 끊을 때
ws.close();
```

### 2. 메시지 송수신

#### 메시지 타입 정의

서버는 받은 데이터를 클라이언트들에게 다시 전송하는 역할만 한다.  
따라서 데이터의 타입은 클라이언트에서 정의된다.  
클라이언트에서 전송하는 데이터는 다음과 같다:

```js
const cursorData = {
  id: userId,
  x: e.clientX,
  y: e.clientY,
  visible: true,
};
```

#### 송수신 메서드

클라이언트에서 마우스를 움직일 때마다 데이터를 보낸다.

```js
// 커서 위치 전송
const sendMyCursorData = (e) => {
  const cursorData = {
    id: userId,
    x: e.clientX,
    y: e.clientY,
    visible: true,
  };

  ws.send(JSON.stringify(cursorData));
};
```

만약 마우스가 화면 밖을 나가면 visible을 false로 설정한다.  
화면 밖으로 나갈 때마다 연결을 끊고 다시 연결하는 것보다, 연결을 유지한 채로 화면에서만 커서를 숨기는 것이 비용 측면에서 더 효율적이다.

```js
// 화면 밖으로 나가면 숨기기
const handleMouseLeave = () => {
  const cursorData = {
    id: userId,
    visible: false,
  };

  ws.send(JSON.stringify(cursorData));
};

document.addEventListener("mouseleave", handleMouseLeave);
```

웹소켓에서 메시지를 받았을 때, id가 없는 경우에는 새로운 element를 생성하고,  
id가 있으면 실시간으로 left와 top 위치를 업데이트하여 화면에 커서를 표시한다.

```js
// 수신 (다른 사용자 커서 업데이트)
ws.onmessage = async (event) => {
  const { id, x, y, visible } = JSON.parse(event.data);
  let cursor = document.getElementById(id);

  if (!cursor) {
    const color = stringToColor(id);
    await loadArrowSVG(id, color);
    cursor = document.getElementById(id);
  }

  if (!visible) {
    cursor.style.display = "none";
  } else {
    cursor.style.display = "block";
    cursor.style.left = `${x}px`;
    cursor.style.top = `${y}px`;
  }
};
```

이때 커서마다 특정 색상을 가져야 한다.  
클라이언트에서 랜덤으로 생성된 id 값을 기준으로 랜덤한 색상을 생성하는 유틸 함수를 만들었다.  
글자가 흰색이기 때문에 너무 밝은 색은 제외하도록 설정하였다.

```js
// 문자열을 색상으로 변환하는 함수
const stringToColor = (str) => {
  let hash = 0;

  for (let i = 0; i < str.length; i += 1) {
    hash = str.charCodeAt(i) + ((hash << 5) - hash);
  }

  let color = "#";
  const BRIGHTNESS_LIMIT = 180;

  for (let i = 0; i < 3; i += 1) {
    let value = (hash >> (i * 8)) & 0xff;
    value = Math.min(value, BRIGHTNESS_LIMIT);
    color += `00${value.toString(16)}`.slice(-2);
  }

  return color;
};
```

<br />

테스트해봤을 때 잘 되는 것을 확인했다. :)
