---
layout: single
title: cocursor React 모듈 개발
categories: cocursor
excerpt: cocursor client 리액트에서 구현
---

## 1. 목표

| 협업 기능이 필요한 프로젝트, 사용자의 관심 지점을 추적하는 기능 등 다양한 창의적 활동에 cocursor 라이브러리가 활용될 수 있도록 한다.

구현하려는 서비스가 사용자에게 설득력있게 다가가려면,  
제공하는 기능이 유용해야 하고, 안정성이 보장되어 신뢰할 수 있어야 한다고 생각한다.  
또한 접근하는데 어려움이 없이 쉽게 사용할 수 있어야 한다.

그에 따라 목표를 다음과 같이 설정하였다.

- 개발자가 쉽게 사용할 수 있어야 함 → apiKey만 넘겨주면 동작하도록 설계
- Zero dependency → React 환경에서 추가 의존성을 요구하지 않도록 구현
- 유연한 커스터마이징 지원
- 사용하는 프로젝트의 성능에 영향을 주지 않도록 최적화

## 2. 기본 구현

### 프로젝트 구성

📦components
┣ 📜CoCursorProvider.tsx
┗ 📜Cursor.tsx

라이브러리는 `CoCursorProvider` 하나만 내보낸다.
`CoCursorProvider`에서 모든 커서를 관리한다.

`Cursor`에서는 x, y 좌표를 받아 top, left 값을 조정하여 커서를 이동시킨다.
또한, id를 기반으로 랜덤한 색상을 생성해 커서의 배경색으로 사용한다.

### 비즈니스 로직

CoCursorProvider는 마운트 시 웹소켓 연결을 하고, 메시지 수신을 설정한다.

```tsx
const [cursors, setCursors] = useState<Record<string, CursorData>>({});

useEffect(() => {
  ws.current = new WebSocket(WS_URL);

  ws.current.onmessage = (event) => {
    const cursorData: CursorData = JSON.parse(event.data);
    setCursors((prev) => ({ ...prev, [cursorData.id]: cursorData }));
  };

  return () => {
    ws.current?.close();
  };
}, []);
```

메시지 송신은 다음과 같이 addEventListener를 통해 관리한다.

```tsx
const sendCursorPosition = (e: MouseEvent) => {
  if (!ws.current || ws.current.readyState !== WebSocket.OPEN) return;

  const cursorData: CursorData = {
    id: userId.current,
    x: e.clientX,
    y: e.clientY,
    visible: true,
    name: userName.current,
  };
  ws.current?.send(JSON.stringify(cursorData));
};

const handleMouseLeave = () => {
  if (!ws.current || ws.current.readyState !== WebSocket.OPEN) return;

  const cursorData = {
    id: userId.current,
    visible: false,
  };
  ws.current?.send(JSON.stringify(cursorData));
};

useEffect(() => {
  window.addEventListener("mousemove", sendCursorPosition);
  document.addEventListener("mouseleave", handleMouseLeave);

  return () => {
    window.removeEventListener("mousemove", sendCursorPosition);
    document.removeEventListener("mouseleave", handleMouseLeave);
  };
}, []);
```

**🤔 왜 mousemove 이벤트는 window, mouseleave 이벤트는 document에 설정했나요?**

1. mousemove를 window에 등록한 이유

- document에 mousemove를 붙이면 문서 내부에서만 이벤트가 발생 => iframe 같은 외부 요소 위에서는 감지되지 않을 수 있다.
- document에 이벤트를 등록하면, 브라우저는 DOM 트리를 따라 이벤트를 전파하므로 불필요한 비용이 발생한다. => window에 직접 등록 시 DOM을 거치지 않는다.

2. mouseleave는 document에 등록한 이유

- mouseleave는 부모 요소에서 자식 요소로 이동할 때도 발생하는 이벤트다.  
  따라서 의도치 않을 때 이벤트가 발생할 수 있다.
- 마우스가 실제 문서 영역을 벗어날 때에 커서를 감추는 것이 기능적으로 맞기에 document에 등록하였다.

## 3. API KEY & 채널 관리

예상 User Flow는 다음과 같다.
CoCursor를 사용하고자 하는 사람은 서비스 페이지에서 API Key를 발급받고, 프로젝트에 적용한다.
그리고 `CoCursorProvider`에 prop으로 API KEY를 넘겨준다.

### API KEY

서버에서는 API KEY를 통해 서비스를 사용하는 프로젝트를 구분한다.  
따라서 서버에게 API KEY를 전달해야 한다.

