---
layout: single
title: "3D 시뮬레이션 구현 회고"
categories: 3D
excerpt: 내가 가진 기술을 활용해 회사 요구사항을 구현하며 배운 점들
---

한 달여간 맡았던 3D 시뮬레이션 기능이 오늘부로 라이브 배포되었다.  
많은 것을 배우고 의미 있었던 프로젝트였기에 회고를 남겨본다.

## 1. 요구사항 분석

요구사항: 선택한 필지에 대해 건폐율, 용적률, 일조권 등을 고려해, 건축 가능한 모델을 사용자에게 시각적으로 보여주는 기능.

내 역할은 여러 조건을 기반으로 계산된 좌표 데이터를 받아, 해당 데이터를 3D 건축 모델로 시각화하여 웹에 표출하는 것이었다.  
초기 계획은 위도/경도 좌표계를 기준으로 받은 데이터를 3D 공간에 적절히 변환하고, 변환된 좌표를 기반으로 Scene을 구성하는 것이었다.  
프로젝트 초반에는 백엔드보다 프론트엔드 작업이 먼저 진행되었기 때문에, 실제 API가 준비되기 전까지 mock 데이터를 활용한 데모 앱을 먼저 구현하여 전체 흐름을 검증했다.

## 2. 3D Demo

먼저 기본적인 3D Scene을 구성하였다.  
필수적으로 들어가야 하는 요소가 조명, 바닥, 건물이다. 또 3D Scene 안에는 포함되지는 않지만 방향을 알려주는 나침반 요소가 필요하다.

### 2.1 조명

조명은 주로 사용하는 조합인
주로 사용하는 조합인 자연광 `ambientLight`과 직사광 `directionalLight` 두 개를 혼합해서 사용하였다.

```tsx
<ambientLight intensity={1} color="white" />
<directionalLight
  intensity={2}
  position={[5, 10, 5]}
  castShadow
  shadow-mapSize-width={1024}
  shadow-mapSize-height={1024}
  shadow-camera-left={-50}
  shadow-camera-right={50}
  shadow-camera-top={50}
  shadow-camera-bottom={-50}
/>
```

그러나 직사광을 하나만 썼을 때 각 층의 교차되는 부분이 선명하게 드러나지 않는 문제점이 있었다.  
이 경우 위에서 아래로 향하는 직사광을 추가했을 때 그림자로 인해 좀 더 선명히 드러나게 할 수 있었다.

```tsx
// 추가
<directionalLight intensity={0.3} position={[0, 10, 0]} />
```

<div style="display: flex; gap: 1rem; align-items: flex-start;">
  <figure style="flex: 1; text-align: center;">
    <img src="/images/2025-03-24-3D-simulation/image2.png" alt="image2" style="width: 100%; height: 300px; object-fit: cover;">
    <figcaption>직사광이 한 개인 경우</figcaption>
  </figure>
  <figure style="flex: 1; text-align: center;">
    <img src="/images/2025-03-24-3D-simulation/image1.png" alt="image1" style="width: 100%; height: 300px; object-fit: cover;">
    <figcaption>위에서 아래 방향의 직사광을 추가한 경우</figcaption>
  </figure>
</div>

<br />

그림자는 그림자를 생성하는 조명에서 설정한다.  
이때 사용하는 카메라는 Orthographic(직교) 카메라이며,  
`shadow-camera-left/right/top/bottom`을 통해 이 카메라의 투영 범위를 설정할 수 있다.

![](/images/2025-03-24-3D-simulation/image4.png)

`shadow-camera-left/right/top/bottom`를 통해 이를 설정할 수 있는데,  
이 값들이 너무 작으면, 아래 이미지처럼 건물이 커질 때 그림자가 일부 잘리는 현상이 발생할 수 있다.  
반대로 너무 크게 설정하면 그림자 품질이 저하되는 단점이 생긴다.  
50 정도의 범위로 설정했을 때 모든 건물 크기를 안정적으로 커버할 수 있었다.

<figure style="text-align: center;">
  <img src="/images/2025-03-24-3D-simulation/image3.png" alt="shadow issue" style="width: 100%; max-width: 600px;">
  <figcaption>투영 범위가 좁을 때 발생하는 그림자 끊김 현상</figcaption>
