---
layout: single
title: Cookie 기반 JWT 인증에서 reissue 다루기
categories: Gradpath
excerpt: 401 응답에 대한 재요청 처리
---

프로젝트에서 인증은 비교적 단순한 규칙을 따르고 있었습니다.
로그인에 성공하면 서버는 `Access Token`과 `Refresh Token`을 Cookie에 담아 응답합니다.  
토큰 저장을 프론트엔드가 다루지 않으므로, 중요했던 지점은 “만료 이후를 어떻게 다룰 것인가”였습니다.

## 왜 reissue가 중요했는가

인증이 필요한 서비스에서는 결국 Access Token이 만료됩니다.  
문제는 이 만료가 사용자의 행동과 딱 맞물려 일어나지 않는다는 점입니다.

사용자는 페이지를 이동하다가, 어떤 목록을 조회하다가, 저장 버튼을 누르다가, 혹은 여러 요청이 동시에 나가는 순간에 토큰 만료를 만나게 됩니다.  
이때 프론트엔드가 적절하게 대응하지 못하면 다음과 같은 일이 쉽게 벌어집니다.

- 같은 시점에 여러 요청이 한꺼번에 `401`을 받는다.
- 각 요청이 제각각 재발급을 시도한다.
- 재발급이 끝나기 전에 일부 요청은 실패하고, 일부는 성공한다.
- 사용자는 갑자기 로그인 페이지로 튕기거나, 저장 버튼을 다시 눌러야 한다.

결국 `reissue`는 단순히 “401이 오면 한 번 더 요청한다” 수준으로 다루기 어려웠습니다.  
특히 여러 요청이 동시에 움직이는 화면에서는 더더욱 그랬습니다.

## 이 프로젝트에서 잡은 전제

이번 프로젝트에서는 인증 정보를 프론트엔드 상태에 직접 들고 가지 않았습니다.  
서버가 토큰을 Cookie에 담아 내려주고, 프론트엔드는 `credentials: 'include'` 기반으로 요청을 보내는 구조였습니다.

```ts
client.setConfig({
  baseUrl,
  credentials: 'include',
});
```

이 구조의 장점은 분명했습니다.

첫째, 토큰을 localStorage나 메모리 상태에 직접 들고 있지 않아도 됩니다.  
둘째, 요청을 보낼 때마다 헤더를 수동으로 붙이지 않아도 됩니다.  
셋째, 토큰 저장 책임이 클라이언트보다 서버와 브라우저 쪽에 더 가깝게 남습니다.

대신 프론트엔드가 반드시 책임져야 하는 부분이 하나 생깁니다.  
바로 인증 실패 이후의 복구 흐름입니다.

즉, “토큰을 어떻게 저장할지”보다 “401이 왔을 때 원래 요청을 어떻게 살릴지”가 핵심이 되었습니다.

## 응답 인터셉터에서 reissue 처리하기

`reissue`를 API 클라이언트의 응답 인터셉터에서 공통 처리하도록 두었습니다.

```ts
client.interceptors.response.use(async (response, _, options) => {
  // 401이 아니면 그대로 통과
  if (response.status !== 401) return response;

  // 로그인/로그아웃/reissue 자체는 재발급 대상에서 제외
  const isExcluded = EXCLUDE_PATHS.some((path) => options.url?.includes(path));
  if (isExcluded) throw response;

  // 원래 요청을 다시 보내기 위해 요청 정보를 복구
  const _fetch = options.fetch!;
  const url = buildUrl(options);
  const requestInit: RequestInit = {
    ...options,
    headers: options.headers as HeadersInit,
    body: getValidRequestBody(options) as BodyInit | null | undefined,
  };

  // 이미 다른 요청이 reissue 중이면 그 결과를 기다렸다가 재시도
  if (isRefreshing && reissuePromise) {
    await reissuePromise;
    return await _fetch(url, requestInit);
  }

  // 첫 번째 401 요청만 실제 reissue를 수행
  isRefreshing = true;
  reissuePromise = reissueToken();

  try {
    // 재발급이 끝나면 원래 요청을 다시 전송
    await reissuePromise;
    return await _fetch(url, requestInit);
  } finally {
    // 성공/실패와 관계없이 reissue 상태 초기화
    isRefreshing = false;
    reissuePromise = null;
  }
});
```

## 동시에 여러 요청이 실패하는 상황을 어떻게 막았는가

