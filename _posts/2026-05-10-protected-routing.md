---
layout: single
title: react-router loader를 통한 보호 라우팅
categories: Gradpath
excerpt: 사용자 타입에 따른 페이지 접근 제한
---

## 왜 인증/권한 라우팅이 따로 중요했는가

로그인 여부에 따라 접근 가능한 페이지가 달라지는 정도라면, 화면이 렌더링된 뒤에 사용자 정보를 확인하고 리다이렉트해도 큰 문제는 없습니다.
하지만 Gradpath 프로젝트의 경우 단순히 “로그인했는가”만 확인하면 끝나는 구조가 아니었습니다.

- 학생과 교수의 접근 페이지가 달랐고
- 학생은 현재 심사 단계에 따라 접근 가능한 메뉴가 달랐습니다.

이런 구조에서 컴포넌트가 렌더링된 뒤에 `useEffect`로 권한을 확인하고 이동시키는 방식이 아닌
“화면을 보여주기 전에 인증과 권한을 먼저 확인하는 흐름”이 필요했습니다.

## 왜 loader를 중심으로 풀었는가

이번 프로젝트에서는 인증과 권한 체크를 라우터 `loader`에서 먼저 수행했습니다.
핵심은 페이지가 렌더링된 뒤에 권한을 확인하는 것이 아니라, 라우트에 진입하기 전에 먼저 걸러내는 것이었습니다.

핵심 아이디어는 단순했습니다.

1. 라우트 진입 시점에 인증 정보를 먼저 조회합니다.
2. 사용자 타입이나 권한이 맞지 않으면 컴포넌트를 렌더링하기 전에 리다이렉트합니다.
3. 필요한 경우 추가 권한 정보를 조회해 세부 접근 가능 여부를 판단합니다.

예를 들어 기본적인 보호 라우트는 아래와 같이 구성했습니다.

```ts
const protectedRoutes = (queryClient: QueryClient) => async () => {
  await queryClient.ensureQueryData(myAuthOptions());
  return null;
};
```

이 구조에서 중요한 점은 `myAuthOptions()`가 단순히 “내 정보 조회” 요청이 아니라, 라우트 진입의 전제 조건 역할을 한다는 점입니다.
즉 여기서 중요한 것은 `ensureQueryData` 자체보다, 이 작업이 컴포넌트가 아니라 `loader` 안에서 수행된다는 점이었습니다.
React Query는 필요한 서버 데이터를 안정적으로 가져오는 역할을 했고, 실제 보호 라우팅의 중심은 `loader`가 맡았습니다.

## 학생/교수 라우트를 분리하면서 인가를 구체화하기

이번 프로젝트에서는 인증만으로 끝나지 않고 사용자 타입도 함께 검증해야 했습니다.
학생 전용 페이지에 교수 계정이 들어가면 안 되고, 반대로 교수 전용 페이지에 학생 계정이 들어가도 안 되는 구조였기 때문입니다.

이 부분은 비교적 직관적으로 구현할 수 있었습니다.

```ts
const protectedProfessorRoutes = (queryClient: QueryClient) => async () => {
  const data = await queryClient.ensureQueryData(myAuthOptions());
  if (data.userType === 'STUDENT') return redirect('/');
  return null;
};
```

학생 라우트는 한 단계 더 나갔습니다.
학생 여부만 확인하면 끝나는 것이 아니라, 현재 심사 단계와 접근 가능한 메뉴를 함께 봐야 했기 때문입니다.

```ts
const protectedStudentRoutes =
  (queryClient: QueryClient) =>
  async ({ request }: LoaderFunctionArgs) => {
    const data = await queryClient.ensureQueryData(myAuthOptions());

    // 학생 전용 라우트이므로, 학생이 아니면 메인으로 돌려보냅니다.
    if (data.userType !== 'STUDENT') return redirect('/');

    const { pathname } = new URL(request.url);

    // 학생에게 허용된 메뉴 목록을 조회합니다.
    const accessibleMenus = await queryClient.fetchQuery(
      getStudentAccessibleMenusOptions(),
    );

    // 현재 경로가 허용된 메뉴에 포함되는지 확인합니다.
    const result = checkStudentPathAccess(accessibleMenus, pathname);

    // 접근이 불가능하면 안내 메시지를 보여주고 학생 메인으로 리다이렉트합니다.
    if (result && !result.allowed) {
      toast({ message: result.message });
      return redirect('/student');
    }

    // 모든 조건을 통과한 경우에만 페이지 렌더링을 허용합니다.
    return null;
  };
```

여기서 중요했던 것은 학생/교수 분기 자체를 페이지 컴포넌트 안에서 처리하지 않았다는 점입니다.
즉 “누가 이 페이지에 들어올 수 있는가”를 화면 로직이 아니라 라우터 정책으로 끌어올린 셈이었습니다.
React Query는 이 판단에 필요한 인증 정보와 접근 가능 메뉴를 조회하는 역할로 사용되었습니다.

## 권한 체크를 화면이 아니라 라우터에 두었을 때의 장점

이 방식을 쓰면서 가장 크게 느낀 장점은 책임 분리가 선명해졌다는 점이었습니다.

컴포넌트는 화면을 그리는 데 집중하고,  
라우터는 이 화면에 들어와도 되는지 판단하는 데 집중하게 되었습니다.

이 차이는 생각보다 컸습니다.

만약 권한 체크를 각 페이지 컴포넌트에서 직접 처리했다면 다음과 같은 코드가 곳곳에 생겼을 가능성이 큽니다.

- 내 정보 조회
- 사용자 타입 확인
- 현재 경로와 권한 비교
- 실패 시 토스트
- 실패 시 navigate

이런 흐름이 여러 페이지에 반복되면, 나중에는 “이 규칙이 어디서 적용되고 있는지” 추적하는 것 자체가 부담이 됩니다.

반대로 `loader`에 모아두면, 적어도 라우팅 단계에서의 인증/권한 정책은 한 군데에서 읽히게 됩니다.
서비스가 커질수록 이 일관성이 꽤 중요하다고 느꼈습니다.