</figure>

### 2.2 건물

건물 모델을 어떻게 구현할지에 대해 많은 고민을 했다.  
우선, 각 필지는 다각형 형태의 프리즘(윗면과 아랫면이 서로 평행하고 합동인 다각형으로 이루어진 입체 도형)으로 표현되어야 한다.  
또한, 각 층의 형태가 다를 수 있으므로, 각 층마다 개별적인 프리즘을 만들어 이를 쌓아올리는 방식으로 구현해야 한다.

삼각기둥, 사각기둥 형태의 프리즘은 Three.js의 `ExtrudeGeometry`를 활용했다.  
`ExtrudeGeometry`는 Three.js에서 2D 형태의 도형을 기반으로 3D 입체 모양을 만드는 데 사용하는 클래스다. 쉽게 말해, 2D 도형을 위로 쭉 밀어올려 입체화하는 기능을 제공한다.

```tsx
function BuildingFloor({ coord, height, floorPosition }) {
  // Three.js Shape로 다각형 그리기
  const shape = useMemo(() => {
    const s = new THREE.Shape();
    s.moveTo(coord[0][0], coord[0][1]);
    for (let i = 1; i < coord.length; i++) {
      s.lineTo(coord[i][0], coord[i][1]);
    }
    s.closePath();
    return s;
  }, [coord]);

  // depth로 얼만큼 키울건지 설정
  const extrudeSettings = useMemo(
    () => ({
      depth: height,
      bevelEnabled: false,
    }),
    [height]
  );

  return (
    <mesh>
      <extrudeGeometry args={[shape, extrudeSettings]} />
      <meshStandardMaterial color="#67D1F7" />
    </mesh>
  );
}
```

Shape로 그린 도형은 기본적으로 XY 평면에 생성되며, 높이를 설정하면 Z축 방향으로 입체화된다.  
하지만 내가 원하는 방향은 XZ 평면에서 시작해 Y축 방향으로 높이가 쌓이는 구조이므로, 회전을 통해 이를 맞춰줘야 한다.

![](/images/2025-03-24-3D-simulation/image5.png)

```tsx
<mesh
  rotation={[-Math.PI / 2, 0, 0]}
  position={[0, floorPosition, 0]}
  castShadow
>
  <extrudeGeometry args={[shape, extrudeSettings]} />
  <meshStandardMaterial color="#67D1F7" />
</mesh>
```

또한, 각 층의 바닥 위치(floorPosition)를 상위 컴포넌트에서 prop으로 받아 조정함으로써 여러 개의 층을 쌓아 하나의 건물 모델을 구성할 수 있다.

<br />

건물의 테두리는 mesh 내부에서 직접 설정하는 대신, 별도의 Object로 추가하여 표현할 수 있다.

처음에는 `edgesGeometry`를 사용해 테두리를 구현했다.  
`edgesGeometry`는 Three.js에서 3D 객체의 모서리만 추출해 선 형태로 시각화할 수 있도록 도와주는 Geometry 클래스다.

하지만 `edgesGeometry`로 렌더링할 경우 선의 두께가 항상 1로 고정되어 있다는 한계가 있었다.  
모바일 화면이나 작은 3D 캔버스에서는 선 두께가 상대적으로 더 굵어 보이는 문제가 발생했기 때문에, 다른 방법을 고려해야 했다.

이를 해결하기 위해 Three.js의 `LineSegments2`를 활용했다.
이를 사용하면 선의 두께를 정밀하게 조절할 수 있으며, 두께를 0.5로 설정해 더 얇은 선을 표현할 수 있었다.  
이때 선을 그리는 객체는 `<primitive>`를 사용해 직접 Three.js 객체를 넣는 방식으로 구현한다.

`<primitive>`: Three.js에서 직접 생성한 객체를 React Three Fiber 컴포넌트로 삽입할 때 사용하는 특수 컴포넌트

