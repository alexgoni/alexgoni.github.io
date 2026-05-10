---
layout: single
title: 'API 호출 관련 베스트 프랙티스에 대한 회고'
categories: Gradpath
excerpt: hey-api, msw, suspensive
---

프론트엔드에서 API를 다루는 방식에는 여러가지가 있습니다.
이번 프로젝트를 진행하면서 지금까지 경험한 방식 중 DX, 안정성 측면에서 가장 좋은 방식이라고 생각이 되어 회고하고자 합니다.

## 1. OpenAPI 생성 코드로 반복 작업 줄이기

프론트엔드에서 API를 붙일 때 가장 피로한 부분 중 하나는 반복입니다.
같은 엔드포인트를 붙이기 위해 타입을 만들고, 요청과 응답 스키마를 정의하고, 검증 규칙을 연결하고, React Query 쿼리와 mutation을 작성하는 일이 계속 반복됩니다.

초기에는 이 작업이 익숙해서 크게 문제처럼 느껴지지 않을 수 있습니다.
하지만 페이지 수가 늘어나고 API 수가 많아질수록, 이 반복은 단순한 노동이 아니라 유지보수 비용으로 바뀝니다.
특히 백엔드 스펙이 바뀌었을 때 타입, validation, 쿼리 코드가 서로 다른 타이밍에 수정되면 프론트엔드 내부에서 계약 불일치가 생기기 쉽습니다.

이때 도움이 되었던 것이 OpenAPI와 hey-api였습니다.

OpenAPI는 서버 API를 사람이 읽는 문서에만 머무르게 하지 않고, 기계가 읽을 수 있는 스펙으로 정의하는 방식입니다.
쉽게 말해 백엔드의 요청 파라미터, 응답 구조, 에러 스키마를 하나의 계약으로 다루는 표준이라고 생각하시면 됩니다.

그리고 `@hey-api/openapi-ts`는 그 계약을 바탕으로 프론트엔드에서 바로 사용할 수 있는 코드를 생성해 주는 도구였습니다.
이번 프로젝트에서는 단순 타입 생성에만 그치지 않고, React Query에서 사용할 query option과 mutation, 그리고 valibot 기반 validation 스키마까지 함께 생성하도록 사용했습니다.

프로젝트에서 잡았던 플로우는 단순했습니다.

1. 백엔드에서 제공하는 OpenAPI 문서를 내려받습니다.
2. 내려받은 문서를 프로젝트 내부에 저장합니다.
3. hey-api를 실행해 타입, 스키마, React Query 헬퍼 코드를 생성합니다.
4. 프론트엔드는 생성된 코드를 그대로 import 해서 사용합니다.

이 방식의 가장 큰 장점은 프론트엔드가 API를 직접 정의하지 않고 계약을 소비하게 된다는 점이었습니다.
예를 들어 특정 API를 붙일 때마다 쿼리 키를 수동으로 만들거나 응답 타입을 따로 적지 않아도 되었고, 생성된 코드에서 바로 `...Options()`, `...Mutation()`을 가져다 쓸 수 있었습니다.
결과적으로 API 연동 코드의 스타일이 팀 내에서 자연스럽게 통일되었습니다.

사용하는 쪽의 코드도 훨씬 단순해졌습니다.
예를 들어 라우터에서 인증 정보를 미리 받아오는 코드도 직접 fetch 함수를 만들 필요 없이 생성된 query option을 그대로 사용할 수 있었습니다.

```ts
const protectedRoutes = (queryClient: QueryClient) => async () => {
  await queryClient.ensureQueryData(myAuthOptions());
  return null;
};
```

mutation도 비슷했습니다.
로그인처럼 자주 쓰는 액션도 생성된 mutation을 가져와 바로 연결하면 되었습니다.

```ts
const mutation = useMutation({
  ...signInMutation(),
  onSuccess: ({ status }) => {
    if (status === 'APPROVED') {
      navigate('/', { replace: true });
    }
  },
});
```

또 하나 좋았던 점은 validation과의 연결이었습니다.
생성된 valibot 스키마를 그대로 활용할 수 있었기 때문에, 프론트엔드에서 별도로 타입을 다시 정의하거나 검증 규칙을 이중으로 작성할 필요가 줄었습니다.
특히 이 프로젝트에서는 validation을 화면 코드에서 매번 수동으로 붙이기보다, 생성된 API 요청 함수 내부에서 바로 수행되도록 가져갈 수 있었습니다.

```ts
export const signIn = <ThrowOnError extends boolean = false>(
  options: Options<SignInData, ThrowOnError>,
) =>
  (options.client ?? client).post<SignInResponses, SignInErrors, ThrowOnError>({
    requestValidator: async (data) => await v.parseAsync(vSignInData, data),
    responseValidator: async (data) =>
      await v.parseAsync(vSignInResponse2, data),
    url: '/auth/sign-in',
    ...options,
  });
```

