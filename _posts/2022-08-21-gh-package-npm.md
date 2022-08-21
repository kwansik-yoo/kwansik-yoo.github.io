---
layout: post
title:  "Github Package를 통해 private Repository 구성하기"
date:   2022-08-21 13:21:43 +0900
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
- [자동 버저닝](#자동-버저닝)

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
1. 배포할 github repository를 생성합니다.       

2. 배포할 프로젝트를 npm 프로젝트로 초기화 합니다.     
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

3. 라이브러리에서 제공할 간단할 모듈을 추가합니다.     
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

1. ```package.json```을 다음과 같이 수정합니다.   

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

2. 프로젝트 루트에 .npmrc 파일을 추가합니다.    
```shell  
$ touch .npmrc
```

3. @{USER_NAME} scope 에 해당하는 모듈에 대한 registry 정보를 서술합니다.

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

1. github action을 서술할 파일을 생성합니다.    

```shell
$ mkdir -p .github/workflows       
$ touch .github/workflows/release.yml       
```

2. 수행할 action을 명세합니다.  

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

1. release 브랜치를 생성하고 코드를 푸시합니다.     
```shell
$ git checkout -b release
$ git push -u origin release
```

2. action이 수행되는 것을 확인할 수 있습니다.    

![](/assets/2022-08-21-gh-package-npm/github-action.png)

3. packages 메뉴를 통해 배포된 패키지를 확인할 수 있습니다.    

![](/assets/2022-08-21-gh-package-npm/github-package.png)

## yarn     

## 자동 버저닝     
    