```tsx
import { LineMaterial } from "three/examples/jsm/lines/LineMaterial";
import { LineSegments2 } from "three/examples/jsm/lines/LineSegments2";
import { LineSegmentsGeometry } from "three/examples/jsm/lines/LineSegmentsGeometry";

// ...

const extrudeGeo = useMemo(
  () => new THREE.ExtrudeGeometry(shape, extrudeSettings),
  [shape, extrudeSettings]
);
const edgesGeo = useMemo(
  () => new THREE.EdgesGeometry(extrudeGeo),
  [extrudeGeo]
);
const lineMat = useMemo(
  () =>
    new LineMaterial({
      color: "black",
      linewidth: 0.5,
    }),
  []
);

return (
  <>
    {/* 테두리 */}
    <primitive
      object={
        new LineSegments2(
          new LineSegmentsGeometry().fromEdgesGeometry(edgesGeo),
          lineMat
        )
      }
      rotation={[-Math.PI / 2, 0, 0]}
      position={[0, floorPosition, 0]}
    />
  </>
);
```

### 2.3 나침반

나침반 UI는 3D Scene에 존재하진 않지만, 3D Scene의 정보를 활용해 그려야하는 UI다.

`useThree` 훅을 통해 카메라 정보를 받아오고, `useFrame`을 사용해 매 프레임마다 카메라의 방향을 계산한 뒤, 그 값을 상태로 저장해 나침반 UI에 전달하는 방식으로 구현했다.

```tsx
const { camera } = useThree();

// 매 프레임마다 카메라 방향을 기준으로 나침반 회전값 업데이트
useFrame(() => {
  const dir = new THREE.Vector3();

  // 카메라가 바라보는 방향 벡터 구하기
  camera.getWorldDirection(dir);

  // 방향 벡터를 기준으로 XZ 평면에서의 각도 계산 (라디안 → 도로 변환)
  const rotation = THREE.MathUtils.radToDeg(Math.atan2(dir.x, dir.z)) - 180;
  setCompassRotation(rotation);
});
```

![](/images/2025-03-24-3D-simulation/image6.gif)

### 2.4 좌표 변환

지도상에서 사용하는 위도, 경도 좌표를 Three.js의 XZ 평면으로 옮기는 과정이 필요하다.  
해당 과정은 다음과 같은 단계로 진행된다:

1. 모든 위도·경도 좌표를 XZ 평면 좌표로 변환
2. 건물 1층 좌표의 무게중심점 계산
3. 모든 건물 좌표를 중심점 기준 상대 좌표로 변환

<br />

지도에서 사용하는 위도·경도(WGS84)를 3D 공간에서 사용할 수 있는 평면 좌표(Web Mercator)로 변환하기 위해 proj4 라이브러리를 사용했다.  
WGS84는 GPS 등에서 사용하는 지리 좌표계이고, Web Mercator는 웹 지도에서 주로 사용되는 평면 좌표계다.  
이 변환을 통해 실세계 단위(미터 기준)의 평면 좌표를 얻을 수 있다.

```ts
// 좌표계 변환
export const lngLatToXZ = (coords: [number, number]): [number, number] => {
  const wgs84 = "EPSG:4326";
  const webMercator = "EPSG:3857";
  const [x, z] = proj4(wgs84, webMercator, coords);

  return [x, z];
};
```

<br />

다음으로는 건물의 중심점을 구하는 과정이다.  
처음에는 모든 꼭짓점 좌표의 평균값을 구해 중심점으로 사용했으나,  
꼭짓점이 한쪽에 몰린 경우 평균값이 실제 중심에서 벗어나는 문제가 있었다.

이 문제를 해결하기 위해 **무게중심**을 구하는 방법을 사용했고,  
이를 위해 GIS 도구인 turf.js 라이브러리를 활용했다.  
구해진 중심 좌표는 3D Scene에서 기준점 (0, 0, 0)에 해당한다.

```ts
// 중심 좌표 계산
export const getCenter = (coords: [number, number][]) => {
  const polygon = turf.polygon([coords]);
  const center = turf.centerOfMass(polygon);
  return center.geometry.coordinates;
};
```

<br />

마지막으로, 모든 좌표를 중심점 기준 상대 좌표로 변환한다.  
변환된 평면 좌표는 실제 미터 단위로 되어 있어, Scene에 렌더링했을 때 건물이 과도하게 커 보이는 문제가 있었다.  
이를 보정하기 위해 SCALE_FACTOR를 적용하여 전체 좌표 크기를 줄여주었다.

