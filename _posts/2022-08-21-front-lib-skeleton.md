---
layout: post
title:  "Frontend Lib Skeleton!"
date:   2022-08-21 13:21:43 +0900
tags: ["hands-on", "vite", "eslint", "typescript", "react"]
---
> ```hands-on``` ```vite``` ```eslint``` ```typescript``` ```react```    

> [Go To Repo](https://github.com/kwansik-yoo/front-lib-skeleton)   
<h3>Table Of Content</h3>

- [vite 프로젝트 초기화](#vite-프로젝트-초기화)
- [eslint 초기 설정](#eslint-초기-설정)
- [vite 번들러를 이용한 라이브러리 빌드](#vite-번들러를-이용한-라이브러리-빌드)
  - [1. vite.config.ts 에 build 관련 설정 추가](#1-viteconfigts-에-build-관련-설정-추가)
  - [2. vite.config.ts 에 따른 entry 파일 생성](#2-viteconfigts-에-따른-entry-파일-생성)
  - [3. 라이브러리로 제공할 함수 구현](#3-라이브러리로-제공할-함수-구현)
  - [4. js 번들 빌드](#4-js-번들-빌드)
  - [5. d.ts 파일 자동 생성을 위한 tsconfig 설정](#5-dts-파일-자동-생성을-위한-tsconfig-설정)
  - [6. 타입 참조 파일 위치 정보 추가](#6-타입-참조-파일-위치-정보-추가)
- [테스트를 위한 Storybook 모듈 추가](#테스트를-위한-storybook-모듈-추가)
  - [1. storybook-vite 초기화](#1-storybook-vite-초기화)
  - [2. View(SimpleCalculator) 테스트를 위한 storybook 컴포넌트 작성](#2-viewsimplecalculator-테스트를-위한-storybook-컴포넌트-작성)
  - [3. storybook을 실행하여 확인](#3-storybook을-실행하여-확인)
- [배포 (TODO)](#배포-todo)


## vite 프로젝트 초기화
> https://vitejs.dev/guide/#scaffolding-your-first-vite-project

```shell
yarn create vite front-lib-skeleton --template react-ts
```

## eslint 초기 설정  
> https://eslint.org/docs/latest/user-guide/getting-started

```shell
npm init @eslint/config

// Terminal Interactions
# ? What type of modules does your project use? … 
# ❯ JavaScript modules (import/export)
# ? Which framework does your project use? … 
# ❯ React
# ? Does your project use TypeScript? › Yes
# ? Where does your code run? …  (Press <space> to select, <a> to toggle all, <i> to invert selection)
# ✔ Node
# ? What format do you want your config file to be in? … 
# ❯ JSON
# ? Would you like to install them now? › Yes
# ? Which package manager do you want to use? … 
# ❯ yarn
```

- IntelliJ 사용시 auto fix 설정
![img.png](/assets/2022-08-21-front-lib-skeleton/idea_eslint_autofix.png)   

## vite 번들러를 이용한 라이브러리 빌드        
> https://vitejs.dev/guide/build.html#library-mode   

### 1. vite.config.ts 에 build 관련 설정 추가

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { resolve } from 'path'   
/**
node 내장 모듈인 path 를 추가
이떄 type 추론을 위해 @types/node 모듈을 dev-dependency에 추가   
$ yarn add -D @types/node
*/

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react()],
  build: {
    lib: {
      entry: resolve(__dirname, 'src/comp/index.ts'), // 빌드가 시작되는 파일을 명시
      name: 'front-lib-skeleton',                     // 라이브러리 이름
      // the proper extensions will be added
      fileName: format => `index.${format}.js`        // 번들링된 최종 output 파일의 이름  
    },
    rollupOptions: {
      // make sure to externalize deps that shouldn't be bundled
      // into your library
      external: ['react'],                            // 라이브러리에 React 모듈이 같이 번들링되면 안되기 때문에 명시 (같이 변들링 되면 라이브러리를 실행하는 쪽 React 전역 변수와 충돌 가능성)
      output: {
        // Provide global variables to use in the UMD build
        // for externalized deps
        globals: {
          react: 'React'                              // 라이브러리에 React 모듈이 같이 번들링되면 안되기 때문에 명시 (SYNTAX =>  <<모듈 이름>> : <<변수 이름>> )      
        }
      }
    }
  }
})

```

### 2. vite.config.ts 에 따른 entry 파일 생성    

> src/comp/index.ts    

```  
src
 L comp
    L index.ts
```

### 3. 라이브러리로 제공할 함수 구현

```  
src
 L comp
    L index.ts
    L util
        L calculator.ts
        L logger.ts
    L view
        L SimpleCalculator.tsx
```

### 4. js 번들 빌드    

```shell
yarn build 
```

> dist/* 빌드된 파일 확인     
```
dist
 L dist
    L index.ex.js
    L index.umd.js 
```

> type을 추론할 수 있는 *.d.ts 파일이 생성되지 않음    

### 5. d.ts 파일 자동 생성을 위한 tsconfig 설정    

> tsconfig.json
```diff
{
  "compilerOptions": {
    "target": "ESNext",
    "useDefineForClassFields": true,
    "lib": ["DOM", "DOM.Iterable", "ESNext"],
    "allowJs": false,
    "skipLibCheck": true,
    "esModuleInterop": false,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "module": "ESNext",
    "moduleResolution": "Node",
    "resolveJsonModule": true,
    "isolatedModules": true,
-    "noEmit": true,
+    "noEmit": false,                   // *.d.ts 파일 생성
    "jsx": "react-jsx",
+    "declaration": true,               // *.d.ts 파일 생성
+    "declarationDir": "dist/@types",   // *.d.ts 파일 생성 위치
+    "emitDeclarationOnly": true        // *.d.ts 파일 생성
  },
  "include": [
-    "src"
+    "src/comp"    // src/comp 하위의 모듈만 컴파일
  ],
  "references": [{ "path": "./tsconfig.node.json"}]
}
```

> typescript 컴파일을 위해 ttypescript 모듈 추가 및 build 스크립트 수정   

> package.json   
```diff
{
  ... 중략
  "scripts": {
    "dev": "vite",
-    "build": "tsc && vite build",
+    "build": "vite build && ttsc -p ./tsconfig.json ",
    "preview": "vite preview"
  }
  "devDependencies": {
  ... 중략
+    "ttypescript": "^1.5.13"
  }
}
```

> dist/* 빌드된 파일 확인
```diff
dist
 L dist
    L index.ex.js
    L index.umd.js
+   L @types
```

### 6. 타입 참조 파일 위치 정보 추가     

> package.json   
```diff
{
  "name": "front-lib-skeleton",
  "private": true,
  "version": "0.0.0",
+  "main": "./dist/index.umd.js",           // 이 라이브러리를 사용하는 프로젝트가 umd 프로젝트인 경우 참조할 번들 파일 위치
+  "module": "./dist/index.es.js",          // 이 라이브러리를 사용하는 프로젝트가 es 프로젝트인 경우 참조할 번들 파일 위치
+  "types": "./dist/@types/index.d.ts",     // 이 라이브러리를 사용하는 프로젝트가 ts 프로젝트인 경우 참조할 타입(d.ts) 파일 위치
  ...중략
}
```


## 테스트를 위한 Storybook 모듈 추가     
> https://storybook.js.org/blog/storybook-for-vite/

### 1. storybook-vite 초기화     

```shell
npx sb init --builder @storybook/builder-vite
```

> 스토리북 실행    
```shell
yarn storybook
```

### 2. View(SimpleCalculator) 테스트를 위한 storybook 컴포넌트 작성    

```
comp
 L stories
    L SimpleCaculator.stoires.tsx
```

> *.stoires.tsx 로 끝나는 파일을 대상으로 storybook이 파일을 스캔함.     
> 관련 설정 (.storybook/main.js)   
```javascript
module.exports = {
    "stories": [
        "../src/**/*.stories.mdx",
        "../src/**/*.stories.@(js|jsx|ts|tsx)"
    ],
    // 중략
}
```

> SimpleCaculator.stories.tsx
```typescript
import React from 'react';
import {ComponentMeta, ComponentStory} from '@storybook/react';
import {SimpleCalculator} from "../comp";

// storybook 컴포넌트에 대한 명세
export default {
  title: 'View/SimpleCalcuator',
  component: SimpleCalculator,
} as ComponentMeta<typeof SimpleCalculator>;

const Template: ComponentStory<typeof SimpleCalculator> = (args) => <SimpleCalculator {...args} />;

export const Primary = Template.bind({});
// 임의의 props 값 전달
Primary.args = {
  x: 10,
  y: 5
};
```

### 3. storybook을 실행하여 확인   
```shell
yarn storybook
```

> 화면 

![img.png](/assets/2022-08-21-front-lib-skeleton/sb_simplecalculator.png)


## 배포 (TODO)  