요청 데이터 스키마도 함께 생성되기 때문에, 어떤 형태의 데이터를 보내야 하는지 역시 생성 코드 안에서 일관되게 관리할 수 있었습니다.

```ts
export const vSignInData = v.object({
  body: vSignInRequest,
  path: v.optional(v.never()),
  query: v.optional(v.never()),
});
```

<br />

백엔드 수정 사항이 OpenAPI 문서 상에 반영이 되어야한다는 작은 불편함은 있었지만,  
API를 붙일 때마다 비슷한 보일러플레이트를 복사하는 일은 크게 줄었고, 백엔드 스펙 변경이 프론트 전체에 어떤 영향을 주는지 훨씬 빠르게 추적할 수 있게 되었습니다.

## 2. MSW로 백엔드 준비 여부와 무관하게 개발하기

실무에서 프론트엔드 개발이 항상 백엔드보다 늦게 시작되거나, 반대로 백엔드가 완전히 준비된 뒤에만 가능한 것은 아닙니다.
오히려 실제 프로젝트에서는 API가 일부만 준비되어 있거나, 응답 형태가 자주 바뀌거나, 특정 엔드포인트만 임시로 막혀 있는 경우가 더 많습니다.

이런 상황에서 가장 흔한 문제는 개발 흐름이 멈춘다는 점입니다.
화면은 만들 수 있는데 데이터를 붙일 수 없고, 컴포넌트는 쪼갤 수 있는데 실제 사용자 흐름은 검증할 수 없고, 결국 프론트엔드가 백엔드 일정에 지나치게 종속됩니다.

플로우는 다음과 같습니다.

```ts
// main.tsx
if (import.meta.env.DEV) {
  const [{ startWorker }] = await Promise.all([import('@/app/providers/msw')]);
  await startWorker();
}
```

개발 서버가 올라오면 먼저 브라우저에서 service worker를 등록합니다.

그 다음 실제 화면에서 API 요청이 발생하면, 그 요청은 바로 서버로 가지 않고 먼저 MSW handler 목록과 매칭됩니다.
예를 들어 알림 목록을 불러오는 요청이 들어오면 아래와 같은 handler가 먼저 그 요청을 받습니다.

```ts
http.get(
  `${baseUrl}/notifications`,
  withMock('notifications', async () => {
    return HttpResponse.json([
      {
        id: '1',
        title: 'mock title',
        body: 'mock body',
      },
    ]);
  }),
);
```

여기서 핵심은 `withMock` 이었습니다.
handler가 무조건 mock 응답을 주는 것이 아니라, 먼저 whitelist 설정을 보고 “이 API를 지금 mock으로 받을지”를 결정하게 됩니다.

```ts
export const withMock = (apiKey, handler) => {
  return async (args) => {
    const isConfigEnabled = MOCK_CONFIG[apiKey];

    if (!isConfigEnabled) {
      return passthrough();
    }

    return handler(args);
  };
};
```

```ts
export const MOCK_CONFIG = {
  'auth-sign-in': false,
  'users-me': false,
  'students-documents': false,
  'notifications-unread-count': false,
  'notifications-device-token': false,
} as const;
```

즉 실제 흐름은 아래 순서와 같이 진행됩니다.

1. 화면에서 fetch 또는 React Query 요청이 발생합니다.
2. 브라우저에 등록된 MSW service worker가 요청을 먼저 가로챕니다.
3. 요청 URL에 맞는 handler가 있는지 확인합니다.
4. handler 내부의 `withMock`이 whitelist 설정을 확인합니다.
5. 설정값이 `true`이면 mock 데이터를 응답합니다.
6. 설정값이 `false`이면 `passthrough()`로 실제 서버 요청을 그대로 보냅니다.

이 구조가 좋았던 이유는 전체 mock 모드와 전체 실서버 모드 사이에 더 현실적인 중간 지점을 만들 수 있었기 때문입니다.
실제 개발에서는 어떤 API는 백엔드가 준비되어 있고, 어떤 API는 아직 없고, 어떤 API는 응답 스펙만 먼저 맞춰보고 싶을 때가 많습니다.
이때 whitelist 방식은 필요한 부분만 골라 mock 하면서도 나머지는 실서버와 붙일 수 있게 해 주었습니다.

결과적으로 프론트엔드는 도메인 흐름을 먼저 완성할 수 있었고, 백엔드 연동이 끝나기 전에도 화면 구조나 상태 흐름, 에러 처리, 사용자 경험을 충분히 검토할 수 있었습니다.

## 3. ErrorBoundary와 Suspense로 로딩과 에러 처리 패턴화하기

