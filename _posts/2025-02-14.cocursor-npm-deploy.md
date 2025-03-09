---
layout: single
title: cocursor npm 모듈 배포
categories: cocursor
excerpt: 만든 클라이언트 코드를 npm에 배포해보자!
---

만든 클라이언트 코드를 npm에 배포해보자!

배포 시에 이 영상을 참고했다.
https://www.youtube.com/watch?v=GVN9d1rFeCo&list=LL&index=3

## 1. 프로젝트 구조

지금까지 완성한 코드 중 배포할 부분만 따로 빼자.

```
📦src
 ┣ 📂components
 ┃ ┣ 📜CoCursorContext.tsx      // CoCursor Hook
 ┃ ┣ 📜CoCursorProvider.tsx     // CoCursor Provider
 ┃ ┣ 📜Cursor.tsx               // 커서 컴포넌트
 ┃ ┗ 📜types.d.ts               // 타입 정의 파일
 ┣ 📂styles
 ┃ ┗ 📜cursor.css               // 커서 스타일
 ┣ 📂utils
 ┃ ┣ 📜stringToColor.ts
 ┃ ┗ 📜throttle.ts
 ┗ 📜index.ts
```

그리고 src 아래에 `index.ts` 배포 시 내보낼 부분을 정의하자.

```tsx
import CoCursorProvider from "./components/CoCursorProvider";
import { useCoCursor } from "./components/CoCursorContext";

export default CoCursorProvider;
export { useCoCursor };
```

사용하는 측에서는 다음과 같이 사용할 수 있다.

```js
import CoCursorProvider, { useCoCursor } from "cocursor";

// ...
```

## 2. tsconfig.json

TS를 사용하므로 tsconfig.json 파일을 설정해줘야 한다.
영상과 동일하게 설정하였다.

```json
{
  "compilerOptions": {
    "target": "ES5",
    "lib": ["DOM", "DOM.Iterable", "ESNext"],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "module": "ESNext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx"
  },
  "include": ["src"]
}
```

## 3. rollup

배포하기 위해서는 JS 모듈 번들러를 사용해야 한다.  
번들러 도구로 Rollup을 사용하였다.  
마찬가지로 영상과 동일하게 설정하였는데, 이해하고 사용하진 않았다. :)

```json
// package.json
// rollup 관련 dependency

{
  "devDependencies": {
    "@rollup/plugin-commonjs": "^28.0.2",
    "@rollup/plugin-node-resolve": "^16.0.0",
    "@rollup/plugin-terser": "^0.4.4",
    "@rollup/plugin-typescript": "^12.1.2",
    "rollup-plugin-dts": "^6.1.1",
    "rollup-plugin-peer-deps-external": "^2.2.4",
    "rollup-plugin-postcss": "^4.0.2"
  }
}
```

```js
// rollup.config.js

import resolve from "@rollup/plugin-node-resolve";
import commonjs from "@rollup/plugin-commonjs";
import typescript from "@rollup/plugin-typescript";
import dts from "rollup-plugin-dts";
import terser from "@rollup/plugin-terser";
import peerDepsExternal from "rollup-plugin-peer-deps-external";
import postcss from "rollup-plugin-postcss";

const packageJson = require("./package.json");

export default [
  {
    input: "src/index.ts",
    output: [
      {
        file: packageJson.main,
        format: "cjs",
        sourcemap: true,
      },
      {
        file: packageJson.module,
        format: "esm",
        sourcemap: true,
      },
    ],
    plugins: [
      peerDepsExternal(),
      resolve(),
      commonjs(),
      typescript({ tsconfig: "./tsconfig.json" }),
      terser(),
      postcss(),
    ],
    external: ["react", "react-dom"],
  },
  {
    input: "src/index.ts",
    output: [{ file: packageJson.types }],
    plugins: [dts.default()],
    external: [/\.css/],
  },
];
```

## 4. npm publish

`.npmignore` 파일을 설정하여 배포 시 포함되지 않을 파일을 설정할 수 있다.
최소한의 dist 파일만 남기기 위해 다음과 같이 설정하였다.

```
.git
.gitignore

node_modules
package-lock.json

src
```

이후 `npm login` => `npm publish` 하면 npm 사이트 게시된 내 모듈을 확인할 수 있다. :)

![alt text](/images/2025-02-14.cocursor-npm-deploy/image.png)
