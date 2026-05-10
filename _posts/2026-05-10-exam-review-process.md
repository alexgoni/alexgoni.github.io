---
layout: single
title: 심사 제출 process
categories: Gradpath
excerpt: PASS 인증 때문에 한 번 끊기는 심사 제출 흐름을 sessionStorage로 복구한 방법
---

이번 프로젝트에서 심사 제출은 단순히 폼을 작성하고 `POST` 요청을 보내는 흐름이 아니었습니다.
최종 제출 전에 전자서명을 등록하고, 이어서 PASS 본인인증까지 완료해야만 실제 제출이 가능했기 때문입니다.

겉으로 보기에는 "제출 버튼을 누르면 인증하고 제출한다"는 단순한 흐름처럼 보이지만, 실제 구현에서는 한 가지 중요한 문제가 있었습니다.

**PASS 인증은 현재 입력 중인 화면 바깥으로 한 번 나갔다가 다시 돌아오는 흐름**이라는 점입니다.

즉, 사용자가 심사 결과를 입력하던 도중

- 서명 이미지를 업로드하고
- PASS 인증을 요청하고
- 외부 인증 플로우를 거친 뒤
- 다시 원래 심사 화면으로 돌아와
- 그제야 최종 제출을 해야 했습니다

이 구조에서는 제출 직전의 폼 값을 어딘가에 안전하게 저장해 두지 않으면, 인증을 마치고 돌아왔을 때 다시 제출을 이어갈 수 없습니다.

이번 글에서는 왜 이런 전략이 필요했는지, 그리고 현재 프로젝트에서 이를 어떻게 구현했는지 정리해보려고 합니다.

## 왜 바로 POST할 수 없었는가

보통 폼 제출은 아래처럼 생각하기 쉽습니다.

```text
폼 입력 완료 ---> 제출 버튼 클릭 ---> POST 요청 ---> 완료
```

하지만 이번 심사 제출은 중간에 PASS 인증이 끼어 있었습니다.

```text
폼 입력 완료
    ---> 서명 등록
    ---> PASS 인증 요청
    ---> 외부 인증 플로우 이동
    ---> 인증 완료 후 원래 화면 복귀
    ---> 제출 요청
```

문제는 최종 제출 API에 `passToken`이 필요하다는 점이었습니다.
이 `passToken`은 심사 화면에 처음 들어왔을 때 이미 가지고 있는 값이 아니라, PASS 인증을 마친 뒤에야 받을 수 있는 값입니다.

즉, 제출에 필요한 데이터는 두 시점에 나뉘어 생깁니다.

- 심사 폼 데이터: 사용자가 현재 화면에서 입력
- `passToken`: PASS 인증 완료 후 복귀하면서 획득

이 둘이 같은 시점에 존재하지 않기 때문에, 그냥 제출 버튼을 누르면 바로 `POST`하는 방식으로는 해결할 수 없었습니다.

## 왜 sessionStorage가 필요했는가

이 문제를 해결하려면 PASS 인증을 보내기 직전에 현재 입력값을 어딘가에 저장해 두고, 인증이 끝난 뒤 돌아왔을 때 다시 꺼내와야 했습니다.

여기서 후보는 몇 가지가 있을 수 있습니다.

- React state에만 들고 있기
- 전역 상태 저장소에 올리기
- URL 파라미터나 `location.state`로 넘기기
- `sessionStorage`에 저장하기

이번 프로젝트에서는 `sessionStorage`를 선택했습니다.

이 선택이 적절했던 이유는 꽤 분명했습니다.

첫째, PASS 인증은 현재 화면 흐름이 한 번 끊기는 구조였습니다.
단순한 컴포넌트 state는 이런 흐름을 안전하게 넘기기 어렵습니다.

둘째, 저장해야 하는 값이 단순한 토큰 하나가 아니라, 심사 결과 입력값 전체였습니다.
이를 URL이나 라우터 state로 직접 넘기는 것은 부담이 컸습니다.

