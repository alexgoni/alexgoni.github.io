---
layout: single
title: 스프린트 기초 프로젝트 회고
excerpt: Fandom K 기초 프로젝트 회고. 배운 점, 아쉬운 점, 그리고 다음 프로젝트에 적용해 볼 보완할 점
---

## 배운 점

### Skeleton 사용

이전에는 로딩 상태 표시를 loading spinner를 사용하거나 따로 처리하지 않았다.
Skeleton을 사용하는 것이 layout shifting을 최소화하고, UX 입장에서 더 좋은 선택임을 배웠다.

### Data Fetching Hook

프로젝트에서 data fetch 커스텀 hook을 만들어 사용하였다.

```js
export default function useRequest({ options, skip = false, deps = [] }) {
  const [data, setData] = useState(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);

  const requestFunc = async (...args) => {
    setIsLoading(true);
    setError(null);

    try {
      const response = await dispatcher({ ...options, ...args });

      setData(() => response);
      return response;
    } catch (err) {
      setError(() => err);
      return err;
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => {
    if (skip) return;
    requestFunc();
  }, deps);

  return { data, isLoading, error, requestFunc };
}
```

data fetch를 할 때 try...catch를 hook 안에서 처리하고 state만 반환하는 장점이 있다.
다만 사용하면서 불편했던 점은 반환된 requestFunc의 options에 다른 값을 넣어 다시 data fetch하고 싶은데 이 부분이 어려웠다.

options을 자유롭게 사용하고, isLoading과 error만 관리하는 커스텀 훅으로 만든다면 이렇게 보완할 수 있을 것 같다.

```js
export default function useAxiosFetch() {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);

  const axiosFetch = async (options) => {
    setIsLoading(true);
    setError(null);

    try {
      const response = await dispatcher({ ...options });

      return response;
    } catch (err) {
      setError(err);
      return err;
    } finally {
      setIsLoading(false);
    }
  };

  return { isLoading, error, axiosFetch };
```

### Style 코드

팀원분의 scss 코드를 보면서 유용한 전략을 많이 배울 수 있었다.

```css
// 말줄임 mixin
// 사용 : @include collpase(line수);
@mixin collapse($line) {
  display: -webkit-box;
  overflow: hidden;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: $line;
}
```

```css
font-size: 62.5%;
// font-size rem으로 통일 : root 10px
// 12px == 1.2rem / 15px == 1.5rem
```

### Persistence

데이터를 생성한 프로그램이 종료되어도 사라지지 않는 데이터의 특성.
이번 프로젝트에서는 localStorage를 통해 데이터를 계속해서 유지하였다.

jotai에서 localStorage를 관리하는 방법을 배웠다.
https://jotai.org/docs/guides/persistence

### 이미지 최적화

페이지에 처음 들어갈 때 필요한 리소스를 모두 다운로드 받는 것이 아닌 이미지가 뷰포트에 진입할 때까지 로딩을 지연시키는 방식을 배웠다.

## 아쉬운 점

- 이슈 트래커 활용 부족, 일정 관리 부족
- 해야할 일 날마다 데일리 스크럼에서 구두로 할당
- UI 개발과 기능 구현에서 맡은 부분이 달라짐
- Test 코드, 성능 최적화 도입하지 못합
- 우리 팀만의 추가 기능 도입하지 못함
- 코드 리뷰 마지막에는 제대로 하지 못함

## 보완할 점, Try

- Jira 이슈 트래킹 도입
- sprint, timeblock 설정
- KPT 회고 매주마다 진행
- 데일리 스크럼에서 현재 작업의 예상 소요 시간 공유
- 기술 세미나 / 팀 위키
- page, section 단위로 담당자 지정
- 공통 부분(코드가 프로젝트 전체적으로 영향을 끼치는 부분)과 개인 부분 나누기
- 머지 스레드 도입
  공통 부분과 같이 머지가 급한 PR은 `@스프린터` 사용과 함께 간단하게 이유 알리기
- 공통 부분은 JSDoc 사용하여 모든 팀원이 사용할 수 있도록 표기
- eslint import 정렬 규칙 도입
- squash & merge 도입
- animation 라이브러리 도입
