---
layout: single
title: 리액트 네이티브 & 웹뷰 Part1
categories: React-Native
excerpt: 패스트캠퍼스 리액트 네이티브 앱 개발의 모든 것 Part1
---

## 앱의 종류

### 1. 네이티브 앱

- 특정 플랫폼(IOS, Android)를 위해 최적화되어 개발된 애플리케이션
- Swift, Kotlin, Java 등으로 작성

**장점**

- 최적화된 코드로 빠른 성능 제공
- 기기의 하드웨어 기능에 완벽하게 접근

**단점**

- 개발 비용
- 버전 유지 관리
- 앱 스토어 심사 과정

### 2. 웹 앱

- 별도의 앱 다운로드 없이 모바일 브라우저를 통해 접근

**장점**

- 어떤 운영 체제에서도 실행할 수 있음
- 앱 스토어 심사 불필요

**단점**

- 기기의 모든 기능을 사용할 수 없음
- URL로 접근 => 접근성이 떨어짐

### 3. 하이브리드 앱

- 네이티브 앱 + 웹 앱
- 웹 기술로 개발하여, 네이티브 앱의 **웹뷰**를 통해 실행

**장점**

- 개발 효율성
- 기기의 기능에 접근 가능
- 앱 스토어를 통하지 않고도 앱의 일부 웹 콘텐츠를 수정하는 것으로, 사용자에게 신속한 변경과 개선을 제공 가능

**단점**

- 웹뷰를 통한 웹 콘텐츠 렌더링은 순수 네이티브 앱에 비해 성능이 떨어질 수 있음

## 웹뷰

![](/images/2024-12-17-react-native-webview-part1/image01.png)

- 웹뷰는 네이티브 애플리케이션 내에서 웹 페이지를 표시하기 위해 사용되는 **컴포넌트**
- 웹뷰는 기본적으로 내장된 브라우저 엔진(Android의 Chromium, IOS의 WebKit)을 사용하여 웹 콘텐츠를 로드하고 표시
- 웹 뷰는 앱 내에서 웹 콘텐츠를 렌더링하지만 단순한 콘텐츠 표시 기능을 넘어서 앱의 네이티브 기능과 웹 콘텐츠 간의 데이터 교환 및 상호 작용이 가능

## React Native

- React와 JS를 이용해서 네이티브 앱 개발
- 하나의 언어로 동시에 개발이 가능한 **크로스 플랫폼 프레임워크**

### React Native 동작 방식

![](/images/2024-12-17-react-native-webview-part1/image02.png)

1. JS 코드 작성
2. 하나의 번들로 컴파일
3. JS 스레드에서 실행
4. 네이티브 기능을 활용하는 코드 => BRIDGE를 통해 통신
5. BRIDGE를 통해 각 플랫폼의 네이티브 코드를 실행

## Expo

- React Native 프로젝트를 더 쉽게 시작하고 개발할 수 있도록 돕는 프레임워크

**장점**

- 쉬운 시작
- Expo Go 앱: 사용중인 디바이스에서 Expo Go라는 모바일 앱을 통해 개발 중인 프로젝트를 실시간으로 테스트 가능

**단점**

- 모든 네이티브 기능에 접근 불가
- 네이티브 코드에 접근하기 어렵기 때문에, 성능 최적화에 한계가 있을 수 있음

![](/images/2024-12-17-react-native-webview-part1/image03.png)

### expo-router

- expo에서 라우팅을 위해 사용하는 라이브러리
- 파일 기반 라우팅 제공 (Next.js의 앱 디렉토리 폴더 구조와 매우 유사)
- 다양한 네비게이션 패턴 지원 (Stack, Tab, Drawer)

```
📦app
 ┣ 📂(tabs)
 ┃ ┣ 📜index.tsx             // "/"에 해당하는 screen
 ┃ ┣ 📜shopping.tsx          // "/shopping"에 해당하는 screen
 ┃ ┗ 📜_layout.tsx           // "/", "/shopping"에 해당하는 공통 레이아웃
 ┣ 📜browser.tsx             // "/browser"에 해당하는 screen
 ┣ 📜login.tsx               // "/login"에 해당하는 screen
 ┗ 📜_layout.tsx             // 전체 레이아웃
```

## Part1 - Project

### 1. 네비게이션 패턴

#### Stack

![](/images/2024-12-17-react-native-webview-part1/image04.png)

```tsx
// _layout.tsx

// name에 매칭되는 파일로 라우팅된다.
<Stack>
  <Stack.Screen name="(tabs)" />
  <Stack.Screen name="browser" />
  <Stack.Screen name="login" />
</Stack>
```

#### Tab

```tsx
// (tabs)/_layout.tsx

<Tabs>
  <Tabs.Screen name="index" />
  <Tabs.Screen name="shopping" />
</Tabs>
```

### 2. 웹뷰 띄우기

네이티브 앱에 웹 콘텐츠를 띄울 때는 `WebView` 컴포넌트를 사용한다.

이 웹뷰 컴포넌트를 사용하는 방법에는 두 가지가 있다.

```tsx
// 1. uri 사용
<WebView source={{ uri: "https://m.naver.com/" }} />
```

