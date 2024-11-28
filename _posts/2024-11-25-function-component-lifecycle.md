리액트 라이프사이클에 대해 예제 코드와 함께 이해해보자.

---

### 0. 클래스형 컴포넌트

컴포넌트가 생겨나고, 변화하고, 없어지는 일련의 프로세스를 클래스형 컴포넌트로 먼저 이해해보자.

![](/images/2024-11-25-function-component-lifecycle/image1.png)

컴포넌트가 처음 생성될 시 constructor 생성자에 의해 만들어진다.
후에 클래스형 컴포넌트에 필수가 되는 render 메서드를 통해 웹페이지를 paint한다.

이 때 개발자가 DOM 요소를 조작하거나 데이터를 변화시키기 위해 **특정 코드를 특정 시점에** 실행시키고자 하는 경우가 있다.

컴포넌트의 생명 주기를 세 부분으로 나누어 해당 메서드에 특정 코드가 실행되게 할 수 있다.

**componentDidMount / componentDidUpdate / componentWillUnmount**

---

### 1. 함수형 컴포넌트

함수형 컴포넌트에서 코드 실행 순서는 다음과 같다.
![](/images/2024-11-25-function-component-lifecycle/image2.png)

[ state ] -> [ return ] -> [ useEffect ]

state가 선언되고 해당 state를 바탕으로 return 문 안에 있는 jsx 코드가 실행된다.
웹페이지에 그려진 이후에 useEffect가 실행된다.

useEffect는 클래스형 컴포넌트의 위 세가지 메서드 역할을 종합한다.
기본적으로 클래스가 mount될 때 한 번 실행되고, 의존성 배열에 넣은 값이 변화될 때마다 실행되어 componentDidUpdate의 역할을 하게 된다.
컴포넌트가 마운트 해제 될 때는 클린업 함수를 return 하여 메모리 누수를 방지할 수 있다.

---

### 2. 함수형 컴포넌트 코드 실행 로직

1. 함수는 state 값이 변경되면 리렌더링된다.
2. useEffect는 의존성 배열의 값이 변할 시 특정 코드를 실행한다.

위 두가지 조건에 유의해 아래 코드를 해석해보자.

```js
export default function App() {
  debugger;
  const [wedding, setWedding] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(false);

  useEffect(() => {
    setLoading(true);

    fetch("http://localhost:8888/wedding")
      .then((response) => {
        if (!response.ok) throw new Error("청첩장 정보를 불러오지 못했습니다.");
        return response.json();
      })
      .then((data) => {
        setWedding(data);
      })
      .catch((e) => {
        console.log(e);
        setError(true);
      })
      .finally(() => {
        setLoading(false);
      });
  }, []);

  if (loading) return <FullScreenMessage type="loading" />;
  if (error) return <FullScreenMessage type="error" />;

  return (
    <>
      <div className={cx("container")}>
        {wedding ? JSON.stringify(wedding) : <div>hi</div>}
      </div>
    </>
  );
}
```

위 코드는 debugger에 의해 3번 걸리게 된다. (React.strictMode 제거 시)
3번 리렌더링을 각 과정과 함께 살펴보자.

**1번 째**

1. state
   state들이 초기화 된다.
2. render
   return 문 안에 의해 웹페이지가 paint된다.
   현재 wedding 값이 null이기 때문에 "hi"가 보여지게 된다.
3. useEffect
   setLoading이 실행된다.
   fetch가 실행된다. 비동기로 실행되기 때문에 아래 then 함수가 바로 실행되지는 않고 useEffect 실행이 마무리 된다.
   _Loading state가 변했기 때문에 함수가 리렌더링된다._

**2번 째**

1. state
   loading 값이 true가 된다.
   setState로 state가 즉각적으로 변하지 않을 때가 많다. 이는 코드가 다 실행되고 나서 다시 state 선언 부분으로 왔을 때 해당 state가 업데이트되기 때문이다.
2. render
   `if (loading) return <FullScreenMessage type="loading" />`
   이 부분에서 return 되어 화면에 loading FullScreenMessage가 보여지게 된다.
3. fetch
   이전에 비동기로 실행되었던 useEffect 안의 fetch에서 then 함수가 실행된다.(클로저)
   이후 setWedding이 실행된다.
   _wedding state가 변했기 때문에 함수가 리렌더링된다._

**3번 째**

1. state
   wedding 값에 데이터가 들어온다.
2. render
   변화된 state를 기반으로 return 문이 재실행되면서 웹페이지를 다시 그리게 된다.
   Loading 컴포넌트 대신 데이터가 그려지게 된다.

---

리액트에서 함수형 컴포넌트의 라이프사이클 동안 코드 실행이 어떻게 진행되는지 알아보았다.
debugger를 통해 하나씩 진행시키면서 state가 어떻게 변경되고 왜 변경된 state는 즉각적으로 반영이 안되는지,
useEffect와 return 문 중 어떤 부분이 코드 실행이 먼저 되는지, 비동기 함수가 있을 때는 어떻게 동작하는지에 대해 알아볼 수 있었다.