```ts
// 축소 비율
export const SCALE_FACTOR = 4;

// 상대 좌표 계산
export const getRelativeCoords = (
  coords: [number, number][],
  center: number[]
): [number, number][] => {
  return coords.map(([lng, lat]) => [
    (lng - center[0]) / SCALE_FACTOR,
    (lat - center[1]) / SCALE_FACTOR,
  ]);
};
```

## 3. 실제 구현

### 3.1 API 응답 처리

백엔드 작업이 완료되었다.  
이제 주요로 할 일은 api 응답을 처리해서 이전에 구현된 데모와 호환될 수 있게 api 응답을 잘 처리하는 일이다.

제일 고민한 부분은 슬라이더 UI로 건폐율을 조정하는 것인데, 요구사항이 이 건폐율에 따라 건축물 모델도 변경이 되어야한다는 것이다.

![](/images/2025-03-24-3D-simulation/image7.png)

연속적인 UI에 의해 **서버 요청을 여러 번 보내게 되는데** 이를 해결하기 위해 두 가지 방법을 사용했다.

첫 번째로 슬라이더 조작이 끝난 시점에만 요청을 보내도록 하였다.  
슬라이더 UI는 MUI를 사용하고 있는데, 이때 `onChange`가 아닌 `onChangeCommitted`를 사용해서 조작이 끝났을 때만 API 요청이 가도록 처리했다.  
표시용 상태(ratio)와 요청에 사용할 상태(debouncedRatio)를 분리해서 관리했다.

```tsx
const [ratio, setRatio] = useState(20);
const [debouncedRatio, setDebouncedRatio] = useState(ratio);

// ...

<StyledSlider
  onChange={(_, newValue) => setRatio(newValue as number)}
  onChangeCommitted={(_, newValue) => {
    setDebouncedRatio(newValue as number);
  }}
/>;
```

<br />

두 번째로 `AbortSignal`를 활용해 이전 요청을 취소하도록 하였다.  
응답을 여러 번 보냈을 때 이전 요청에 대해 응답을 기다리고 있어, 가장 마지막 응답이 지연되는 문제가 발생했다.  
이를 해결하기 위해 이전 요청이 아직 처리 중이라면 해당 요청을 abort하고, 최신 요청만 처리하도록 쿼리 훅을 구성했다.

```ts
export const useGetLandUseBuild = (
  { pnu, buildRatio }: { pnu: string; buildRatio: number },
  options?: UseQueryOptions<LandUseBuildDto>
) => {
  const controllerRef = useRef<AbortController | null>(null);

  const getLandUseBuild = async () => {
    if (controllerRef.current) {
      controllerRef.current.abort();
    }
    controllerRef.current = new AbortController(); // 이전 요청 중단

    const body = JSON.stringify({ pnu, buildRatio });
    const res = await xapi.post(`${axiosPath}/build`, body, {
      timeout: 30_000,
      signal: controllerRef.current.signal,
    });
    return res?.data;
  };

  return useQuery<LandUseBuildDto>(
    ["landUseBuild", { pnu, buildRatio }],
    getLandUseBuild,
    options
  );
};
```

### 3.2 웹뷰 통신

모바일 앱에서 보여주는 웹뷰도 구현해야 했다.  
처음에는 뷰만 보여주고, 따로 통신할 필요가 없을 것이라고 생각했다.

그러나 모바일 환경에서는 따로 처리해야 할 사항들이 있었다.

첫 번째로, 3D Scene에서의 컨트롤과 화면 스크롤 이벤트가 중첩되어 의도치 않게 동작했다. 화면을 내릴 때 3D 화면이 움직이며 동시에 화면도 같이 스크롤되었고, 이로 인해 사용자 경험이 좋지 않았다.  
이와 같은 현상은 슬라이더 조작 시에도 발생했다.

따라서 캔버스를 터치했을 때, 슬라이더를 조작할 때는 화면 스크롤을 막는 방향으로 네이티브 팀과 협의했다.  
웹에서 할 일은 해당 이벤트 발생 시 boolean 값을 네이티브로 전달하는 것이었다.
아래와 같이 구현했다.