```tsx
// 2. html 사용
<WebView
  originWhitelist={["*"]} // 탐색을 허용할 url을 확인
  source={{ html: "<h1><center>Hello world</center></h1>" }}
/>
```

### 3. SafeAreaView

SafeArea는 노치, 상태바 등으로 가려지지 않는 화면 영역을 의미한다.

IOS는 `SafeAreaView` 컴포넌트 안에 children을 넣었을 때 문제없이 보여지나,  
Android는 상태바에 여전히 가려지는 이슈가 존재한다.

이를 해결하기 위해 Android일 때만 SafeAreaView의 paddingTop을 상태바만큼의 값을 준다.

```tsx
const styles = StyleSheet.create({
  safearea: {
    paddingTop: Platform.OS === "android" ? StatusBar.currentHeight : 0,
    flex: 1,
  },
});

export default function HomeScreen() {
  return <SafeAreaView style={styles.safearea}>{/* ... */}</SafeAreaView>;
}
```

### 4. 웹뷰에서 페이지 이동 처리하기

#### 조건

- 홈 Stack에서 `https://m.naver.com/`로 시작하는 URL은 해당 Stack에서 보여줍니다.
- `https://m.naver.com/`로 시작하지 않는 URL은 browser Stack으로 이동하여 보여줍니다.

#### Problem Solving

WebView의 `onShouldStartLoadWithRequest` Prop을 이용한다.

- WebView를 load할 때 실행되는 함수
- true를 반환하면 load를 계속 진행하고, false를 반환하면 load 중단

```tsx
<WebView
  onShouldStartLoadWithRequest={(request) => {
    if (
      request.url.startsWith("https://m.naver.com/") ||
      request.mainDocumentURL?.startsWith("https://m.naver.com/") // mainDocumentURL: IOS에서의 URL
    ) {
      return true;
    }

    if (request.url !== null && request.url.startsWith("https://")) {
      // browser Stack으로 이동
      router.navigate({
        pathname: "browser",
        params: { initialUrl: request.url },
      });
      return false;
    }

    return true;
  }}
/>
```

### 5. 로딩바 구현

1. loadingBackground View 내부에 loadingProgress View를 생성한다.  
   이때 loadingProgress View는 `Animated.View`를 사용한다.
2. progress 진행 정도를 저장하는 변수를 만든다.  
   이때 `new Animated.Value()`로 선언한다.
3. WebView의 `onLoadProgress` prop을 사용해 progress를 업데이트한다.
4. loadingProgress의 width를 progress와 일치 시킨다.

```tsx
export default function BrowserScreen() {
  const progressAnim = useRef(new Animated.Value(0)).current;

  return (
    <>
      <View style={styles.loadingBarBackground}>
        <Animated.View
          style={[
            styles.loadingBar,
            {
              width: progressAnim.interpolate({
                inputRange: [0, 1],
                outputRange: ["0%", "100%"],
              }),
            },
          ]}
        />
      </View>

      <WebView
        onLoadProgress={(e) => {
          progressAnim.setValue(e.nativeEvent.progress);
        }}
      />
    </>
  );
}

const styles = StyleSheet.create({
  loadingBarBackground: {
    height: 3,
    backgroundColor: "white",
  },
  loadingBar: {
    height: "100%",
    backgroundColor: "green",
  },
});
```

### 6. Pull to Refresh

> 스크린을 위로 당겼을 때 refresh 되는 기능 구현하기

웹뷰 콘텐츠 내에서 스크롤은 가능하지만, 앱 내에서의 스크롤은 원래 지원하지 않는다.  
앱 내에서 스크롤을 가능케하려면 `<ScrollView />` 컴포넌트를 사용해야 한다.

![alt text](/images/2024-12-17-react-native-webview-part1/image05.png)

`<ScrollView />` 컴포넌트에는 `refreshControl` 이라는 prop이 있다.  
해당 prop에 `<RefreshControl />` 컴포넌트를 내려줌으로서 refresh 상태를 표시하고, refresh 중일 때의 실행할 함수를 지정할 수 있다.

```tsx
export default function ShoppingScreen() {
  const [isRefreshing, setIsRefreshing] = useState(false);
  const webViewRef = useRef<WebView | null>(null);

  const onRefresh = useCallback(() => {
    if (!webViewRef.current) return;
    setIsRefreshing(true);
    webViewRef.current.reload();
  }, []);

  return (
    <SafeAreaView style={styles.safearea}>
      <ScrollView
        contentContainerStyle={{ flex: 1 }}
        refreshControl={
          <RefreshControl refreshing={isRefreshing} onRefresh={onRefresh} />
        }
      >
        <WebView
          ref={webVieRef}
          source={{ uri: SHOPPING_HOME_URL }}
          onLoadEnd={() => {
            setIsRefreshing(false);
          }}
          renderLoading={() => <></>} // 빈 화면으로 발생하는 깜빡임 문제 해결
          startInLoadingState={true} // 빈 화면으로 발생하는 깜빡임 문제 해결
        />
      </ScrollView>
    </SafeAreaView>
  );
}
```

