# JSP에서 Tailwind CSS 사용하기

## 1. CLI로 사용하는 방법

### 1.1 Tailwind 설치

#### 1.1.1 npm을 사용한 설치

```bash
npm init -y 
npm install -D tailwindcss
npx tailwindcss init
```

#### 1.1.2 npm 없이 실행하기 (맥북용)

```bash
curl -sLO https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-macos-x64
chmod +x tailwindcss-macos-x64
mv tailwindcss-macos-x64 tailwindcss
./tailwindcss init
```

### 1.2 input.css 생성

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

- 이 파일을 통해 실제 CSS 파일을 생성한다.
- 단지 jsp파일에서 사용되는 css를 만들어주는 역할을 한다. 즉 배포 환경에서는 위 css 파일은 필요없다.
- 파일의 위치는 원하는 위치에 두면 되고, 빌드할 때 -i 옵션에서 정확한 경로만 지정해주면 된다.

### 1.3 Tailwind 설정 파일 (tailwind.config.js) 생성

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./src/main/webapp/WEB-INF/views/**/*.jsp"],
  theme: {
    extend: {
      fontFamily: {
        montserratSubrayada: ['"Montserrat Subrayada"', 'serif'],
        montserratUnderline: ['"Montserrat Underline"', 'serif'],
        notoSansKR: ['"Noto Sans KR"', 'sans-serif'],
      },
    },
  },
  plugins: [],
}
```

- Tailwind CSS를 사용하여 css를 만들 경로와 추가 설정들을 담당
- `tailwind.config.js`는 개발 단계에서만 필요하며, 배포 환경에서는 필요하지 않다.

### 1.4 Tailwind 빌드하기

```bash
# npm을 사용한 빌드
npx tailwindcss -c tailwind.config.js -i ./input.css -o ./src/main/webapp/static/css/style.css --watch

# npm 없이 실행하는 경우
./tailwindcss -c tailwind.config.js -i ./input.css -o ./src/main/webapp/static/css/style.css --watch
```

#### 명령어 설명

- `./tailwindcss` 또는 `npx tailwindcss`
  - Tailwind CLI 실행.
- `-c tailwind.config.js`
  - Tailwind 설정 파일을 지정.
- `-i ./input.css`
  - 입력 파일 (Tailwind 규칙을 포함한 CSS 파일) 지정.
  - 이 파일에는 `@tailwind base;`, `@tailwind components;`, `@tailwind utilities;` 같은 Tailwind 지시어가 포함되어야 함.
- `-o ./src/main/webapp/static/css/style.css`
  - 출력 CSS 파일 저장 경로 지정.
  - 실제 배포 환경에서 포함되어야 하는 css파일
- `--watch`
  - 파일 변경 감지 후 자동 업데이트.
  - -i 로 지정된 입력 파일이 변경되면, 자동으로 CSS를 재컴파일하고 -o로 지정된 출력 파일에 반영.

#### 빌드 과정 설명

- `input.css`의 Tailwind 규칙을 기준으로 `tailwind.config.js`에서 적용 경로와 추가 설정을 읽어 최종 CSS 파일을 생성.
- `--watch` 옵션을 사용하면, 파일 변경 시 자동으로 업데이트됨.
- JSP 프로젝트에서 사용할 경우, 명령어를 실행한 상태를 유지하거나 CSS 업데이트 시마다 명령어를 실행해야 함.

### 1.5 IntelliJ에서 자동완성 설정

> IntelliJ에서 Tailwind 자동완성을 사용하려면 npm 방식으로 설치해야 함.

#### 설정 방법

1. `Settings` → `Languages & Frameworks` → `Style Sheets` → `Tailwind CSS`
2. `includeLanguages`에 다음을 추가:
   ```json
   { "jsp": "html" }
   ```



## 2. CDN으로 사용하는 방법

Tailwind CSS를 별도의 설치 없이 빠르게 적용하려면 CDN을 사용할 수 있음.

### 2.1 CDN 링크 추가

JSP 파일의 `<head>` 태그 안에 아래 링크 추가:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body>
  <h1 class="text-3xl font-bold underline">
    Hello world!
  </h1>
</body>
</html>
```

### 2.2 사용 예시

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Tailwind 적용 테스트</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-100 text-center py-10">
    <h1 class="text-3xl font-bold text-blue-500">Hello, Tailwind!</h1>
    <p class="text-gray-700">CDN 방식으로 Tailwind CSS 적용하기</p>
</body>
</html>
```

### 2.3 CDN 방식의 장점과 단점

| 장점          | 단점                                             |
| ----------- | ---------------------------------------------- |
| 간단한 설정      | 일반적으로 이미 Tailwind에 정의된 스타일만 사용 가능 |
| 빠른 적용 가능    | 사용하지 않는 CSS도 포함됨                               |
| 빌드 과정 필요 없음 | 오프라인 환경에서는 사용 불가                               |

CDN 방식은 빠르게 적용할 때 유용하지만, 커스텀 설정이 필요한 경우 CLI 방식이 더 적합함.