```tsx
// 캔버스 터치 시
<Canvas
  shadows
  onPointerDown={() => {
    transferToNative("true", "disableNativeScroll");
  }}
  onPointerUp={() => {
    transferToNative("false", "disableNativeScroll");
  }}
></Canvas>

// 슬라이더 조작 시
<StyledSlider
  onPointerDown={() => {
    transferToNative("true", "disableNativeScroll");
  }}
  onChangeCommitted={(_, newValue) => {
    transferToNative("false", "disableNativeScroll");
    setDebouncedRatio(newValue as number);
  }}
/>
```

두 번째로, UI 높이가 API 응답에 따라 변하기 때문에 웹뷰의 높이를 네이티브에게 알려줘야 했다.  
만약 모든 페이지가 웹뷰였다면 크게 상관없었겠지만, 3D 시뮬레이션에서는 네이티브 안에 특정 부분만 웹뷰가 존재하고, 네이티브 개발자 분들이 웹뷰를 위한 공간을 미리 지정해 놓았기 때문에, 높이가 변할 때마다 이를 알려줄 필요가 있었다.

```ts
// 건축 불가 필지인 경우 웹뷰 높이 변경 필요
useEffect(() => {
  if ((data && data.result !== 200) || isError) {
    transferToNative(CONSTRUCTION_BAN_HEIGHT, "updateViewHeight");
  }
}, [data, isError]);

// 일조권 적용 대상 여부에 따라 웹뷰 높이 변경 필요
useEffect(() => {
  if (data && data.result === 200) {
    if (data.ancientLights === true) {
      transferToNative(SUN_RIGHTS_VIEW_HEIGHT, "updateViewHeight");
    } else {
      transferToNative(DEFAULT_VIEW_HEIGHT, "updateViewHeight");
    }
  }
}, [data]);
```

<br />

구현 완료된 화면 모습은 다음과 같다.

![](/images/2025-03-24-3D-simulation/image8.gif)

## 4. 트러블 슈팅

### 4.1 앱 크래쉬

프로젝트 초반에 발견한 문제로, 앱에서 3D Scene을 많이 회전시킬 때 앱이 크래시 나는 현상이 있었다.  
추가적으로 테스트했을 때 앱보다는 오랜 시간 회전을 해야했지만, 결국에는 PC 웹도 터져버리는 현상이 발생했다.

문제의 원인은 나침반과 컴포넌트 구조에 있었다.

```tsx
export default function App() {
  const [compassRotation, setCompassRotation] = useState(0);

  return (
    <>
      {/* 3D Scene */}
      <ThreeComponent setCompassRotation={setCompassRotation} />
      {/* 나침반 */}
      <Compass compassRotation={compassRotation} />
    </>
  );
}
```

나침반 UI는 3D Scene 바깥에 위치하는 UI였지만, 방향에 대한 상태가 필요하여 가장 상위 컴포넌트에서 방향 상태를 정의하고 이를 prop으로 내려주고 있었다.

그런데 `ThreeComponent` 안에서 `useFrame`을 사용하여 매 프레임마다 `setCompassRotation`을 호출하고 있었기 때문에, setState가 지속적으로 호출되어 가장 상위 컴포넌트가 매 프레임마다 리렌더링되었다.

결국, `ThreeComponent` 내부의 3D Scene을 그리는 무거운 작업이 매번 호출되어 앱이 크래시되었다.

이 문제를 해결하기 위해 리렌더링을 최소화해야 했고, 전역 상태 관리 라이브러리를 사용하는 방법을 택했다.

방향을 계산하는 로직을 3D 시뮬레이션 컴포넌트 중 가장 하위에 위치시켰고, 방향 상태를 Recoil을 통해 관리하여 리렌더링을 최소화할 수 있었다.

```tsx
export function Compass() {
  const compassRotation = useRecoilValue(compassRotationState);

  return (
    <CompassWrapper
      style={{
        transform: `rotate(${compassRotation}deg)`,
      }}
    >
      <CompassNeedle />
      <CompassLabel>N</CompassLabel>
    </CompassWrapper>
  );
}
```

해당 방식으로 변경했을 때, 앱과 웹 모두에서 크래시 문제를 해결할 수 있었다.