셋째, 이 데이터는 장기 보관이 아니라 "이번 탭에서, 이번 제출 흐름 동안만" 살아 있으면 되는 성격이었습니다.
그래서 `localStorage`보다 `sessionStorage`가 더 적절했습니다.

즉, 이번 문제에서 `sessionStorage`는 범용 저장소라기보다 **외부 인증 플로우를 사이에 둔 일시적인 제출 복원 장치**에 가까웠습니다.

## 실제 구현 흐름

전체 흐름은 아래처럼 구성되어 있습니다.

```text
1. 사용자가 심사 폼 작성
2. 전자서명 다이얼로그에서 서명 이미지 업로드
3. PASS 인증 직전에 현재 폼 값을 sessionStorage에 저장
4. PASS 인증 요청
5. 인증 완료 후 원래 심사 화면으로 복귀하면서 token 획득
6. 저장해둔 draft + passToken을 합쳐 최종 제출
7. 제출 성공 시 draft 삭제
```

이 흐름의 핵심은 "PASS 인증 전 저장"과 "복귀 후 자동 제출" 두 단계였습니다.

## 1. PASS 인증 전에 draft를 저장했습니다

전자서명 다이얼로그에서 제출 버튼을 누르면, 먼저 서명 이미지를 업로드한 뒤 현재 폼 값을 `sessionStorage`에 저장합니다.
그 다음 PASS 인증을 요청합니다.

```tsx
<SignatureSubmitButton
  onClick={() => {
    // PASS 인증으로 화면을 벗어나기 전에 현재 입력값을 먼저 저장합니다.
    saveDraft(getFormValues());
    onClose();
    // draft 저장이 끝난 뒤 PASS 인증을 시작합니다.
    requestPassAuth();
  }}
>
  완료
</SignatureSubmitButton>
```

즉, PASS 인증으로 흐름이 넘어가기 전에

- 심사 입력값
- 업로드된 서명 파일 key

를 먼저 저장해 두는 구조입니다.

이 순서가 없으면 인증을 마치고 돌아왔을 때, 사용자가 직전에 입력한 내용과 업로드한 서명 정보를 다시 복원할 수 없습니다.

## 2. draft는 sessionStorage에 저장했습니다

draft 저장 자체는 비교적 단순하게 구성되어 있습니다.

```ts
export function useExamReviewDraft(storageKey: string) {
  const saveDraft = useCallback(
    (values: SubmitExamReviewRequest) => {
      // 제출 직전 폼 값을 현재 탭의 sessionStorage에 저장합니다.
      sessionStorage.setItem(storageKey, JSON.stringify(values));
    },
    [storageKey],
  );

  const loadDraft = useCallback((): Partial<SubmitExamReviewRequest> | null => {
    // PASS 인증 후 돌아왔을 때 이전에 저장한 draft를 다시 읽어옵니다.
    const saved = sessionStorage.getItem(storageKey);
    return saved ? JSON.parse(saved) : null;
  }, [storageKey]);

  const clearDraft = useCallback(() => {
    // 제출이 끝난 뒤에는 남아 있는 draft를 정리합니다.
    sessionStorage.removeItem(storageKey);
  }, [storageKey]);

  return { saveDraft, loadDraft, clearDraft };
}
```

심사 결과는 `examResultId` 기준으로 저장 키를 나눴습니다.

```ts
export const STORAGE_KEY = {
  examReviewDraft: (examResultId: string) =>
    `exam-review-draft:${examResultId}`,
} as const;
```

이렇게 해두면 여러 심사 흐름이 섞이지 않고, 어떤 심사 결과에 대한 draft인지도 명확하게 구분할 수 있습니다.

## 3. PASS 인증이 끝나면 token을 들고 원래 화면으로 돌아옵니다

