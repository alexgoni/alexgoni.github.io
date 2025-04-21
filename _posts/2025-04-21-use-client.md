---
layout: single
title: "use client"
categories: Next.js
excerpt: 클라이언트 컴포넌트 바로알기
---

| 렌더링: react 코드를 html로 바꾸는 작업

Next.js는 기본적으로 모든 컴포넌트를 서버에서 pre-render한다.  
이는 React와 다른 점인데, React는 브라우저에서 처음부터 렌더링을 시작하지만, Next.js는 초기 HTML을 서버에서 먼저 만들어서 사용자에게 전달한다.

그런데 "use client" 지시어가 붙은 클라이언트 컴포넌트조차도 처음엔 서버에서 HTML로 렌더링된다.  
하지만 이 컴포넌트는 클라이언트에서도 렌더되는 컴포넌트로 hydration 대상이 된다.  
(hydration: 단순 HTML을 React App으로 초기화하는 작업)

ex) /page ---> boring HTML ---> :) ---> init(boring HTML) ---> React App :)

하지만 모든 컴포넌트가 hydration을 필요로 하지는 않는다.  
단순히 보여주기만 하고 사용자와 상호작용하지 않는 컴포넌트는 HTML만 있으면 충분하며, **이 경우 클라이언트에서 JS를 다운로드할 필요도 없다.**

즉, **사용자는 use client가 붙은 components의 JS 코드만 다운받는다.**  
이는 불필요한 JS를 줄이고, 페이지 로딩 속도를 개선하는 데 중요한 역할을 한다.

기존 Page Router에서는 모든 컴포넌트가 hydration 대상이 되었고, 모든 JS 파일이 클라이언트 번들에 포함되었다면,  
App Router에서는 "use client" 지시어를 통해 hydration 대상을 선택적으로 지정할 수 있다.

<br />

🤔 그렇다면 모든 파일에 "use client"를 붙여야 하나요?

클라이언트 컴포넌트의 하위 컴포넌트들은 "use client" 지시어를 안 써도 된다.  
"use client"는 해당 파일을 클라이언트 컴포넌트로 만들겠다는 명시적인 신호이며, **그 안에서 import한 모든 컴포넌트들은 클라이언트 번들에 포함된다.**

즉, 하위의 컴포넌트들은 실제로 사용자에게 JS 파일이 전달되게 되는 것이다.

만약 "use client" 없이 useState, useEffect 등을 썼는데 에러가 나지 않았다면,  
그 이유는 해당 컴포넌트가 클라이언트 컴포넌트 하위에 있기 때문이다.

<br />

기본적으로 데이터는 상위에서 하위로 흐른다는 React의 법칙에 따라  
서버 컴포넌트가 클라이언트 컴포넌트를 렌더링하는 것은 괜찮다.

그러나 클라이언트 컴포넌트가 서버 컴포넌트를 렌더링하는 것은 허용되지 않는다.  
이는 브라우저에서 서버 코드를 실행할 수 없기 때문이다.

다만, `children`을 통해 클라이언트 컴포넌트 하위에 서버 컴포넌트를 가질 수는 있다.

```tsx
// This pattern works:
// You can pass a Server Component as a child or prop of a
// Client Component.
import ClientComponent from "./client-component";
import ServerComponent from "./server-component";

// Pages in Next.js are Server Components by default
export default function Page() {
  return (
    <ClientComponent>
      <ServerComponent />
    </ClientComponent>
  );
}
```

이처럼 `<ClientComponent>`는 children에 대해 구체적인 내용을 알지 못하며, 그저 자리를 비워둔 채로 렌더링을 넘긴다.  
즉, `<ServerComponent>`는 `<ClientComponent>` 외부에서 처리되어 삽입되며, 클라이언트 컴포넌트는 이를 직접 실행하거나 관여하지 않는다.
