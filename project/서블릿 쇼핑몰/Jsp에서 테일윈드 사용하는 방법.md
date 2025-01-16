# Jsp에서 tailwind 사용하는 방법


## CLI로 사용하는 방법

### 1. 테일윈드 설치

#### 1.1 npm 명령어를 통해서 설치하는 방법
```bash
npm init -y 
npm install -D tailwindcss
npx tailwindcss init
```

#### 1.2 npm 명령어 없이 실행하기
```bash
curl -sLO https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-macos-x64
chmod +x tailwindcss-macos-x64
mv tailwindcss-macos-x64 tailwindcss
./tailwindcss init
```

### 2. input.css 만들기
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```
- 이 파일에 위 코드를 넣는다.
- 위 파일을 통해 실제로 사용되는 css 파일로 변환시킨다.
- 즉 배포 환경에서는 위 css 파일은 필요없다. 단지 jsp파일에서 사용되는 css를 만들어주는 역할을 한다.

### 3. tailwind.config.js 만들기
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
- content의 경로에서 테일윈드 css를 사용할 수 있게 해주는 js 파일이다.

### 4. Tailwind 빌드하기
```bash
# npm으로 설치했다면 이 명령어
 npx tailwindcss -c tailwind.config.js -i ./input.css -o ./src/main/webapp/static/css/style.css --watch

# npm없이 설치했다면 이 명령어
./tailwindcss -c tailwind.config.js -i ./input.css -o ./src/main/webapp/static/css/style.css --watch
```
- `./tailwindcss` or `npx tailwindcss`
  - Tailwind CLI 실행 파일을 지정
  - 현재 디렉토리(./)에 있는 tailwindcss 실행 파일을 실행
- `-c tailwind.config.js`
  - Tailwind CSS에 사용할 구성 파일을 지정.
  - 여기서는 tailwind.config.js를 사용해 프로젝트에 맞춘 설정을 적용.
  - 생략하면 Tailwind는 기본값으로 동작하거나 루트 디렉토리의 tailwind.config.js를 자동 탐지.
- `-i ./input.css`
  - 입력 파일 경로를 지정.
  - 여기서는 `./input.css` 파일을 읽어서 Tailwind 규칙이 포함된 최종 CSS 파일을 생성.
  이 파일에는 @tailwind base;, @tailwind components;, @tailwind utilities; 같은 Tailwind 지시어가 포함되어야 함.
- `o ./src/main/webapp/static/css/style.css`
  - 출력 파일 경로를 지정. 
  - Tailwind CSS가 컴파일된 결과를 ./src/main/webapp/static/css/style.css 파일에 저장
- `--watch`
  - 파일 변경을 실시간으로 감지.
  - -i로 지정된 입력 파일이 변경되면, 자동으로 CSS를 재컴파일하고 -o로 지정된 출력 파일에 반영. 

- input.css에 적힌 tailwind 규칙에 따라서 `tailwind.config.js`를 읽어 tailwind css가 적용될 경로(+ 추가적인 설정 포함)를 찾아 목적 경로의 파일에 저장한다. 
- watch 명령어를 통해 css 변경을 자동으로 감지해서 업데이트한다.
- jsp 개발에서 사용할 경우 cmd에서 위 명령어를 실행한 상태로 유지하거나 css를 업데이트할 때마다 실행시켜야한다.


> 인텔리제이에서 자동완성 기능을 사용하고 싶으면 npm 명령어를 사용해야 한다.

인텔리제이 사용할 경우 자동완성하려면 아래 설정을 추가한다.
settings > Languages & Frameworks > Style Sheets > Tailwind CSS
includeLanguage에  "jsp":"html" 추가
## CDN으로 사용하는 방법