어떻게 전달할 수 있을까?  
URL을 통한 전달은 API KEY가 너무 공개적으로 노출이 되는 것 같아 부담스러웠다.

웹소켓 객체 생성 시 다음과 같이 정보를 넣어줄 수 있는 배열이 있다.

```js
new WebSocket(url, protocols);
```

본래의 용도는 핸드셰이크 과정에서 클라이언트와 서버 간 특정 프로토콜을 협상하기 위해 사용된다.  
클라이언트에서 어떤 프로토콜을 사용할지 지정하고, 서버는 이에 따라 응답한다.

현재 프로토콜을 따로 분리하고 있지는 않기 때문에 `sec-websocket-protocol`를 API KEY 전달용으로 사용하였다.

이 `sec-websocket-protocol`를 사용하여 API KEY를 전달하면,  
서버에서는 다음과 같이 API KEY를 받을 수 있다.

```js
const apiKey = req.headers["sec-websocket-protocol"];
```

### 채널 관리

같은 프로젝트에서도 페이지마다 커서 정보를 공유할 범위를 다르게 설정할 수 있어야 한다.  
예를 들어, 하나의 프로젝트 내에서도 서로 다른 페이지에서 다른 채널을 사용하고 싶을 수 있다.

채널은 공개되어도 문제 없는 정보라고 판단되어 URL 쿼리 스트링을 사용했다.

```js
wsURL = `${WS_URL}?channel=${channel}`;
```

## 3. 훅 추가

### 추가 기능

현재, cocursor 대한 모든 설정은 최상위 컴포넌트인 `CoCursorProvider`에서 관리하고 있다.

```tsx
<CoCursorProvider
  apiKey={process.env.REACT_APP_COCURSOR_API_KEY as string}
  myName="alexgoni" // 내 커서 라벨 텍스트
  allowInfoSend={true} // 정보 송신 여부
  disabled={false} // cocursor 기능 허용 여부
>
  {/* // ... */}
</CoCursorProvider>
```

하지만 사용 중 특정 버튼을 눌러서 cocursor 기능을 끈다거나, 사용자 이름이 업데이트 되어서 새로운 이름을 부여해야 하는 상황이 생길 수 있다.  
이러한 상황은 대부분 CoCursorProvider 하위 컴포넌트에서 일어날 것이다.

그렇다면 사용자는 어떻게 설정해야할까?
사용자가 직접 Context를 생성하고, 다시 props를 넘겨줘야 할까?

이는 너무 불친절하다.
대부분의 라이브러리처럼 별도의 훅을 제공하여 설정을 쉽게 변경할 수 있도록 해보자.

1. Context 생성

상태를 변경할 수 있는 Context를 생성하자.

```tsx
import { createContext, useContext } from "react";

interface CoCursorContext {
  channel?: string;
  myName?: string;
  allowInfoSend: boolean;
  quality: "high" | "middle" | "low";
  disabled: boolean;
  setChannel: (channel?: string) => void;
  setMyName: (name?: string) => void;
  setAllowInfoSend: (block: boolean) => void;
  setQuality: (quality: "high" | "middle" | "low") => void;
  setDisabled: (disable: boolean) => void;
}

export const CoCursorContext = createContext<CoCursorContext | null>(null);

export function useCoCursor() {
  const context = useContext(CoCursorContext);

  if (!context) {
    throw new Error("CoCursor 컨텍스트를 호출할 수 없는 범위입니다.");
  }

  return context;
}
```

`useCoCursor` 훅을 사용하면 상태를 쉽게 가져오고, 변경할 수 있다.

2. `CoCursorProvider`에서 Context Provider 적용

props으로 넘겨준 값은 초기값으로 저장하고, 이후의 상태는 Context를 통해 관리하며, 바로 업데이트될 수 있도록 하였다.

```tsx
export default function CoCursorProvider({
  children,
  apiKey,
  channel: initialChannel,
  myName: initialMyName,
  allowInfoSend: initialAllowInfoSend = true,
  quality: initialQuality = "high",
  disabled: initialDisabled = false,
}: Props) {
  const [channel, setChannel] = useState(initialChannel);
  const [myName, setMyName] = useState(initialMyName);
  const [allowInfoSend, setAllowInfoSend] = useState(initialAllowInfoSend);
  const [quality, setQuality] = useState<"high" | "middle" | "low">(
    initialQuality
  );
  const [cursors, setCursors] = useState<Record<string, CursorMessage>>({});
  const [disabled, setDisabled] = useState(initialDisabled);

  // ...

  return (
    <CoCursorContext.Provider
      value={{
        channel,
        setChannel,
        myName,
        setMyName,
        allowInfoSend,
        setAllowInfoSend,
        quality,
        setQuality,
        disabled,
        setDisabled,
      }}
    >
      {/* ... */}
    </CoCursorContext.Provider>
  );
}
```