### 7. 웹뷰에서의 인증

expo에서는 쿠키를 활용하는 모듈을 사용할 수 없는 이슈  
=> 웹의 document 객체의 cookie 프로퍼티를 활용하는 것으로 해결

**웹과 웹뷰 소통하기**
![](/images/2024-12-17-react-native-webview-part1/image06.png)

#### 로그인

1. 웹뷰 내에서 로그인
2. 네이버는 로그인 시 네이버 홈 화면으로 이동
   이를 이용해 이동하는 url을 확인하고, `router.back()` 호출
   앱 내에 존재하는 모든 웹뷰를 `reload`

   ```tsx
   // login.tsx

   <WebView
     source={{ uri: LOGIN_URL }}
     onNavigationStateChange={(e) => {
       if (e.url === "https://m.naver.com/") {
         webViewRefs.current.forEach((webView) => {
           webView.reload();
         });
         router.back();
       }
     }}
   />
   ```

3. WebView 로드가 완료되면 쿠키를 읽는 JS 삽입

   ```jsx
   // hooks/useLogin.ts

   const loadLoggedIn = useCallback(() => {
     webViewRefs.current.forEach((webView) => {
       webView.injectJavaScript(`
           (function() {
             window.ReactNativeWebView.postMessage(document.cookie);
           })();
         `);
     });
   }, []);

   const onMessage = useCallback((e: WebViewMessageEvent) => {
     const cookieString = e.nativeEvent.data;
     // 네이버에서는 로그인 성공 시 쿠키에 "NID_SES"가 포함됨
     setIsLoggedIn(cookieString.includes("NID_SES"));
   }, []);
   ```

   ```
   // index.tsx

   <WebView
     onLoadEnd={() => {
       loadLoggedIn();
     }}
     onMessage={onMessage}
   />;
   ```

#### 로그아웃

1. 앱 내 모든 웹뷰에 즉시 소멸되는 쿠키 삽입
2. `isLoggedIn` 상태를 false로 변경
3. 앱 내 모든 웹뷰 reload

```tsx
const logout = useCallback(() => {
  webViewRefs.current.forEach((webView) => {
    webView.injectJavaScript(`
        (function() {
          document.cookie = 'NID_SES=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/; domain=.naver.com';
          window.ReactNativeWebView.postMessage(document.cookie);
        })();      
      `);
  });

  setIsLoggedIn(false);

  if (webViewRefs) {
    webViewRefs.current.forEach((webView) => {
      webView.reload();
    });
  }
}, []);
```

### 8. 웹뷰 고도화

#### 핀치 줌/아웃 비활성화

앱에서는 줌인, 줌아웃이 일반적으로 허용되지 않는다.

- 웹페이지 내에서 줌인, 줌아웃을 비활성화하거나
- 웹페이지를 수정할 수 없다면, `injectedJavaScript`를 이용한다.

```js
const DISABLE_PINCH_ZOOM = `(function() {
  const meta = document.createElement('meta');
  meta.setAttribute('content', 'width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no');
  meta.setAttribute('name', 'viewport');
  document.getElementsByTagName('head')[0].appendChild(meta);
})();`;
```

#### 링크 롱프레스 프리뷰 비활성화

```tsx
<WebView allowsLinkPreview={false} />
```

#### 텍스트 롱프레스 액션(드래그) 비활성화

```tsx
const DISABLE_TEXT_SELECT = `(function() {
  document.body.style['user-select'] = 'none';
  document.body.style['-webkit-user-select'] = 'none';
})();`;
```

#### 안드로이드 백버튼과 웹뷰 연결

- 안드로이드 기본 백버튼 동작은 앱 스크린 네비게이션
- 웹뷰 페이지 네비게이션이 되도록 설정 필요
- @react-native-community/hooks의 useBackHandler 사용

```ts
useBackHandler(() => {
  if (webViewRef.current && canGoBack) {
    webViewRef.current?.goBack();
    return true; // 아무 일 x
  }

  // 기본 안드로이드 핸들러: 앱 종료
  return false;
});
```

#### expo 앱 아이콘, 이름 설정

app.json의 name과 icon 변경

```json
{
  "expo": {
    "name": "Naver",
    "icon": "./assets/icon.png"
    // ...
  }
}
```

## 회고

- iOS와 Android를 동시에 고려해야 하는 어려움: 각 플랫폼의 특성을 신경 쓰며 개발해야 하기 때문에 꼼꼼함이 필수임.
- Expo Go의 한계: 실시간으로 앱을 테스트할 수 있는 Expo Go는 항상 최신 버전의 Expo만 지원하기 때문에, 해당 버전에 에러가 있어도 다운그레이드가 불가능한 어려움이 있음.
  → 다음 실습에서는 react-native-cli를 사용해볼 예정임.
- 모듈과 컴포넌트 활용: 기본적으로 잘 만들어진 컴포넌트나 모듈이 많아 간단한 기능은 prop을 내려주거나 모듈을 사용해 쉽게 구현했음.
  하지만 프로젝트 규모가 커지고 기능이 복잡해져도 확장성과 유지 보수가 괜찮을지는 의문이 남음.