### 4.2 타입 충돌

빌드 시 MUI와 R3F의 타입이 충돌하는 문제가 발생했다.  
이는 꽤나 오래전부터 있던 이슈로 보이는데 아직 해결이 되지 못한 것 같다.  
이슈: https://github.com/pmndrs/react-three-fiber/discussions/1752

같은 Box에 대해 타입 추론이 엉켜 이런 문제가 발생하는 것으로 보이는데  
일단 현재 3D 코드에서는 Box를 사용하지 않으므로, MUI의 타입을 우선하기 위해 다음과 같이 정의하였다.

```tsx
declare global {
  namespace JSX {
    interface IntrinsicElements {
      // R3F의 box와 MUI의 Box 타입이 충돌하는 이슈가 있습니다.
      // MUI의 타입을 우선하기 위해 추가하였습니다.
      // 관련 이슈: https://github.com/pmndrs/react-three-fiber/discussions/1752
      box?: never;
    }
  }
}
```

## 5. 새로운 요구사항 - 미니뷰

늘 그렇듯 요구사항은 추가되기 마련이다.  
이번에는 건축 가능한 필지를 클릭한 경우 웹 오른쪽 하단에 작게 3D 미니뷰 UI를 구현하라는 요구사항이 추가되었다.

꽤나 재밌는 작업이라고 생각이 들었고, 비즈니스 적으로도 합리적이다라는 생각이 들어 추가된 요구사항치고는 재밌게 작업했다.

이전에 앱 크래쉬 사건 이후로 3D 관련 상태는 리렌더링을 최소화하기 위해 모두 전역 상태 관리 라이브러리로 관리하고 있었다.

그래서 필지를 클릭할 때마다 좌표와 같은 전역 상태가 업데이트 되고 있어서,
미니뷰에서는 해당 좌표 상태를 가져와 3D Scene에 넣어주기만 하면 되었다!
생각보다 크게 할 작업이 없었고, 이전에 구조를 괜찮게 설계했나 싶어 뿌듯했다.

그래서 주로 집중한 작업은 3D Scene에 애니메이션을 추가하는 작업이였다.  
미니뷰의 경우 카메라 회전과 건축물을 올리는 애니메이션 두 개를 중첩해서 역동적으로 구현해보았다.

먼저 회전 애니메이션의 경우 나침반과 동일하게 useThree의 camera와 useFrame을 사용한다.  
매프레임 별로 카메라 위치를 조정하여, 타겟을 중심으로 회전하는 애니메이션을 구현하였다.

```tsx
const angle = useRef(0);
const { camera } = useThree();

useFrame(() => {
  angle.current -= 0.005;

  // 카메라가 도는 원 궤도의 반지름 설정 (건물 높이에 비례)
  const radius = maxValueRef.current * 3;

  // 삼각함수를 이용해 x, y, z 좌표를 계산하여 원을 따라 카메라 이동
  camera.position.set(
    Math.sin(angle.current) * radius, // x축 위치
    maxValueRef.current, // y축 높이는 고정
    Math.cos(angle.current) * radius // z축 위치
  );

  camera.lookAt(cameraTarget);
});
```

<br />

건축물이 올라오는 애니메이션은 다음과 같은 방식으로 구현했다.  
각 층별 모델의 Y축 위치를 0에서 시작하여, 목표 위치(floorPosition)까지 일정 속도로 점진적으로 올리는 방식이다.  
이를 통해 건축물이 서서히 생성되는 듯한 시각적 효과를 연출할 수 있다.

```tsx
// 처음에는 y = 0 위치에서 시작
const animatedHeight = useRef(0);

useFrame(() => {
  // 아직 건축물이 목표 위치보다 낮은 경우, 애니메이션을 계속 진행
  if (animatedHeight.current < floorPosition) {
    // 상승 속도 설정
    const riseSpeed = 0.1;

    // 현재 높이에 속도를 더하되, 목표 위치를 초과하지 않도록 제한
    animatedHeight.current = Math.min(
      animatedHeight.current + riseSpeed,
      floorPosition
    );

    // 건축물 mesh, 건축물 외곽선 매 프레임별로 업데이트
    meshRef.current.position.y = animatedHeight.current;
    edgesRef.current.position.y = animatedHeight.current;
  }
});
```

