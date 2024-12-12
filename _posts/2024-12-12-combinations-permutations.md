---
layout: single
title: "조합과 순열"
categories: 자료구조&알고리즘
---

조합: 서로 다른 n개의 물건에서 순서를 생각하지 않고 r개를 택하는 경우, nCr  
순열: 서로 다른 n개의 물건에서 r개를 택하여 한 줄로 배열하는 것 (순서를 생각), nPr

### 목표

하나의 배열과 선택할 원소의 개수를 넣었을 때 가능한 모든 배열을 출력하는 함수 만들기

### 핵심 개념

1. 배열을 순회하며 하나를 고정한다.
2. fixed를 제외한(조합: fixed 다음부터의 배열 / 순열: fixed를 제외한 배열) 배열에 대해 재귀적으로 함수를 실행한다.
3. 아래에서부터 results를 합친다.

```
// 예시
시작
  1을 선택(고정)하고 -> 나머지 [2, 3, 4] 중에서 2개씩 조합을 구한다.
  [1, 2, 3] [1, 2, 4] [1, 3, 4]
  2를 선택(고정)하고 -> 나머지 [3, 4] 중에서 2개씩 조합을 구한다.
  [2, 3, 4]
  3을 선택(고정)하고 -> 나머지 [4] 중에서 2개씩 조합을 구한다.
  []
  4을 선택(고정)하고 -> 나머지 [] 중에서 2개씩 조합을 구한다.
  []
종료
```

---

### 조합

재귀 함수를 작성할 때는 종료 조건과 각 단계마다 어떤 동작을 수행할 것인지 기술하자.

종료 조건: selectNum이 1이 될 때  
각 단계마다 동작:  
fixed 다음부터 배열의 조합을 구한다.  
결과가 리턴되면 fixed와 조합 배열을 합친다.  
results 배열에 push한다.

```js
function getCombinations(arr, selectNum) {
  const results = [];
  if (selectNum === 1) return arr.map((each) => [each]);

  arr.forEach((fixed, idx) => {
    const rest = arr.slice(idx + 1);
    const combinations = getCombinations(rest, selectNum - 1); // fixed 다음부터 배열의 조합을 구한다.
    const attached = combinations.map((each) => [fixed, ...each]); // 결과가 리턴되면 fixed와 조합 배열을 합친다.
    results.push(...attached); // results 배열에 push한다.
  });

  return results;
}
```

### 순열

순열을 구하는 함수는 조합과 한 부분을 제외하고 모두 같다.  
순서를 신경 쓰기 때문에 fixed 다음부터의 배열이 아닌 fixed를 제외한 배열에 대해 재귀적으로 함수를 호출하면 된다.

```js
function getPermutations(arr, selectNum) {
  const results = [];
  if (selectNum === 1) return arr.map((each) => [each]);

  arr.forEach((fixed, idx) => {
    const rest = [...arr.slice(0, idx), ...arr.slice(idx + 1)];
    const permutations = getPermutations(rest, selectNum - 1); // fixed를 제외한 배열의 순열을 구한다
    const attached = permutations.map((each) => [fixed, ...each]); // 결과가 리턴되면 fixed와 순열 배열을 합친다.
    results.push(...attached); // results 배열에 push한다.
  });

  return results;
}
```
