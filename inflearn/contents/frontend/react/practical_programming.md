# Repository

- [npm없이 환경 구축](https://github.com/seolys/react-none-npm)

---

npm으로 패키지 관리을 하기위한 package.json파일 생성

```
npm init -y
```

babel 설치

```
npm install @babel/core @babel/cli @babel/preset-react
```

- @babel/core : 바벨의 핵심 기능을 가지고 있는 패키지
- @babel/cli : cli에서 사용할 바이너리가 들어있음.
- @babel/preset-react : react를 위한 플러그인을 모아놓은 모듈

바벨에는 프리셋과 플러그인이라는 개념이 있음

- 플러그인 : 하나의 변환하는 기능.
- 프리셋 : 여러개의 기능을 모아놓은 것.

js파일 컴파일

- src폴더의 파일을 현재경로(.)로 컴파일한다.

```
npx babel --watch src --out-dir . --presets @babel/preset-react
```