이번 구현에서 가장 중요했던 부분은 single-flight에 가까운 제어였습니다.  
즉, 재발급은 한 번만 수행하고, 나머지 요청은 그 결과를 기다리게 만드는 방식입니다.

이를 위해 아래 두 값을 사용했습니다.

```ts
let isRefreshing = false;
let reissuePromise: ReturnType<typeof reissueToken> | null = null;
```

흐름은 단순합니다.

1. 첫 번째 `401` 요청이 들어오면 `reissueToken()`을 실행합니다.
2. 그동안 들어오는 다른 `401` 요청은 새로운 재발급을 시도하지 않습니다.
3. 대신 이미 진행 중인 `reissuePromise`를 기다립니다.
4. 재발급이 끝나면 대기 중이던 요청들이 원래 요청을 다시 보냅니다.

이 방식이 좋았던 이유는 재발급을 “토큰 만료 이벤트”로 다루는 대신, “공유 가능한 비동기 작업”으로 다뤘기 때문입니다.  
토큰이 만료된 순간마다 새로 재발급을 날리는 것이 아니라, 한 번 시작된 재발급을 기준으로 다른 요청들이 합류하는 구조였습니다.

## 재발급 대상에서 제외한 요청들

모든 `401`에 대해 무조건 재발급을 시도하지는 않았습니다.  
아래 경로는 예외로 두었습니다.

```ts
const EXCLUDE_PATHS = ['/auth/sign-in', '/auth/sign-out', '/auth/reissue'];
```

이 예외 처리는 생각보다 중요했습니다.

예를 들어 로그인 요청 자체가 실패했는데 다시 재발급을 시도하는 것은 의미가 없습니다.  
로그아웃도 마찬가지입니다.  
그리고 `reissue` 요청이 실패했을 때 다시 `reissue`를 시도하면 사실상 무한 루프에 가까운 흐름이 만들어질 수 있습니다.

그래서 “인증 복구를 위한 요청”과 “복구 대상이 되는 요청”을 분리하는 것이 필요했습니다.  
이 구분이 없으면 인터셉터는 편리해 보여도, 실패 시나리오에서 오히려 더 위험해질 수 있습니다.

## 원래 요청을 다시 보내는 과정에서 신경 쓴 점

이번 구현에서 생각보다 까다로웠던 부분은 “재발급 후 무엇을 다시 보낼 것인가”였습니다.

인터셉터 안에서 재시도할 때는 원래 요청을 최대한 그대로 복구해야 합니다.  
URL만 다시 만들면 끝나는 문제가 아니었습니다.  
이미 직렬화된 `headers`, `body`, `fetch` 옵션을 다시 사용해야 했고, 요청 본문도 안전하게 재구성해야 했습니다.

그래서 아래처럼 기존 hey-api에서 generated 된 `buildUrl`, `getValidRequestBody`를 사용해 재시도용 `RequestInit`을 다시 만들었습니다.

```ts
const _fetch = options.fetch!;
const url = buildUrl(options);
const requestInit: RequestInit = {
  ...options,
  headers: options.headers as HeadersInit,
  body: getValidRequestBody(options) as BodyInit | null | undefined,
};
```

이 부분에서 느낀 점은, reissue 로직은 결국 “토큰 갱신” 문제가 아니라 “실패한 요청 재생” 문제에 더 가깝다는 것이었습니다.  
재발급이 성공해도 원래 요청을 제대로 복구하지 못하면 사용자 입장에서는 결국 실패로 느껴집니다.

## 마무리

쿠키 기반 인증에서 이번에 가장 중요하게 다룬 것은 reissue였습니다.
토큰을 어디에 저장하느냐보다, 만료된 뒤의 요청을 어떻게 다시 살릴 것이냐가 훨씬 더 중요한 문제라고 느꼈습니다.

특히 다음 세 가지가 핵심이었다고 생각합니다.

- 인증 상태는 Cookie 기반으로 단순하게 유지할 것
- `401` 대응은 화면이 아니라 API 클라이언트 레벨에서 공통 처리할 것
- 재발급은 중복 실행하지 않고 하나의 Promise로 합류시킬 것

이 구조 덕분에 인증 로직이 여러 화면으로 퍼지지 않았고, 동시에 여러 요청이 실패하는 상황도 안정적으로 다룰 수 있었습니다.
