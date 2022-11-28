---
layout: post
title:  "프론트엔드(npm) 프로젝트 빌드 시간 줄이기"
date:   2022-11-28 16:13:00 +0900
tags: ["vite", "frontend", "npm", "react", "build", "optimization"]
---
> ```vite```, ```frontend```, ```npm```, ```react```, ```build```, ```optimization```    

<h3>Table Of Content</h3>

- [발단](#발단)
- [문제 해결](#문제-해결)
- [기타](#기타)
  - [1. devDependency 명시만으로는 bundle에서 제외되지 않는다.](#1-devdependency-명시만으로는-bundle에서-제외되지-않는다)
  - [2. webpack 도 동일하게 external 옵션을 제공한다.](#2-webpack-도-동일하게-external-옵션을-제공한다)
  - [3. app(front-server) 에서도 external + CDN 조합을 통해 적용할 수 있다.](#3-appfront-server-에서도-external--cdn-조합을-통해-적용할-수-있다)

## 발단
진행중인 프로젝트의 프론트엔드 아키텍쳐는 다음과 같다.

```
Front-Server                            : 클라이언트에게 페이지를 서빙하는 실제 서버
    +-------- MicroService #1 (npm)     
    +-------- MicroService #2 (npm)     
    ...
    +-------- MicroService #n (npm)     : 페이지를 구성하는 컴포넌트, 함수 라이브러리
    ...
    +-------- OpenSource #1 (npm)
    +-------- OpenSource #2 (npm)
    ...
    +-------- OpenSource #n (npm)       : 오픈소스 라이브러리
```

이러한 구조의 가장 큰 특징은
```모든 종속성이 빌드타임에 통합된다는 것``` 이다.

빌드타임 통합은
- 애플리케이션의 ```안정성```을 높이고,
-  ```가시적인 통합```을 통해 테스트가 용이할 수 있으나,
-  자칫하면, 배포를 위한 ```빌드시간```이 기하급수로 늘어날 수 있다.


이번 프로젝트에서도, 이와 같은 일이 발생했다.      
1개의 ```front-server```가 ```14개의 마이크로 서비스``` 라이브러리로 구성되어 있었으며, 마이크로 서비스가 각각 비대해짐에 따라, build 시간은 점진적으로 증가했다.


| Bundle Size                       | Build Time |
| --------------------------------- | ---------- |
| 7997.19 KiB / gzip: 2134.17 KiB   | 03:57      |
| 41196.05 KiB / gzip: 10193.83 KiB | 08:19      |
| 57758.64 KiB / gzip: 14203.73 KiB | 24:16      |

> 배포하는데 24분이 소요되는 것은 정말 끔찍한 일이다.

## 문제 해결

문제의 핵심은 번들사이즈에 있으며, 이를 줄이기 위해 vite (rollup)에서 제공하는 external 옵션을 적극 도입하여 다음과 같은 룰을 일괄 적용했다.
1. MicroService는 어떠한 dependecy도 직접 가지지 않는다.
    ```json
    // package.json
    {
        "dependencies": {}
    }
    ```
2. Runtime에 필요한 번들은 peerDependencies로 명세한다.
    ```json
    // package.json
    {
        "dependencies": {},
        "peerDependencies": {
            "react": "^18.2.0",
            "react-dom": "^18.2.0",
            ...
        }
    }
    ```
3. peerDependencies는 external 옵션으로 추가한다.
    ```typescript
    // vite.config.ts
    import { defineConfig } from 'vite'
    import react from '@vitejs/plugin-react'
    import peerDepsExternal from 'rollup-plugin-peer-deps-external';

    // https://vitejs.dev/config/
    export default defineConfig({
    plugins: [react({
        exclude: [/\.stories\.(mdx|[tj]sx?)$/, /\.bar\.(js)$/],
    })],
    build: {
        rollupOptions: {
        // external: ['react', 'react-dom'],  ==> peerDepsExternal 플러그인으로 대체
            output: {
                // Provide global variables to use in the UMD build
                // for externalized deps
                globals: {
                    "react": "React",
                    "react-dom": "ReactDOM"
                }
            },
            plugins: [
                peerDepsExternal(),
            ]
        }
    }
    })
    ```

4. Storybook Test를 위해 필요한 번들은 devDependencies로 명세한다.
    ```json
    // package.json
    {
        "dependencies": {},
        "peerDependencies": {
            "react": "^18.2.0",
            "react-dom": "^18.2.0",
            ...
        },
        "devDependencies": {
            "react": "^18.2.0",
            "react-dom": "^18.2.0",
            ...
        }
    }
    ```

모든 MicroService에 이와 같은 룰을 적용하여 다음과 같은 성과를 달성했다.

| Bundle Size                            | Build Time  |
| -------------------------------------- | ----------- |
| 7997.19 KiB / gzip: 2134.17 KiB        | 03:57       |
| 41196.05 KiB / gzip: 10193.83 KiB      | 08:19       |
| 57758.64 KiB / gzip: 14203.73 KiB      | 24:16       |
| ------------ 적용 후 ------------      |             |
| ```34144.25 KiB / gzip: 8224.87 KiB``` | ```03:16``` |

> Bundle Size : 약 23.6 MB 감소   
> Build Time : 21:00 감소

## 기타

### 1. devDependency 명시만으로는 bundle에서 제외되지 않는다.
vite 가 번들링을 진행할 때 종속 번들의 참조는 node_modules에 다운로드된 패키지와, 번들 설정에 기인한다.    
즉, package.json에 명시된 deps의 종류는 아무런 영향을 미치지 않는다.       
따라서, bundle에서 제외하고자 한다면, 반드시 external 옵션에 명시해야한다.

이를 더 확실하게 강제하는 방법은 CI/CD 스크립트에서
```
// 빌드를 위한 종속성(vite, babel, ttypescript...)은 global 설치
yarn install --producetion=true
```
와 같이 devDeps 만 다운로드 받도록 하여, 의도치 않는 종속성 빌드를 방지할 수 있다.

### 2. webpack 도 동일하게 external 옵션을 제공한다.


### 3. app(front-server) 에서도 external + CDN 조합을 통해 적용할 수 있다.

특정 MicroService 또는 Open Source의 크기가 크다면, 최종 endpoint 번들인 app 에서도 external에 명시하고, CDN 을 통해 같은 효과를 달성할 수 있다.

이러한 패턴을 조금 더 연구해보면, 빌드타임 통합에서 런타임 통합으로 아키텍쳐의 변화를 가져갈 수도 있을 것 같다.     