<br />

이제 `useCoCursor` 훅을 활용하면, 하위 컴포넌트에서도 간편하게 설정을 변경할 수 있다.

```tsx
export function ChildComponent() {
  const { setAllowInfoSend } = useCoCursor();

  const handleClick = () => {
    setAllowInfoSend(false);
  };

  return <button onClick={handleClick}>커서 정보 송신 중지</button>;
}
```

## 4. 커서 위치 계산 로직 변경

기존에는 clientX, clientY 값을 사용하여 커서 위치를 설정하였다.

하지만 화면 비율이 다를 경우, 다른 사용자에게 표시되는 내 커서의 위치가 실제와 다르게 보이는 경우가 있었다.  
이를 해결하기 위해 전체 페이지 기준 좌표(pageX, pageY)를 사용하도록 변경하였다.

<br />

그러나 단순히 pageX, pageY를 사용하면, 스크롤 위치에 따라 커서가 올바르게 표시되지 않는 문제가 발생한다.  
이를 해결하기 위해 스크롤 보정 값을 함께 고려해야 한다.

<br />

**해결 방법: 비율 및 스크롤 보정 적용**

1. 커서 위치 전송

기존에는 특정 좌표값을 전송하였다면, 이제는 비율값을 전송한다.

```tsx
const sendCursorPosition = (e: MouseEvent) => {
  const width = document.documentElement.scrollWidth;
  const height = document.documentElement.scrollHeight;

  const cursorData: CursorMessage = {
    type: "cursor",
    id: userId.current,
    x: e.pageX / width, // 전체 페이지 크기 대비 비율
    y: e.pageY / height, // 전체 페이지 크기 대비 비율
    visible: true,
    name: myName || "anonymous",
  };
};
```

2. 커서 렌더링

```tsx
function Cursor({ data }: { data: CursorMessage }) {
  const { x, y } = data;

  const width = document.documentElement.scrollWidth;
  const height = document.documentElement.scrollHeight;

  return (
    <div
      style={{
        top: y * height - window.scrollY,
        left: x * width - window.scrollX,
      }}
      className="cocursor__cursor-wrapper"
    ></div>
  );
}
```

- `x * width`, `y * height`로 화면 크기에 맞게 좌표 변환
- window.scrollX, window.scrollY 값을 빼서 스크롤 보정 적용

<br />

물론 이 방식으로도 화면 비율이 다르다면 완벽하게 정확한 위치를 특정할 수 없다.

현재 cocursor는 전체 페이지를 대상으로 기능을 제공하지만,  
크기가 고정된 부모 영역을 대상으로 하여 커서 기능을 제공하는 경우에는 정확한 위치를 제공할 수 있을 것이다.  
이때는 clientX, clientY를 통해서 부모 요소 기준 좌표를 공유하는 방식이 적절해 보인다.

## 5. 최적화

현재 커서 전송 및 수신은 마우스가 움직일 때마다 연속적으로 발생한다.
이는 서버 비용 증가와 렌더링 비용 증가로 이어질 수 있다.

이를 방지하기 위해, 쓰로틀을 적용하여 이벤트 발생 간격을 조절하는 최적화를 진행하였다.

1. 쓰로틀 구현

```ts
// throttle.ts

const throttle = (fn: Function, delay: number) => {
  let lastCall = 0;
  return (...args: any[]) => {
    const now = Date.now();
    if (now - lastCall >= delay) {
      lastCall = now;
      fn(...args);
    }
  };
};

export default throttle;
```

2. quality 옵션 적용

사용자는 `quality` 옵션을 통해 쓰로틀 간격을 조정할 수 있다.
전송하는 함수에 쓰로틀을 적용하고, quality에 따라 간격을 다르게 설정하였다.

```tsx
const throttleMS = (() => {
  if (quality === "high") return 0;
  if (quality === "middle") return 10;
  if (quality === "low") return 30;
  return 0;
})();
const throttledSendCursorPosition = throttle(sendCursorPosition, throttleMS);
```

3. 최적 쓰로틀 간격 설정

쓰로틀 간격이 40ms 이상일 경우, 커서 움직임이 너무 끊기는 느낌이 들어
최대 값을 30ms로 제한하여 자연스러운 움직임을 유지하도록 설정하였다.
