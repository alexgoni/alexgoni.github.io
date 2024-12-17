---
layout: single
title: vscode extension 만들기
---

기존 React에서 사용하는 snippets extension이 길다고 느껴 직접 짧은 버전의 React Snippets을 만들어보았다.  
직접 만들고 배포까지의 과정을 정리해보았다.

## 1. yo 패키지 다운로드

```
npm install -g generator-code

yo code [생성할 이름]
```

## 2. 기능 구현

snippets 데이터 파일을 작성하고 package.json에 입력

```js
  "contributes": {
    "snippets": [
      {
        "language": "javascript",
        "path": "./snippets/data.json"
      },
      {
        "language": "javascriptreact",
        "path": "./snippets/data.json"
      },
      {
        "language": "typescript",
        "path": "./snippets/data.json"
      }
    ]
  },
```

추가로 package.json에 입력해야하는 정보

- publisher
- icon
- keywords
- repository
- categories

<br>

_전체 폴더를 .vscode의 extension 디렉토리에 넣으면 개인 vscode에서 extension을 바로 사용할 수 있지만, market place에 배포하여 다른 환경에서도 사용할 수 있도록 진행하였다._

## 3. Azure

프로젝트 생성 후 토근 발급

## 4. 배포

- `vsce login [사용자 이름]` (marketplace에서 publisher를 등록한다.)
- `vsce package`
- `vsce publish`

배포한 extension 정보: [https://marketplace.visualstudio.com/items?itemName=alexgoni.short-react-snippets](https://marketplace.visualstudio.com/items?itemName=alexgoni.short-react-snippets)

<br>

참고 자료

- [https://www.youtube.com/watch?v=IwC7BSep2zo](https://www.youtube.com/watch?v=IwC7BSep2zo)
- [https://www.youtube.com/watch?v=Yyc7ZuLmtJY&t=133s](https://www.youtube.com/watch?v=Yyc7ZuLmtJY&t=133s)
