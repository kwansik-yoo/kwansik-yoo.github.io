---
layout: post
title:  "프론트엔드(npm) 프로젝트 빌드 시간 줄이기"
date:   2022-11-28 16:13:00 +0900
tags: ["vite", "frontend", "npm", "react", "build", "optimization"]
---
> ```vite```, ```frontend```, ```npm```, ```react```, ```build```, ```optimization```    


<h2> Table Of Content </h2>

- [1. 발단](#1-발단)
- [2. 문제 해결 과정](#2-문제-해결-과정)
  - [2.1. 문제의 정의](#21-문제의-정의)
  - [2.2. bundle 크기를 줄이기 위한 노력](#22-bundle-크기를-줄이기-위한-노력)
    - [2.2.1. MicroService](#221-microservice)
    - [2.2.2. MicroApp](#222-microapp)
  - [2.3. 적용 후 효과](#23-적용-후-효과)
- [3. 결론](#3-결론)
- [4. 기타](#4-기타)
  - [4.1. ```devDependencies```는 ```bundle```에서 제외되지 않는다.](#41-devdependencies는-bundle에서-제외되지-않는다)
  - [4.2. webpack 도 동일하게 external 옵션을 제공한다.](#42-webpack-도-동일하게-external-옵션을-제공한다)
  - [4.3. ```external + CDN``` 조합을 통해 ```Composition in Rumtime```을 적용할 수 있다.](#43-external--cdn-조합을-통해-composition-in-rumtime을-적용할-수-있다)

## 1. 발단       
진행중인 프로젝트의 프론트엔드 아키텍쳐는 다음과 같다.     

```
MicroApp                                            : 클라이언트에게 서빙할 페이지 (NginX | NodeJS 로 serving)
    +-------- MicroService #1, #2, ..., #n (npm)    : 페이지를 구성하는 컴포넌트, 함수 라이브러리
    +-------- OpenSource #1, #2, ..., #n (npm)      : 오픈소스 라이브러리
```

이러한 구조의 가장 큰 특징은 **```모든 종속성이 빌드타임에 통합된다는 것```** 이다.    

**```빌드타임 통합```** 은    
- ```npm```을 통한 가시적인 통합을 통해 모듈간 버전 호환성을 미리 체크할 수 있다.
- 하지만 각 MicroService의 종속성이 복잡해 짐에 따라 자칫하면, 배포를 위한 ```빌드시간```이 기하급수로 늘어날 수 있다.    

---

이번 프로젝트에서도, 이와 같은 일이 발생했다.      
1개의 ```MicroApp```이 ```14개의 MicroService``` 로 구성되어 있었으며, 각 ```MicroService```가 비대해짐에 따라,    
build 시간은 점진적으로 증가했다.    

> 빌드시간 추이      
>| Bundle Size                       | Build Time |
>| --------------------------------- | ---------- |
>| 7997.19 KiB / gzip: 2134.17 KiB   | 03:57      |
>| 41196.05 KiB / gzip: 10193.83 KiB | 08:19      |
>| 57758.64 KiB / gzip: 14203.73 KiB | 24:16      |

**배포하는데 24분이 소요되는 것은 정말 끔찍한 일이다.**

## 2. 문제 해결 과정    
> 바쁘신 분들은 바로 [결론](#3-결론)으로...

### 2.1. 문제의 정의     

javascript 프로젝트의 ```build```의 의미는 ```bundling```과 같다.       
```bundling```이란 여러 개의 js 모듈을 entry point를 기준으로 하나, 또는 여러개의 js 파일로 합쳐, ```선언의 순서```, 비효율적인 ```IO(network|file ...)``` 등의 문제를 해결한다. *(css, static asset에 대한 설명은 생략)*   
이때 ```bundling```의 원리를 생각해보면, 
- 수백 ~ 수만개의 모듈간 종속성을 비교해야하며,     
- 이를 위해 각 모듈의 ```bundle```을 memory에 올려두거나 어떠한 IO 작업을 통해 비교해야 할 것이다.    
- 따라서 각 종속 모듈의 ```bundle```의 크기가 크다면, 번들링을 위해 요구되는 memory와 시간이 선형적(또는 지수적(exponential))으로 증가할 수 밖에 없다.        

***따라서 ```bundle```의 크기를 줄인다면, ```build``` 시간을 줄일 수 있을 것이라는 가정을 설정했다.***

### 2.2. bundle 크기를 줄이기 위한 노력     

우리가 bundle 크기를 줄일 수 있는 대상은 ```Opensource``` 를 제외한, ```MicroApp```, ```MicroService``` 이다.     

#### 2.2.1. MicroService              

***```MicroService bundle```의 목적은 오직 ```business logic```의 제공이다.***     
> 즉, ```client```에게 바로 ```serving```될 bundle이 아니라는 의미이기도 하다.        

다음의 케이스를 살펴보자.   
- ```GitReporter``` 라는 Microservice는 git url 정보를 통해 git의 report 정보를 가공하여 제공한다.     
- ReportAPI.js    
```typescript
import axios from 'axios';
import makeReport from './makeReport';

export const getReport = (gitUrl: string): Promise<GitReport> => {
    const res = await axios.get(`https://git-reporter/report?gitUrl=${gitUrl}`);
    return makeReport(res);
}
```     
- package.json    
```json
{
  "name": "git-reporter",
  "dependencies": {
    "axios": "^1.2.1",
  },
  "peerDependencies": {},
  "devDependencies": {
    "@types/node": "^18.11.11",
    "@vitejs/plugin-react": "^2.2.0",
    "typescript": "^4.6.4",
    "vite": "^3.2.3"
  }
}
```    

별도의 설정없이 build를 진행하면 다음과 같은 ```bundle```을 확인할 수 있다.     
```bash
❯ yarn build
yarn run v1.22.18
$ tsc && vite build
vite v3.2.5 building for production...
✓ 47 modules transformed.
dist/index.es.js   35.69 KiB / gzip: 11.72 KiB
dist/index.umd.js   26.81 KiB / gzip: 10.46 KiB
Done in 6.50s.
```

***```bundle``` 파일을 확인해보면, 우리가 작성한 코드 이외에도 많은 코드가 포함된 것을 확인할 수 있다.   
이는 우리가 사용한 ```axios``` 모듈의 일부가 포함된 것을 의미한다.      
즉, 이 코드 chunk는 불필요하게 포함된 코드라고 할 수 있다.***    


앞서 언급한 바와 같이 ```GitReporter```는 ReportAPI.js 의 ```business logic``` 제공할 책임을 가지지만,     
```client```에게 바로 ```serving``` 되지 않기 때문에 ```axios``` 모듈을 제공할 책임은 가지지 않는다.    
따라서 ```bundle```에 ```axios``` 는 포함될 필요가 없으며, 최종 endpoint project에서 적절한 version의 ```axios```를 포함시키면 된다.     

따라서 다음의 설정을 통해 ```bundle```에서 ```axios``` 모듈을 제외하고 ```build```한다.   
```typescript
// vite.config.ts
import { defineConfig } from 'vite'
const packageJson = require('./package.json');
// https://vitejs.dev/config/
export default defineConfig({
  plugins: [],
  build: {
    ...
    rollupOptions: {
      external: [
          ...Object.keys(packageJson.dependencies)   // ['axios']
      ]
    }
  }
})
```
```bash
❯ yarn build
yarn run v1.22.18
$ tsc && vite build
vite v3.2.5 building for production...
✓ 3 modules transformed.
dist/index.es.js   0.20 KiB / gzip: 0.17 KiB
No name was provided for external module 'axios' in output.globals – guessing 'axios'
dist/index.umd.js   0.54 KiB / gzip: 0.35 KiB
Done in 7.04s.
```

***```bundle``` 파일을 확인해보면, 우리가 작성한 코드만 포함되어 있는 것을 확인할 수 있다.***    

마지막으로, ```endpoint project```에서 version 호환을 확인할 수 있도록 ```peerDeps```를 명시한다.    

- package.json    
```diff
{
  "name": "git-reporter",
  "dependencies": {
    "axios": "^1.2.1",
  },
  "peerDependencies": {
+    "axios": "^1.2.1",
  },
  "devDependencies": {
    "@types/node": "^18.11.11",
    "@vitejs/plugin-react": "^2.2.0",
    "typescript": "^4.6.4",
    "vite": "^3.2.3"
  }
}
```    

> 참고    
> MicroService에서 ```dependencies```를 비워두는 방식(test를 위해 ```devDependencies``` 추가)도 시도해보았으나,    
> ```diff
>{
>  "name": "git-reporter",
>  "dependencies": {
>-    "axios": "^1.2.1",
>  },
>  "peerDependencies": {
>+    "axios": "^1.2.1",
>  },
>  "devDependencies": {
>    "@types/node": "^18.11.11",
>    "@vitejs/plugin-react": "^2.2.0",
>    "typescript": "^4.6.4",
>    "vite": "^3.2.3",
> +    "axios": "^1.2.1" 
>  }
>}
>```    
>이 관계가 ```n-depth```로 늘어나면, ```dependency drilling``` 문제가 발생한다.    

#### 2.2.2. MicroApp              
모든 ```MicroService```가 앞선 규칙을 적용했다면, ```MicroApp```은 ```peerDeps```만 잘 고려하여, 종속성을 최종 ```bundle```에 포함시키면 된다.      

단, 때에 따라, 대용량 ```bundle``` 종속성이 있다면,     
 - ```external``` 옵션과,
 - ```html```를 통한 통합으로 ```build``` 시간을 개선할 수 있다. *(단, 실제로 효과는 미미하다.)*    
```html
<head>
    <script src="some_big.js"></script>
</head>
```


### 2.3. 적용 후 효과     

 | Bundle Size                            | Build Time  |
 | -------------------------------------- | ----------- |
 | 7997.19 KiB / gzip: 2134.17 KiB        | 03:57       |
 | 41196.05 KiB / gzip: 10193.83 KiB      | 08:19       |
 | 57758.64 KiB / gzip: 14203.73 KiB      | 24:16       |
 | ------------ 적용 후 ------------      |             |
 | ```34144.25 KiB / gzip: 8224.87 KiB``` | ```03:16``` |

> Bundle Size : 약 23.6 MB 감소   
> Build Time : 21:00 감소



## 3. 결론      

1. ```js project```가 제공할 ```bundle```은 두가지 목적으로 나누어 정의한다.    
   1. ```business logic``` 제공 => 외부 종속성 모듈은 ```bundle```에 ```미포함```    
   2. ```client```에게 ```serving``` => 모든 외부 종속성을 ```bundle```에 ```대부분 포함``` 
      - 때에 따라 ```run-time``` 통합도 고려 가능
2. ```bundler``` 설정을 통해 ```bundle```에서 제외할 모듈 설정     

## 4. 기타     

### 4.1. ```devDependencies```는 ```bundle```에서 제외되지 않는다.        
```package.json``` 에는 3가지의 ```dependency``` 가 있다.    
- dependencies: production 구동에 반드시 필요한 종속성    
- devDependencies: production 구동에 필요하지 않지만, 개발간 필요한 종속성     
- peerDependencies: version 호환이 필요한 종속성         

만약 위의 ```GitReporter``` 예시에서 다음과 같이 package.json이 바뀐다면 무엇이 달라질까?    
```diff
// package.json
{
  "name": "git-reporter",
  "dependencies": {
-    "axios": "^1.2.1"
  },
  "peerDependencies": {},
  "devDependencies": {
    "@types/node": "^18.11.11",
    "@vitejs/plugin-react": "^2.2.0",
    "typescript": "^4.6.4",
    "vite": "^3.2.3",
+    "axios": "^1.2.1"
  }
}
```    
***아무것도 달라지지 않는다.***   
```bash
❯ yarn build
yarn run v1.22.18
$ tsc && vite build
vite v3.2.5 building for production...
✓ 47 modules transformed.
dist/index.es.js   35.69 KiB / gzip: 11.72 KiB
dist/index.umd.js   26.81 KiB / gzip: 10.46 KiB
Done in 6.89s.
```

이를 통해 우리가 알 수 있는 것은,     

```bundler``` 가 ```bundling```을 진행할 때 ```target module(GitReporter)```에서 참조하는 ```source module(axios)```이 ```node_modules```에 존재한다면, ```bundle``` 대상으로 식별한다.    
***즉, package.json에 명시된 deps의 종류는 아무런 영향을 미치지 않는다.***       

따라서, ```bundle```에서 제외하고자 한다면, 반드시 ```external``` 옵션에 명시해야한다.     

그렇다면, ```dependencies```와 ```devDependencies```는 아무런 차이가 없는 것일까?    
다음의 케이스를 생각해보자.    
- ```GitReporter``` 라는 ```MicroService``` 를 npm 에 배포하였다.    
- ```WorkTogether```라는 ```MicroApp``` 에서 이 모듈을 사용한다.     
만약, ```GitRepoter```의 package.json에 ```axios``` 가  
- ```devDependencies```에만 명시되어 있다면, ```WorkTogther```는 직접 ```axios``` 모듈을 ```install``` 해야 한다.         
- ```dependencies```에 명시되어 있다면, ```WorkTogther```는 직접 ```axios``` 모듈을 ```install``` 하지 않아도 된다. (```npm list -a```)         

더 나아가, 개발자의 실수에 의해 ```devDependencies```에만 명시하고, ```bundle```에 숨어들어와 배포된 경우,     
엄격한 CI/CD를 위해,    
```
// 빌드를 위한 종속성(vite, babel, ttypescript...)은 global 설치
yarn install --producetion=true
```
와 같이 devDeps 만 다운로드 받도록 하여, 의도치 않는 종속성 빌드를 방지할 수 있다.


### 4.2. webpack 도 동일하게 external 옵션을 제공한다.     


### 4.3. ```external + CDN``` 조합을 통해 ```Composition in Rumtime```을 적용할 수 있다.      




