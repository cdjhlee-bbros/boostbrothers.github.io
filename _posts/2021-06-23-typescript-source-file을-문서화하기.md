---
layout: post
title:  "typescript source file을 문서화하기"
date:   2021-06-23 15:00:00
author: 고윤호
categories: Technology
tags:   typescript, documentation, package
cover:  "/assets/2021-06-23/documentation.jpg"
---

# 목차

- [개요](#개요)
  * [Motivation](#motivation)
- [원리](#원리)
  * [typescript 패키지를 이용](#typescript-패키지를-이용)
    + [Typescript AST](#typescript-ast)
    + [AST 하위 그룹](#ast-하위-그룹)
  * [type 파싱하기](#type-파싱하기)
  * [Handlebars.js를 이용](#handlebarsjs를-이용)
    + [예상 질문: Handlerbars를 사용한 이유?](#예상-질문-handlerbars를-사용한-이유)
- [Demo](#demo)
  * [Install](#install)
  * [Usage](#usage)
  * [Example Source File](#example-source-file)
  * [Example Markdown File](#example-markdown-file)
- [마무리](#마무리)
  * [라이브러리를 만들면서 느낀점](#라이브러리를-만들면서-느낀점)
  * [이후 개선하고 싶은 점](#이후-개선하고-싶은-점)
  * [개인적인 소감](#개인적인-소감)
- [Author](#author)

# 개요

## Motivation

비브로스 백엔드 팀은 서비스 운영을 위해서 많은 mongoose 스키마와 API 인터페이스를 관리하고 있습니다. 

현재까지 백엔드 팀에서 사용하고 있는 Mongoose의 스키마 정의는 Confluence 문서에 사람이 직접 작성, 수정하는 형태라 누락되거나 오기입 될 수 있었습니다. 때문에 스키마를 정의하면 Mongoose Schema와 Interface, Confluence 문서 세가지를 생성, 수정하기 때문에 관리 포인트도 늘어나고 정보가 상이할 경우 대조, 추적해야하는 번거로움도 있었습니다.

이를 해결하기 위해서 비브로스에서는 Typescript Interface를 분석하여 markdown 파일로 내보내주는 라이브러리를 작성하였고 이제 여러분들에게 소개해드립니다. 👏

# 원리

## typescript 패키지를 이용

typescript를 이용한다는게 무슨 의미일까요? typescript는 그 자체로 TS compiler 역할을 수행하기도 하지만 휼륭한 TS Source Parser이기도 합니다. 종종 typescript 패키지를 이용하시면서 내부 패키지를 열어보시면 `const ts = require('typescript')` 이런 코드가 있는 것을 보실 수 있으실텐데요. 별도 외부 라이브러리 없이 typescript를 분석하거나 코드를 수정하실 수 있습니다.

### Typescript AST

`typescript`를 sourcefile을 분석하면 Typescript AST를 얻을 수 있습니다.

> AST란? [Abstract Syntax Tree](https://ko.wikipedia.org/wiki/%EC%B6%94%EC%83%81_%EA%B5%AC%EB%AC%B8_%ED%8A%B8%EB%A6%AC). 프로그래밍 언어로 작성된 소스코드를 Node의 Tree 구조로 표현한 객체라고 생각하시면 됩니다. 자세한 내용은 링크를 살펴보세요.

typescript답게 AST 객체도 정형화된 객체들이 이미 `typescript` 안에 각 type별로 정의되어 있습니다.

파일 경로를 통해 바로 분석된 AST를 얻는 방법은 찾지 못했지만 `createSourceFile` 함수를 통해 파일 경로에서 읽어온 파일 내용을 넣어주면 분석된 AST를 얻을 수 있습니다.

```tsx
import * as ts from 'typescript';
import * as path from 'path';

const sourceFilePath = '...' // .ts 타입스크립트 소스파일 경로
const file = ts.sys.readFile(sourceFilePath); // source file 내용 불러오기
const {name} = path.parse(sourceFilePath); // source file에서 파일 이름만 parse
const node = ts.createSourceFile(name, file, ts.ScriptTarget.Latest); // AST 생성
```

AST 객체 안에는 다양한 Node 개념들이 있습니다. 여기서 일일이 다 설명 드릴 수 없을 정도로 다양한 Node 들이 있으며 그 하위에는 Type Node, Token 등 개념들이 등장합니다. 이에 대한 자세한 문서를 찾을 수 없어서 작업을 하며 개인적으로 추측한 정보를 정리하면 아래와 같습니다.

- Node는 Syntax 구문의 집합(으로 추측).
- Token은 파일을 구성하는 데이터의 최소 단위(로 추측)
    - EOF
    - questionMark(?)
    - 등등

Tip! Typescript AST에 대해서 online으로 viewer를 제공하는 [홈페이지](https://ts-ast-viewer.com/#)를 참고하면 유용합니다.

### AST 하위 그룹

- SourceFile(최상위 소스파일 정보)
    - SyntaxList
        - ~~Declaration(Node)

            `importDeclaration`, `EnumDeclaration`, `InterfaceDeclaration`, `TypeAliasDeclaration` 등

        - PropertySignature(Node)

            `InterfaceDeclaration`, `TypeAliasDeclaration` 등 field name과 type을 가지는 property

        - EnumMember(Node)

            `EnumDeclaration` 내부에 존재

각 노드별 재귀적으로 갖는 Node 종류

- JSDocComment
- SyntaxList
- Identifier
- 각종 Token들
- 등등

## type 파싱하기

제가 이번에 작업하면서 필요했던건 크게 `interface`, `enum`, `type` 이렇게 세가지입니다. [AST 하위 그룹](https://www.notion.so/typescript-source-file-ce7b6d78758f42ec9ae2692b0d535051)에서도 얘기했던 ~~Declaration 하위에 있는 `InterfaceDeclaration`, `EnumDeclaration`, `TypeAliasDeclaration` 의 Declaration을 각 name, jsdoc, properties 별로 분류하여 파싱하고 property signature members는 name, jsdoc(description, default), questionMark(is required)로 분류하여 테이블을 만들었습니다.

## [Handlebars.js](https://handlebarsjs.com/)를 이용

개발 초기에는 내부 Schema 라이브러리에만 한정적으로 적용되는 템플릿만 적용하면 되었습니다. 그래서 ES6의 Template Literals로 작성하여 String 응답 형태로 최종 조합물을 내보내는 형태였습니다.

그러나 공통 라이브러리로 분리하여서 public 패키지로 분리하면 좋겠다는 의견을 반영하면서 다양한 케이스에 적용할 수 있는 템플릿 라이브러리를 이용하려고 Handlebars.js를 도입하였습니다.

### 예상 질문: Handlebars를 사용한 이유?

![handlebars image](/assets/2021-06-23/handlebars1.png)

![handlebars image 2](/assets/2021-06-23/handlebars2.png)

- 근소한 다운로드 수 차이
- x3 stars, likes on slant(~~그리고 4배차이의 hate~~)
- [비교 블로그 글 참고](https://velog.io/@parkoon/%EC%8B%A4%EB%AC%B4%EC%97%90%EC%84%9C-Handlebars-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0-feat-express#handlebars%EC%9D%98-%EC%9E%A5-%EB%8B%A8%EC%A0%90)

# Demo

복잡한 내용을 구구절절 설명 드리는 것보다 어떤 데이터가 어떤 식으로 보이는지 빨리 보여드리는게 좋겠죠? 😉

- Repository : [@boostbrothers/ts.md](https://github.com/boostbrothers/ts.md)
- [Example files](https://github.com/boostbrothers/ts.md/tree/master/example)

## Install

```bash
$npm i -D @boostbrothers/ts.md # local
# 실패 시: package를 찾을 수 없을 경우
$npm i -D --registry https://npm.pkg.github.com/ @boostbrothers/ts.md
```

package를 찾을 수 없을 경우 registry에 `[https://npm.pkg.github.com](https://npm.pkg.github.com)`을 [추가](https://docs.github.com/en/packages/learn-github-packages/installing-a-package)해주세요.

## Usage

```bash
$npx tsmd
```

## Example Source File

```tsx
/**
 * Test Literal Types
 */
type LiteralTypes = 'Test1' | 'Test2';

/**
 * Test TypeLiteral Type
 */
type TypeLiteralType = {
	f1: string;
	f2: Partial<TypeLiteralType>;
}

/**
 * Test Interface.
 * 
 * - markdown bullet list 1
 * - markdown bullet list 2
 * 
 * @template Gen String Generic Parameter
 */
interface Interface<Gen extends string = LiteralTypes> {
	/** first field */
	field1: number;
	/**
	 * second field
	 * @pattern [0-9A-Za-z]{6, 16}
	 */
	field2: string;
	field3: null;
	field4?: unknown;
	/**
	 * 5th field
	 * @deprecated don't use any type
	 */
	field5?: any;
	field6?: LiteralTypes;
	/**
	 * @min 0
	 * @max 100
	 */
	field7: number[];
	field8: {
		field81: 'field1' | 'field2';
	};
	field9?: Array<LiteralTypes>;
	field10: Gen;
}
```

## Example Markdown File

~~~markdown
# test.interface

## LiteralTypes

- type: `type`

Test Literal Types

```typescript
Test1 | Test2
```

## TypeLiteralType

- type: `type`

Test TypeLiteral Type

### Properties

이름 | 타입 | 필수 | 기본값 | 설명
---|---|:---:|---|--
f1| String|  ✅ | | 
f2| [`Partial`](#Partial)\<[`TypeLiteralType`](#TypeLiteralType)\>|  ✅ | | 

## Interface

- type: `interface`

Test Interface.

- markdown bullet list 1
- markdown bullet list 2

### Tags

- `@template`: String Generic Parameter
### Generics

이름 | extends of | default
--- | --- | ---
Gen| String| [`LiteralTypes`](#LiteralTypes)
### Properties

이름 | 타입 | 필수 | 기본값 | 설명
---|---|:---:|---|---
field1| Number|  ✅ | | first field
field2| String|  ✅ | | second field<br />`@pattern: [0-9A-Za-z]{6, 16}`
field3| Null|  ✅ | | 
field4| Unknown| | | 
~~field5~~| Any| | | 5th field<br />`@deprecated: don't use any type`
field6| [`LiteralTypes`](#LiteralTypes)| | | 
field7| Array\<Number\>|  ✅ | | <br />`@min: 0`<br />`@max: 100`
field8| Object|  ✅ | | 
field8.field81| field1 \| field2|  ✅ | | 
field9| [`Array`](#Array)\<[`LiteralTypes`](#LiteralTypes)\>| | | 
field10| [`Gen`](#Gen)|  ✅ | |
~~~

이상 내용이 길어지는 것을 방지하기 위하여 사용법에 관련된 부분은 [ts.md의 README.md](https://github.com/boostbrothers/ts.md) 파일을 읽어주시기 바랍니다.

# 마무리

## 라이브러리를 만들면서 느낀점

첫째로, 프로젝트를 진행하면서 어려웠던 점은 typescript 모듈을 어떻게 써야 하는지 나와 있는 문서가 전혀 없었다는 점입니다. 공식 문서가 없다 보니 어떻게 찾아봐야 할지도 막막하고 `typescript`라는 키워드로 검색을 하다보니 검색 결과도 내가 필요한 키워드가 검색되지 않았습니다.

국내외 블로그들을 여럿 검색해보면서 어떻게 시작해야하는지 초기 시작 부분을 제외하고는 도움을 받기 어려운 점이 있었습니다.

둘째로, Handlebars.js를 쓰면서 디버깅하는데 어려움이 있었습니다. 그냥 typescript 코드를 분석하는데는 어려움이 없었지만 handlebars를 쓰면서는 템플릿에 어느 부분이 틀렸는지 어떤 내용이 잘못 됐는지 알아차리기가 쉽지 않았습니다.

## 이후 개선하고 싶은 점

현재는 한 파일안에 분석한 모든 파일을 다 밀어넣다보니 큰 프로젝트의 경우 한 파일이 길어지는 부분이 있습니다. 이 점을 `.tsmdrc.json` 파일을 여럿으로 쪼개 만들 수 있는 방법이 있지만 원초적으로 이런 부분을 개선하기 위해서 파일별로 출력하는 기능과 파일에서 TOC(목차)를 만들어주는 기능을 추가하면 보기 좀 더 좋을 것이라는 생각이 듭니다.

반면에 저희가 사용하고 있는 시스템에 맞춰 작업을 진행하다보니 아직 지원하지 않는 기능들이 있는데 예를 들어 `class` 같은 몇가지 문법을 아직 지원하지 않습니다. 혹시 이슈로 누군가 등록해주시거나 PR을 올려주신다면 요청한 기능들에 대하여 추가할 의지는 있습니다.

## 개인적인 소감

이전에도 typescript의 type들에 대하여서 굉장히 하드하게 사용하기도하고 여러 오픈소스 프로젝트에 참여하기도 하면서 typescript를 잘 써오고 있었는데요. typescript의 AST를 분석 해보기도 하고 typescript 소스파일을 바탕으로 결과물을 내는 프로젝트를 진행하다보니 조금은 다른 각도에서 typescript의 매력을 느낀 시간이었습니다.

---

# Author

## [고윤호](https://yoonhogo.github.io/blog)

- [비브로스](https://bbros.co.kr/) 백엔드팀 [똑닥](https://www.ddocdoc.com/) 서버 개발자
- Node.js Typescript 개발자
