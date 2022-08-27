---
layout: post
title:  "NPM Auto Versioning을 위한 파이프라인 구성하기"
date:   2022-08-27 17:50:00 +0900
tags: npm yarn github-package automation versioning
---
> ```npm``` ```yarn``` ```github-package``` ```automation``` ```versioning```   

> [Go To Repo](https://github.com/kwansik-yoo/github-package-npm-demo)   


<h3>Table Of Content</h3>

- [Intro](#intro)
- [자동 버저닝을 위한 github action 정의](#자동-버저닝을-위한-github-action-정의)
- [릴리즈 (beta)](#릴리즈-beta)

## Intro    
npm 패키지를 배포할 때 versioning의 자유도는 높습니다.     
때로는 이 자유도가 식별하기 힘든 버전의 혼란을 야기하기도 합니다.     
> - foo@1.1.0-beta.1
> - foo@1.1.0-beta1   
> - foo@1.1.0-1

이러한 문제를 방지하기 위해선 두가지 문제를 고려해볼 필요가 있습니다.        
1. 로컬 배포 금지         
2. 파이프라인을 통한 자동 버저닝      

로컬 배포를 금지하기 위해선 write 토큰을 공개하지 않는 방법으로 해결이 가능합니다.    

이 글에서는 자동 버저닝 방법에 대해 소개합니다.     
파이프라인을 구성하는 다양한 플랫폼이 있지만 오늘은 github action을 기준으로 다루고자 합니다.


> {USER_NAME}로 표기된 부분은 본인의 github username으로 대체합니다.     

## 자동 버저닝을 위한 github action 정의          

✅ github action 템플릿 (major 버전 기준)    

> release-major.yml    

```yml
name: Release

on:
  push:
    branches: 
      - release-beta

env:
  GH_READ_PACK_TOKEN: ${{secrets.GITHUB_TOKEN}}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 12
      - run: yarn
      - run: yarn test

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
      - run: yarn
      - run: echo "Check latest version..." && yarn info --json > info.json
      - run: echo "Auto versioning..." && yarn version --new-version $(echo $(node -p "require('./info.json').data.version"))
      - run: yarn version --no-git-tag-version --prerelease
      - run: yarn publish
        env:
          NODE_AUTH_TOKEN: ${{secrets.GITHUB_TOKEN}}
```

✅ 각 버전별(major, minor, patch, beta) 달라지는 부분은 다음과 같습니다.    

```diff
name: Release

on:
  push:
    branches: 
-      - release-major
+      - release-minor || release-patch || release-beta


  publish-gpr:
    steps:
-      - run: yarn version --no-git-tag-version --major
+      - run: yarn version --no-git-tag-version --minor || --patch || prerelease
      
```

위와 같이 각 버전 형태별 yml 파일을 별도로 생성해줍니다.

## 릴리즈 (beta)     

```1.0.1``` 가 최신 버전임을 확인하고, beta 버전인 ```1.0.2-0``` 으로 자동 버저닝 후 배포합니다.   

![](/assets/2022-08-27-npm-auto-versioning/auto-versioning-release.png)