---
layout: single
title: presigned URL
categories: Gradpath
excerpt: 서버를 거치지 않고 파일 업로드
---

## presigned URL

presigned URL은 클라이언트가 클라우드 스토리지 서비스(S3 등)에 직접 접근해 파일을 업로드하거나 다운로드할 수 있도록 만들어진 임시 보안 URL입니다.

보통 스토리지의 파일은 보안상 비공개로 유지됩니다. presigned URL은 그중 특정 파일에 대해서만, 정해진 시간 동안 한시적으로 접근을 허용하는 일종의 "임시 출입증"에 가깝습니다.

이 방식의 핵심은 파일 본문이 서버를 거치지 않고, 클라이언트와 스토리지가 직접 통신한다는 점입니다.

1. URL 요청: 클라이언트가 서버에 특정 파일에 대한 접근 권한을 요청합니다.
2. URL 생성: 서버는 자신의 자격 증명을 사용해 유효 기간과 허용 작업이 포함된 서명 URL을 생성합니다.
3. URL 전달: 생성된 URL을 클라이언트에게 응답합니다.
4. 직접 통신: 클라이언트는 해당 URL로 스토리지에 직접 업로드하거나 다운로드합니다.

파일 크기가 커질수록 서버를 거쳐 파일을 전송하는 방식은 대역폭과 서버 자원을 많이 소모하게 됩니다. 반면 presigned URL 방식에서는 서버가 파일 자체를 중계하지 않고 URL만 발급하므로, 서버 부하를 줄이면서도 필요한 권한 제어를 유지할 수 있습니다.

## 파일 업로드 플로우

일반적인 presigned URL 업로드 흐름은 아래와 같습니다.

```text
[브라우저]
  파일 선택
      |
      v
[서버]
  presigned PUT URL 발급 요청
      |
      v
[브라우저]
  presigned PUT URL 수신
      |
      v
[S3 / 스토리지]
  파일 바이너리 직접 업로드
      |
      v
[브라우저]
  업로드 완료 후 key 확보
      |
      v
[서버]
  이후 비즈니스 요청에서 key 전달
```

1. 클라이언트가 백엔드에 “이 파일을 업로드하고 싶다”고 요청합니다.
2. 백엔드는 업로드 경로와 파일명을 바탕으로 presigned PUT URL을 발급합니다.
3. 클라이언트는 그 URL로 파일 바이너리를 직접 업로드합니다.
4. 업로드가 끝나면 스토리지 key 또는 파일 메타데이터를 이후 요청에 포함해 사용합니다.

여기서 중요한 점은 백엔드가 파일 자체를 받는 것이 아니라, "업로드 가능한 URL과 key를 발급하는 역할"을 한다는 점입니다.
그래서 업로드가 끝난 뒤 프론트엔드는 보통 파일 자체가 아니라 `key`를 서버에 넘기게 됩니다.

## 이 프로젝트에서 업로드는 어떻게 적용했는가

```ts
function useFileUploadToS3() {
  const mutation = useMutation({
    mutationFn: async ({ file, pathType }) => {
      // 1. 먼저 업로드용 presigned URL을 발급받습니다.
      const presignedUrl = await generatePutUrl({
        query: {
          pathType,
          fileName: file.name,
        },
      });

      if (presignedUrl.error) {
        throw new Error('presigned url 생성 실패');
      }

      // 2. 발급받은 URL로 S3에 파일을 직접 업로드합니다.
      const result = await uploadFileToS3(presignedUrl.data.url, file);

      if (!result.success) {
        throw result.error;
      }

      // 3. 이후 서버에 저장할 수 있도록 key를 반환합니다.
      return presignedUrl.data.key;
    },
  });

  return { mutation };
}
```

```ts
export const uploadFileToS3 = async (presignedUrl: string, file: File) => {
  const response = await s3.put(presignedUrl, {
    body: file,
  });

  if (!response.ok) {
    return failure({
      type: 'CODE',
      origin: 'S3',
      code: response.status,
      message: response.statusText,
    });
  }

  return success({
    status: 200,
    message: 'OK',
    data: {},
  });
};
```

이 구조에서는 “파일 업로드 API”가 파일 본문 전체를 받는 것이 아니라, 업로드 가능한 URL을 발급하고 업로드 후 사용할 `key`를 돌려주는 역할에 더 가깝습니다.
프론트엔드는 그 key를 이후 폼 제출이나 후속 API 요청에 포함해 사용하면 됩니다.