![](/images/2025-03-24-3D-simulation/image9.gif)

## 6. 최적화

구현을 마친 뒤, 마지막 한 주는 성능 점검에 특히 신경을 썼다.
내가 만든 기능이 전체 프로젝트에 부정적인 영향을 주지는 않는지, 이전의 앱 크래쉬 경험 이후 좀 더 치밀하게 체크하고자 노력했다.

### 6.1 메모리 누수 개선

Three.js 코드를 전반적으로 점검하며 메모리 누수가 발생할 수 있는 지점을 모두 확인했다.  
그중에서도 메모리 누수 가능성이 높다고 판단된 부분은 외곽선을 그리는 로직이었다.

더 얇은 선을 표현하기 위해 `edgesGeometry` 대신 `LineSegments2` 객체를 사용했는데,  
이 객체가 메모리 누수의 원인이 될 수 있다고 판단되었다.

현재 사용 중인 R3F에서는 일반적인 Three.js 객체들을 렌더 트리에서 제거하면 WebGL 리소스를 자동으로 정리해주지만,  
개발자가 직접 생성해 primitive에 전달한 객체는 자동 해제가 되지 않는다.  
따라서 컴포넌트가 언마운트될 때 dispose() 메서드를 호출하여 메모리를 수동으로 해제해주었다.

```tsx
// primitive에 할당하기 위해 직접 생성한 Three.js 객체는 수동으로 메모리 해제
useEffect(() => {
  return () => {
    extrudeGeo.dispose();
    edgesGeo.dispose();
    lineMat.dispose();
  };
}, [extrudeGeo, edgesGeo, lineMat]);
```

![](/images/2025-03-24-3D-simulation/image10.png)

개발자 도구의 Performance 탭에서 Memory 체크박스를 선택한 뒤 녹화를 진행한 결과,  
JS heap, Nodes, Listeners 그래프가 계단식으로 계속 증가하지 않고, 일정 시간마다 감소하는 모습을 확인할 수 있었다.

### 6.2 CPU 점유율 개선

특정 상황에서(개발자 도구의 Elements 탭이 열려 있고 3D Scene을 빠르게 회전할 때)
CPU 점유율이 최대 150%까지 치솟는 현상이 발생했다.

일반적인 서비스 평균 CPU 점유율이 20~50% 수준이었던 것을 고려할 때,
이 문제는 명확한 성능 이슈로 판단되었다.

이 문제를 발생시키는 트리거가 회전에 있었기 때문에, 나침반 컴포넌트 쪽의 최적화 작업을 진행했다.

```tsx
const { camera } = useThree();
const [compassRotation, setCompassRotation] =
  useRecoilState(compassRotationState);
const dirRef = useRef(new THREE.Vector3());
const compassRotationRef = useRef(compassRotation);

useFrame(() => {
  const dir = dirRef.current;
  camera.getWorldDirection(dir);

  const rotation = THREE.MathUtils.radToDeg(Math.atan2(dir.x, dir.z)) - 180;

  // 이전 값과의 차이가 3도 이상일 때만 상태 업데이트 (불필요한 리렌더링 방지)
  if (Math.abs(rotation - compassRotationRef.current) > 3) {
    compassRotationRef.current = rotation;
    setCompassRotation(rotation);
  }
});
```

나침반 리렌더링 주기를 줄여 불필요한 상태 업데이트를 방지한 결과,
최대 CPU 점유율이 50%를 넘지 않도록 안정화시킬 수 있었다.

---

이렇게 한 달여간의 첫 장기 프로젝트가 마무리되었다.  
처음 면접에서 대표님께서 "이런 기능, 만들 수 있겠어요?"라고 물으셨을 때,  
사실 완벽한 확신은 없었지만 그래도 자신 있게 "네!"라고 대답했던 기억이 난다.

나를 필요로 하는 자리에서 내 기술로 역할을 증명할 수 있었던 업무였기에,  
한 달 동안 책임감을 갖고 최선을 다할 수 있었고, 즐겁게 몰입할 수 있었다.  
아무래도 오래 기억에 남을 프로젝트가 될 것 같다.
