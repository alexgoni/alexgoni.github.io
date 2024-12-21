---
layout: single
title: 리액트 네이티브 & 웹뷰 Part2
categories: React-Native
excerpt: 유튜브 구간 반복 재생 앱
---

## 1. Static HTML WebView

- 유튜브는 웹용 IFrame Player API만 제공 중
- 웹뷰를 이용해서 앱에서 YouTube IFrame API 사용

참조 문서: [https://developers.google.com/youtube/iframe_api_reference?hl=ko](https://developers.google.com/youtube/iframe_api_reference?hl=ko)

## 2. TextInput

웹에서의 input 태그와 같은 역할

```tsx
<TextInput
  value={url}
  style={styles.input}
  placeholder="클릭하여 링크를 삽입하세요"
  placeholderTextColor="#aeaeb2"
  onChangeText={setUrl}
  inputMode="url"
/>
```

`inputMode`를 통해 키보드의 형태를 바꿀 수 있다.

## 3. Media 옵션

### 3.1 웹뷰 뷰포트 크기 최적화

```html
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
</head>
```

브라우저에게 뷰포트에 대한 메타데이터를 제공하지 않으면, 화면 너비를 데스크톱 화면 크기를 가정으로 렌더한다.

![](/images/2024-12-21-react-native-webview-part2/image01.png)

- `width=device-width`: 뷰포트의 너비를 장치의 화면 너비에 맞게 설정
- `initial-scale=1`: 페이지가 처음 로드될 때의 확대 수준을 1배로 설정

### 3.2 전체 화면 재생 비활성화

```tsx
<WebView source={source} allowsInlineMediaPlayback />
```

`allowsInlineMediaPlayback` prop을 true로 설정해줘야 영상이 실행될 때 자동으로 전체 화면으로 재생되는 것을 막을 수 있다.

### 3.3 자동 재생 활성화

```tsx
<WebView source={source} mediaPlaybackRequiresUserAction={false} />
```

`mediaPlaybackRequiresUserAction`의 기본값은 true, 즉 자동 재생을 막는다.  
이 값을 비활성화함으로서 자동 재생을 활성화할 수 있다.

## 4. 앱과 웹 통신

웹에서 전송하는 메시지를 앱으로 받을 때는 WebView의 `onMessage`를 이용했었다.  
하지만 웹뷰에서 전송하는 메시지의 종류가 여러 개일 때는 구분을 해줄 필요가 있다.

메시지를 보낼 때 메시지의 타입과 함께 데이터를 보내는 방식으로 이 문제를 해결할 수 있다.

```js
function postMessageToRN(type, data) {
  const message = JSON.stringify({ type, data });
  window.ReactNativeWebView.postMessage(message);
}
```

```tsx
<WebView
  source={source}
  onMessage={(e) => {
    const { type, data } = JSON.parse(e.nativeEvent.data);
    if (type === "player-state") setIsPlaying(data === 1);
    if (type === "duration") setDurationInSec(data);
    if (type === "current-time") setCurrentTimeInSec(data);
  }}
/>
```

## 5. PanResponder를 통한 사용자 제스처 처리

**PanResponder**

- React Native에서 터치와 제스처 이벤트를 쉽게 처리할 수 있도록 도와주는 유틸리티
- 사용자가 화면을 터치하고 이동하는 동작을 감지하고 이에 대한 반응을 정의할 수 있음

### 주요 콜백 함수

#### `onStartShouldSetPanResponder`

- 사용자가 화면을 터치했을 때 PanResponder가 활성화될지 여부를 결정
- 반환값이 true일 경우, PanResponder는 해당 터치 이벤트를 처리
- 주로 사용자가 화면을 터치하는 순간에 조건을 체크하여 활성화할지 결정하는 데 사용

#### `onMoveShouldSetPanResponder`

- 사용자가 터치한 상태에서 이동할 때 PanResponder가 활성화될지 여부를 결정
- 반환값이 true인 경우 PanResponder는 해당 이동 이벤트를 처리

#### `onPanResponderGrant`

- PanResponder가 활성화되거 제스처가 시작될 때 호출
- 주로 초기화 작업 수행

#### `onPanResponderMove`

- 사용자가 터치한 상태에서 이동할 때 호출
- gestureState 객체를 통해 터치 이동 정보를 받아 처리
- UI 요소를 드래그할 때 사용

```tsx
const panResponder = useRef(
  PanResponder.create({
    onStartShouldSetPanResponder: () => true,
    onMoveShouldSetPanResponder: () => true,
    onPanResponderGrant: () => {
      webViewRef.current?.injectJavaScript("player.pauseVideo();");
    },
    onPanResponderMove: (e, gestureState) => {
      const newTimeInSec =
        (gestureState.moveX / YT_WIDTH) * durationInSecRef.current;
      seekBarAnimRef.current.setValue(newTimeInSec);
    },
    onPanResponderRelease: (e, gestureState) => {
      const newTimeInSec =
        (gestureState.moveX / YT_WIDTH) * durationInSecRef.current;
      webViewRef.current?.injectJavaScript(
        `player.seekTo(${newTimeInSec}, true);`
      );
      webViewRef.current?.injectJavaScript("player.playVideo();");
    },
  })
);

<View style={styles.seekBarBackground} {...panResponder.current.panHandlers}>
  {/* ... */}
</View>;
```