PASS 인증이 끝나면 검증 페이지에서 결과를 확인하고, 원래 심사 화면으로 다시 이동시킵니다.
이때 `token`은 라우터 state로 함께 넘깁니다.

```ts
if (flow === 'exam') {
  // PASS 인증이 끝나면 token을 라우터 state에 담아 원래 심사 화면으로 돌려보냅니다.
  navigate(getPassAuthReturnPath(flow, params), {
    state: {
      token: data.data.token,
    },
    replace: true,
  });
}
```

즉, 심사 화면으로 다시 돌아왔을 때는

- 라우터 state에는 `passToken`
- `sessionStorage`에는 이전에 저장한 draft

가 각각 남아 있게 됩니다.

이제야 최종 제출에 필요한 두 조각이 한곳에 모이는 셈입니다.

## 4. 돌아온 뒤에는 draft와 token을 합쳐 자동 제출했습니다

`useSubmitExamReview` 훅에서는 복귀 후 이 두 값을 합쳐 최종 제출을 수행합니다.

```ts
function useSubmitExamReview({
  examResultId,
  storageKey,
  onSuccess,
}: {
  examResultId: string;
  storageKey: string;
  onSuccess: () => void;
}) {
  const { getState, setState } = useTypedNavigate<{ token: string }>();
  // PASS 인증을 마치고 돌아오면서 받은 token입니다.
  const passToken = getState()?.token;
  // 인증 전에 저장해 둔 심사 draft를 다룹니다.
  const { loadDraft, clearDraft } = useExamReviewDraft(storageKey);

  const { mutate } = useMutation({
    ...submitExamReviewMutation(),
    onSuccess: () => {
      // 제출 성공 후에는 draft와 라우터 state를 모두 비웁니다.
      clearDraft();
      setState(null);
      toast({ message: EXAM_RESULT_TEXT_MAP.evaluationSuccess });
      onSuccess();
    },
  });

  useEffect(() => {
    // 화면에 다시 진입하면 저장해 둔 draft를 먼저 읽어옵니다.
    const draft = loadDraft();
    if (!(passToken && draft)) return;

    // draft와 PASS token을 합쳐 최종 제출 payload를 복원합니다.
    const parsed = v.safeParse(vSubmitExamReviewRequest, {
      passToken,
      ...draft,
    });

    if (parsed.success) {
      // 조건이 모두 갖춰졌다면 사용자의 추가 액션 없이 자동 제출합니다.
      mutate({
        path: { examResultId },
        body: parsed.output,
      });
    }
  }, [examResultId, loadDraft, mutate, passToken]);
}
```

이 훅의 핵심은 `useEffect`입니다.

심사 화면에 다시 진입했을 때

- `passToken`이 있고
- 저장해둔 draft도 있으면

그 둘을 합쳐 validation 한 뒤, 자동으로 `submitExamReview` mutation을 실행합니다.

즉, 사용자는 PASS 인증을 마치고 돌아온 뒤 다시 한 번 제출 버튼을 누를 필요가 없습니다.
프론트가 이전 흐름을 이어서 복구해 주는 구조입니다.

## 마무리

이번 심사 제출 구현에서 가장 중요했던 것은 PASS 인증 때문에 한 번 끊기는 흐름을 어떻게 다시 이어 붙일 것인가였습니다.

- PASS 인증 전에 현재 심사 입력값을 `sessionStorage`에 저장할 것
- 인증 완료 후 `passToken`만 들고 원래 화면으로 복귀할 것
- 복귀 후에는 저장된 draft와 token을 합쳐 자동으로 제출할 것

돌이켜보면 이 구조 덕분에 사용자는 외부 인증을 거치고도 제출 흐름이 끊겼다고 느끼지 않을 수 있었습니다.
그리고 프론트엔드 입장에서도 "인증이 끼어 있는 제출"을 단순한 버튼 클릭이 아니라, 단계가 있는 흐름으로 설계해야 한다는 점을 분명히 배울 수 있었습니다.
