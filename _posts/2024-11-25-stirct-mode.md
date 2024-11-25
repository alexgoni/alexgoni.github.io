---
layout: single
title: "리액트 Strict Mode"
---

React 개발하면서 골치 아프게 했던 StrictMode.
이 StrictMode에 대한 설명이 자세히 없어서 [공식문서](https://react.dev/reference/react/StrictMode#enabling-strict-mode-for-entire-app)와 함께 공부해보았다.

---

### Usage

```js
const root = createRoot(document.getElementById("root"));
root.render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

index.js에서 최상위 컴포넌트인 App을 StrictMode로 감싸준다.

StrictMode는 하위의 모든 컴포넌트를 검사하기 때문에 특정 컴포넌트에서만 생략할 수 없다.
특정 부분만 StrictMode로 검사는 다음과 같이 할 수 있다.

```js
function App() {
  return (
    <>
      <Header />
      <StrictMode>
        <main>
          <Sidebar />
          <Content />
        </main>
      </StrictMode>
      <Footer />
    </>
  );
}
```

StrictMode는 쉽게 설명하면 코드를 두 번 실행하여 잠재적인 버그나 메모리 누수를 개발자에게 알려주는 역할을 한다.
두 가지 예시를 통해 자세히 알아보자.

---

### Fixing bugs found by double rendering in development

**StrictMode 첫 번째 역할 : 비순수 함수 catch**

**리액트 규칙**
**모든 함수형 컴포넌트는 순수 함수로 작성되어야 한다.**

같은 입력을(props, state, context) 넣으면 같은 JSX를 return해야 한다.

비순수 함수 예시

```js
let guest = 0;

function Cup() {
  guest = guest + 1;
  return <h2>Tea cup for guest #{guest}</h2>;
}

export default function TeaSet() {
  return (
    <>
      <Cup />
      <Cup />
      <Cup />
    </>
  );
}
// Cup 컴포넌트가 호출될 때마다 다른 jsx가 리턴된다.
```

만약 함수가 순수하다면 두 번 렌더링 되는 것은 같은 결과를 출력할 것이다. 그러나 순수하지 않다면(입력값을 변경하는 경우) 다른 결과가 리턴된다.

이를 미리 발견하기 위해 StrictMode가 사용된다.
다음 예시를 살펴보자.

```js
export default function StoryTray({ stories }) {
  const [isHover, setIsHover] = useState(false);
  const items = stories;
  items.push({ id: "create", label: "Create Story" });

  return (
    <ul
      onPointerEnter={() => setIsHover(true)}
      onPointerLeave={() => setIsHover(false)}
      style={{
        backgroundColor: isHover ? "#ddd" : "#fff",
      }}
    >
      {items.map((story) => (
        <li key={story.id}>{story.label}</li>
      ))}
    </ul>
  );
}
```

**StrictMode 해제 시**
위 코드는 li 태그에 마우스를 hover할 경우 state가 변경되면서 리렌더링된다.
`const items = stories;` 리렌더링 될 시 items 상수는 사라지더라고 재선언되더라도 같은 주소값을 참조하고 있는 stories props에는 값이 추가된 상태이다.
이 때문에 마우스를 hover 했을 때 `Encountered two children with the same key` 에러가 발생한다.

**StrictMode 적용 시**
이전에는 마우스를 호버해야 에러를 확인할 수 있었다.
하지만 StrictMode는 항상 함수를 두 번 렌더하기 때문에 마우스를 호버하지 않고도 에러를 바로 확인할 수 있다.
이는 개발자가 놓칠 수 있는 에러 부분을 명확하게 해준다.

---

### Fixing bugs found by re-running Effects in development

**StrictMode 첫 번째 역할 : 클린업 함수 필요성 제기**

useEffect를 사용하면서 클린업 함수를 리턴해주면 컴포넌트가 언마운트될 때 해당 함수가 실행된다.
StrictMode를 사용함으로써 리액트는 추가된 setup+cleanup 사이클을 돌리고 이를 통해 메모리 누수나 버그를 개발자에게 알리고자 한다.

```js
// chat.js
let connections = 0;

export function createConnection(serverUrl, roomId) {
  return {
    connect() {
      console.log(
        '✅ Connecting to "' + roomId + '" room at ' + serverUrl + "..."
      );
      connections++;
      console.log("Active connections: " + connections);
    },
    disconnect() {
      console.log('❌ Disconnected from "' + roomId + '" room at ' + serverUrl);
      connections--;
      console.log("Active connections: " + connections);
    },
  };
}

//App.js
export default function ChatRoom() {
  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
  }, []);
  return <h1>Welcome to the {roomId} room!</h1>;
}
```

**StrictMode 해제 시**

```
✅ Connecting to "general" room at https://localhost:1234
Active connections: 1...
```

콘솔에 다음과 같이 출력되며 아무 이상이 없어 보인다.
하지만 컴포넌트가 언마운트되고 다시 마운트될 때 다음과 같이 출력된다.

```
✅ Connecting to "general" room at https://localhost:1234...
Active connections: 1
✅ Connecting to "general" room at https://localhost:1234...
Active connections: 2
```

connections 값이 추가된 것을 보아 연결이 끊어지지 않고 새로운 연결은 만든 것을 알 수 있다.
이는 메모리 누수와 잠재적인 버그 요소이다.

**StrictMode 적용 시**
처음 마운트 될 때부터 두 번 렌더되기 때문에 콘솔에 바로 다음과 같이 출력된다.

```
✅ Connecting to "general" room at https://localhost:1234...
Active connections: 1
✅ Connecting to "general" room at https://localhost:1234...
Active connections: 2
```

이를 통해 개발자는 클린업 함수가 필요하다는 것을 알 수 있고 다음과 같이 코드를 교정할 수 있다.

```js
useEffect(() => {
  const connection = createConnection(serverUrl, roomId);
  connection.connect();
  return () => connection.disconnect();
}, []);
```

StrictMode가 없을 때는 Effect가 클린업 함수가 필요하다는 것을 놓칠 수 있다. 이를 명확하게 알려주는 역할을 한다.

---

StrictMode에 대해서 잘 알지 못했을 때는 왜 콘솔에 두 번 출력되고 의도한데로 출력되지 않는지에 대해 스트레스만 쌓였다.
이 후에 StrictMode의 존재를 알았을 때는 해당 상황 때마다 모드를 해제하고 개발하였다.

공부를 하고 난 뒤에는 StrictMode가 어떤 역할을 하고 왜 필요한지에 대해 알았고 처음 의도한대로 동작하지 않을 때에도 StrictMode가 어떻게 개입하고 있는지에 대해 설명할 수 있게 되었다.
리액트에서 모드를 적용하고 개발하는 것을 권고하고, React DevTools 사용 시에는 두 번째 렌더의 console.log()는 흐리게 보여지게 해준다고 한다.

> Note
> If you have React DevTools installed, any console.log calls during the second render call will appear slightly dimmed. React DevTools also offers a setting (off by default) to suppress them completely.

해당 동작을 이해하고 있고 도구도 존재하기 때문에 앞으로는 StrictMode를 이용해 명확하지 않은 버그도 잡을 수 있도록 해야겠다.
