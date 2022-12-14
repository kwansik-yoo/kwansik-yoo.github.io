---
layout: post
title:  "Github Package를 통해 private Repository 구성하기"
date:   2022-08-27 15:14:00 +0900
tags: npm yarn github-package
---
> ```npm``` ```yarn``` ```github-package```    

> [Go To Repo](https://github.com/kwansik-yoo/github-package-npm-demo)   



<h3>Table Of Content</h3>

- [Intro](#intro)
- [배포할 프로젝트 생성](#배포할-프로젝트-생성)
- [배포 설정을 위한 npm 설정](#배포-설정을-위한-npm-설정)
- [github action 정의](#github-action-정의)
- [릴리즈](#릴리즈)
- [yarn](#yarn)
- [배포된 라이브러리 사용하기](#배포된-라이브러리-사용하기)

## Intro    
다양한 javascript 라이브러리를 관리하기 위해서는 npm repository가 필요합니다.    
이를 위해서는 다양한 선택지가 있습니다.     
- [NPM](https://www.npmjs.com/)
  - 배포를 위해선 유료 라이센스 구매가 필요합니다.       
- [nexus container](https://hub.docker.com/r/sonatype/nexus3/)    
  - 간단히 docker image를 통해 설치할 수 있습니다.   
  - 단, 공식 이미지는 arm(apple m1)을 지원하지 않습니다.    
- [github package](https://docs.github.com/en/packages)    
  - 일부 용량, 버전 갯수 전까지는 무료 이용이 가능합니다. [(billing)](https://docs.github.com/en/billing/managing-billing-for-github-packages/about-billing-for-github-packages#about-billing-for-github-packages)    
  
이 글에서는 간단히 개인 프로젝트용으로 사용하기 용이한 github package에 대해서 소개합니다.    

> {USER_NAME}로 표기된 부분은 본인의 github username으로 대체합니다.     

## 배포할 프로젝트 생성      
✅ 배포할 github repository를 생성합니다.       

✅ 배포할 프로젝트를 npm 프로젝트로 초기화 합니다.     
```shell
$ git clone git@github.com:{USER_NAME}/github-package-npm-demo.git
$ cd github-package-npm-demo   
$ npm init 
$ npm install
// push to remote git     
$ git add .
$ git commit -m "init npm"
$ git push
```   

✅ 라이브러리에서 제공할 간단할 모듈을 추가합니다.     
```shell
$ touch index.js    
```

> index.js    
```javascript
alert('github-package-npm-demo!!');
```

```shell
// push to remote git     
$ git add .
$ git commit -m "add simple module"
$ git push
```

## 배포 설정을 위한 npm 설정    

✅ ```package.json```을 다음과 같이 수정합니다.   

```diff
{
-  "name": "github-package-npm-demo",
+  "name": "@{USER_NAME}/github-package-npm-demo",
  "scripts": {
-    "test": "echo \"Error: no test specified\" && exit 1"
+    "test": "echo \"Test is executed...\" && exit 0"
  },
+  "publishConfig": {
+   "@{USER_NAME}:registry": "https://npm.pkg.github.com"
+ }
}
```

```shell
// push to remote git     
$ git add .
$ git commit -m "[npm configuration] package.json"
$ git push
```

✅ 프로젝트 루트에 .npmrc 파일을 추가합니다.    
```shell  
$ touch .npmrc
```

✅ @{USER_NAME} scope 에 해당하는 모듈에 대한 registry 정보를 서술합니다.

> .npmrc     

```
@{USER_NAME}:registry=https://npm.pkg.github.com
```

```shell
// push to remote git     
$ git add .
$ git commit -m "[npm configuration] .npmrc"
$ git push
```

## github action 정의    

✅ github action을 서술할 파일을 생성합니다.    

```shell
$ mkdir -p .github/workflows       
$ touch .github/workflows/release.yml       
```

✅ 수행할 action을 명세합니다.  

> release.yml      

```yml
name: Release

on:
  push:
    branches: 
      - release

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 12
      - run: npm ci
      - run: npm test

  publish-gpr:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      packages: write
      contents: read
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 12
          registry-url: https://npm.pkg.github.com/
      - run: npm ci
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{secrets.GITHUB_TOKEN}}
```

- release 브랜치에 푸시되면 action이 실행됩니다.   
- ${{secrets.GITHUB_TOKEN}}은 action 파이프라인에서 자동으로 주입됩니다.     

```shell
// push to remote git     
$ git add .
$ git commit -m "add github actions"
$ git push
```

## 릴리즈     

✅ release 브랜치를 생성하고 코드를 푸시합니다.     
```shell
$ git checkout -b release
$ git push -u origin release
```

✅ action이 수행되는 것을 확인할 수 있습니다.    

![](/assets/2022-08-21-gh-package-npm/github-action.png)

✅ packages 메뉴를 통해 배포된 패키지를 확인할 수 있습니다.    

![](/assets/2022-08-21-gh-package-npm/github-package.png)

## yarn     

✅ github actions 수정    
```diff 
// 생략
-      - run: npm ci
+      - run: yarn
-      - run: npm test
+      - run: yarn test
// 생략
-      - run: npm ci
+      - run: yarn
-      - run: npm publish
+      - run: yarn publish
```

✅ 버전 업그레이드 (patch)    
> package.json     
```diff
- "version": "1.0.0",
+ "version": "1.0.1",
```

```shell
// push to remote git     
$ git add .
$ git commit -m "apply yarn"
$ git push
```

## 배포된 라이브러리 사용하기     

✅ package에 접근할 github access-key를 생성합니다.    

> https://github.com/settings/tokens/new  
> 이때 access 범위는 read package로 설정합니다.    

![](/assets/2022-08-21-gh-package-npm/github_access_key.png)

✅ 발급한 access-token을 환경변수로 선언합니다.     

```shell
$ echo "export GH_READ_PACK_TOKEN=<<some-token>>" >> ~/.zshrc
```

> 발급한 access-token은 github package에 존재하는 모든 public 패키지(다른 유저의 패키지 포함)에 접근이 가능합니다.       

> ⚠️ 발급한 access-token 을 형상관리하는 경우(remote repo에 push하는 경우) 해당 토큰은 github 정책에 의해 삭제되므로 주의합니다.    

✅ .npmrc 설정을 통해 생성한 토큰을 registry auth token 으로 설정합니다.        

> .npmrc     

```diff   
+ //npm.pkg.github.com/:_authToken=${GH_READ_PACK_TOKEN}
@{USER_NAME}:registry=https://npm.pkg.github.com
```



✅ 배포한 라이브러리를 다운로드 받을 수 있습니다.    

```shell
$ yarn add -D @{USER_NAME}/github-package-npm-demo
```    