API 호출이 많아질수록 또 하나 복잡해지는 부분이 로딩과 에러 처리였습니다.
화면 수가 적을 때는 각 페이지에서 `isLoading`, `isError`를 직접 분기해도 크게 문제가 되지 않습니다.
하지만 페이지가 늘어나고, 다이얼로그 안에서도 비동기 요청이 발생하고, 인증이나 권한 같은 전역 의미의 에러까지 섞이기 시작하면 화면마다 처리 방식이 제각각 달라지기 쉽습니다.

이번 프로젝트에서는 이 문제를 `suspensive`와 함께 ErrorBoundary, Suspense 조합으로 풀어보았습니다.
핵심은 로딩과 에러를 개별 화면이 알아서 처리하는 문제가 아니라, 공통 정책으로 다루는 문제로 바꾸는 것이었습니다.

또 백엔드의 에러 응답이 비교적 일관된 형태를 가지고 있었기 때문에, 이 점도 공통 처리 전략을 세우는 데 큰 도움이 되었습니다.
예를 들어 에러 응답은 아래와 같은 형태로 내려왔습니다.

```
{
  "origin": "COMMON",
  "code": 100,
  "message": "User not found"
}
```

이렇게 `origin`, `code`, `message`가 일정하게 내려오면 프론트엔드도 “이 에러를 어떤 레벨에서 처리할지”를 코드로 분기하기 쉬워집니다.
그래서 이번 프로젝트에서는 공통 ErrorBoundary를 두고, 같은 코드의 에러는 항상 같은 방식으로 처리하도록 만들었습니다.

```tsx
export function ErrorResetBoundary({ children }: PropsWithChildren) {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          fallback={({ error, reset }) => {
            const errorResponse = toError(error);
            if (errorResponse.type === 'API') {
              if (isCriticalApiError(errorResponse.code)) throw error;
              if (isCompletionRequiredError(errorResponse.code)) {
                return (
                  <Navigate to="/professor/profile/convert/external" replace />
                );
              }
              if (isRetryableApiError(errorResponse.code)) {
                return <DataLoadingPageError reset={reset} />;
              }
            }
            throw error;
          }}
        >
          {children}
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

이 패턴에서 좋았던 점은 “무조건 한 화면 안에서 해결하려고 하지 않았다”는 점이었습니다.
예를 들어 재시도 가능한 네트워크 문제는 재시도 UI로 보내고, 인증 실패나 권한 실패처럼 전역적 의미가 있는 에러는 상위 에러 페이지로 전파하고, 일부 예상 가능한 비즈니스 에러는 해당 화면에서만 국소적으로 처리하는 식으로 역할을 나눌 수 있었습니다.

로딩 처리도 마찬가지였습니다.
컴포넌트마다 `if (isLoading)`를 반복해서 쓰기보다, Suspense 경계에서 fallback UI를 처리하게 하니 실제 화면 컴포넌트는 “데이터가 준비된 이후의 상태”에 더 집중할 수 있었습니다.
즉, 본문 컴포넌트는 성공 케이스에 집중하고, 로딩과 실패는 바깥 경계가 맡는 구조에 가까워졌습니다.

결과적으로 얻은 가장 큰 장점은 일관성이었습니다.
사용자 입장에서는 비슷한 종류의 실패를 만났을 때 비슷한 방식으로 안내를 받게 되었고, 개발자 입장에서는 “이 에러는 어디서 처리해야 하는가”를 매번 새로 고민하기보다 이미 정해둔 경계 안에서 판단할 수 있게 되었습니다.

## 마무리

이번 프로젝트에서 API 호출 관련 베스트 프랙티스를 정리해 보면 결국 세 문장으로 요약할 수 있을 것 같습니다.

첫째, 계약은 사람이 반복해서 쓰기보다 생성 코드로 가져오는 편이 낫습니다.
둘째, 백엔드 준비 상태와 무관하게 프론트엔드가 움직일 수 있어야 개발 속도가 유지됩니다.
셋째, 로딩과 에러는 화면마다 따로 처리하기보다 공통 패턴으로 관리하는 편이 훨씬 안정적입니다.

OpenAPI 생성 코드, MSW, ErrorBoundary와 Suspense는 각각 다른 문제를 해결하는 도구처럼 보이지만, 실제로는 모두 같은 방향을 향하고 있었습니다.
프론트엔드가 API 앞에서 덜 흔들리고, 더 예측 가능하게 동작하도록 만드는 방향이었습니다.

돌이켜보면 이번 프로젝트에서 가장 크게 배운 점은, 잘 만든 API 호출 코드는 단순히 요청을 잘 보내는 코드가 아니라 변경과 실패를 잘 견디는 코드라는 사실이었습니다.
특히 비교적 짧은 시간 안에 완성해야 했던 프로젝트였기 때문에, 이런 전략이 없었다면 개발 속도와 안정성을 동시에 가져가기 어려웠을 것이라고 생각합니다.
좋은 경험을 얻었던 방향이였던만큼 앞으로도 다듬어서 다른 프로젝트에도 이러한 전략을 게속해서 적용하고 싶습니다.
