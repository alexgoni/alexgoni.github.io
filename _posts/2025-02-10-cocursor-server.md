---
layout: single
title: cocursor 서버
categories: cocursor
excerpt: API Key 인증 로직 & 채널 구분
---

## 1. API Key 인증 로직

cocursor는 API Key를 사용한 인증 방식을 적용하여 웹소켓 통신을 보호한다.

**인증 흐름**

1. 사용자는 CoCursor 서비스에서 API Key를 발급받는다.
2. 클라이언트에서 웹소켓 서버와 연결할 때, `sec-websocket-protocol` 헤더에 API Key를 포함하여 요청을 보낸다.
3. 서버는 받은 API Key를 Firestore에서 조회하여 유효성을 검증한다.
4. API Key가 없거나, 유효하지 않다면 연결을 즉시 종료한다.

```js
wss.on("connection", async (ws, req) => {
  const apiKey = req.headers["sec-websocket-protocol"];

  // API Key가 없는 경우
  if (!apiKey) {
    ws.send(JSON.stringify({ type: "error", message: "API Key is required" }));
    ws.close();
    return;
  }

  try {
    // Firestore에서 API Key 검증
    const apiKeyDoc = await db.collection("apiKeys").doc(apiKey).get();
    if (!apiKeyDoc.exists || !apiKeyDoc.data().active) {
      console.log(`[거부됨] 잘못된 API Key: ${apiKey}`);
      ws.send(JSON.stringify({ type: "error", message: "Invalid API Key" }));
      ws.close();
      return;
    }
  } catch (error) {
    console.error("API Key 검증 중 오류 발생:", error);
    ws.send(
      JSON.stringify({
        type: "error",
        message: "Server error during API Key validation",
      })
    );
    ws.close();
    return;
  }
});
```

글을 쓰면서 다시 보니 `sec-websocket-protocol`도 클라이언트에서 노출된다.
이렇게 되면 다른 사용자가 해당 API Key를 가져다가 다른 프로젝트에서 사용할 수도 있고, 그럴 경우 원치 않는 커서 정보가 공유될 위험이 있다.
보안 측면에서 더 안전한 방법을 사용할 필요가 있다.

## 2. 채널 구분

프로젝트에서 cocursor를 사용할 때, 사용자는 기능이 적용되는 페이지를 구분할 필요가 있을 수 있다.
(ex. A팀 대시보드 페이지의 커서 정보가 B팀 대시보드 페이지와 공유되서는 안됨)

보통 웹소켓에서는 채널 개념을 사용해 이를 구분하므로, 나도 채널이라는 용어를 썼다.

**구조**

- 전체 프로젝트를 관리하는 projectRooms 객체
  - 각 API Key가 하나의 프로젝트
  - 각 프로젝트 안에 여러 채널이 존재 (URL에서 channel 값 기준)
  - 각 채널 안에 연결된 ws 객체를 Set으로 관리 (추가/삭제 용이)

```js
wss.on("connection", async (ws, req) => {
  // ...
  const urlParams = new URLSearchParams(req.url.split("?")[1]);
  const channelName = urlParams.get("channel") || "default";

  // 인증 로직 이후

  // 프로젝트 방이 없으면 생성
  if (!projectRooms[apiKey]) {
    projectRooms[apiKey] = {};
  }

  // 채널이 없으면 생성
  if (!projectRooms[apiKey][channelName]) {
    projectRooms[apiKey][channelName] = new Set();
  }

  // 현재 유저를 해당 프로젝트 & 채널에 추가
  projectRooms[apiKey][channelName].add(ws);

  ws.on("message", (data) => {
    try {
      const cursorData = JSON.parse(data);
      const message = { type: "cursor", ...cursorData };
      projectRooms[apiKey][channelName].forEach((client) => {
        if (client !== ws && client.readyState === WebSocket.OPEN) {
          client.send(JSON.stringify(message));
        }
      });
    } catch (error) {
      console.error(`메시지 처리 중 에러 발생: ${error.message}`);
      ws.send(
        JSON.stringify({ type: "error", message: "Invalid message format" })
      );
    }
  });
});
```

웹소켓 연결이 끊어질 때는 비어있는지 확인하여 메모리 누수를 최소화하였다.

```js
ws.on("close", () => {
  projectRooms[apiKey][channelName].delete(ws);
  console.log(`[${apiKey} - ${channelName}] 사용자가 연결 종료`);

  // 채널이 비었으면 채널 삭제
  if (projectRooms[apiKey][channelName].size === 0) {
    delete projectRooms[apiKey][channelName];
  }

  // 프로젝트 방이 비었으면 프로젝트 방 삭제
  if (Object.keys(projectRooms[apiKey]).length === 0) {
    delete projectRooms[apiKey];
  }
});
```
