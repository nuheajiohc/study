# 개발자도구만 열면 CSS가 깨지는 이유? DevTools 재정의 문제 해결

개발 중 CSS가 어떻게 적용되는지 확인하기 위해 크롬 **개발자도구(DevTools)**를 열었는데 의도한 CSS가 적용되지 않는 문제가 생겼다.  
처음 페이지를 로드할 때는 제대로 보였지만, 개발자도구를 열기만 하면 이전의 CSS로 적용이 되는 현상이 발생했다.  
사파리나 다른 브라우저에서는 아무런 문제가 없었다.

이 문제를 해결하는 과정에서 **Chrome DevTools**의 **Override 기능**을 알게 되었고, 이를 활용하는 방법까지 정리해 보았다.

## 1. 이상한 점 발견 : "style.css"만 문제?
내 프로젝트에서는 Tailwind를 사용 중이었고, 빌드 후에는 Tailwind CSS는 일반 CSS로 변환된다. (이 부분은 [Tailwind CSS 사용하기(3 버전)](https://nuheajiohc.tistory.com/37) )에 정리해두었다.)  
처음엔 Tailwind의 빌드 문제를 의심했지만, 프로젝트를 실행할 때 Tailwind가 관여하지 않기 떄문에 Tailwind의 문제는 아니라는 것을 바로 깨달았다.  
그러다 **style.css**라는 파일명이 의심되어 다른 이름으로 변경하여 실행했더니 문제가 해결됐다.  
문제는 해결됐지만 왜 style.css라는 이름에서만 이런 일이 생기는 지 궁금했다.

## 2. 진짜 원인 발견 : "Devtools에 의해 재정의되었습니다" 메세지
원인을 찾기 위해 개발자도구의 네트워크(Network) 탭을 확인했다. 그 과정에서 style.css 파일이 로드될 때 보라색 점이 찍혀 있는 것을 발견했다.   
![devtools override](img/devtools%20override.png)  
파일에 마우스를 가져다 대니 "**DevTools에 의해 재정의되었습니다**"라는 메세지가 나타났다.  

## 3. Override에서 style.css 발견 > 해결
개발자도구의 소스탭에서 재정의(override)메뉴를 확인해 보니 **style.css**파일이 등록되어 있었다.  
이로 인해 DevTools가 브라우저가 요청한 최신 CSS가 아니라, 로컬에 저장된 예전 버전의 CSS를 불러오고 있었던 것이었다. 
즉, CSS를 요청할 때 재정의된 위치의 CSS로 연결이 되어서 나타났던 현상이었다.  
그래서 재정의메뉴에 있는 style.css를 삭제하고 나니 완전히 해결되었다.

> 이건 사용자가 따로 설정하지 않는 한 재정의되지 않는다.  
아마도 전에 테일윈드를 빌드하지 않은 채로 CSS가 적용되지 않는 원인을 찾다가 실수로 재정의를 해버린 것 같다.

## 4.재정의(Override)는 언제 사용할까?
Override 기능은 브라우저에서 로컬 파일을 저장하고, 이를 우선 적용하도록 만드는 기능이다.  
그리고 이 기능은 개발자도구를 열 때만 적용이 되기 때문에 개발을 할 때 사용할 수 있는 기능이라는 것을 알 수 있다.  
그래서 서버를 수정하거나 배포하지 않고, 로컬에서 수정한 css/js를 저장해두고 반복적으로 테스트하고 싶을 때 사용하면 유용할 것 같다.  

> 나는 백엔드를 공부하고 있지만, 언젠가 UI 테스트나 프론트엔드 디버깅을 할 때 이 기능을 유용하게 사용할 수 있을 것 같다.

## 참고
[크롬 devtools 공식 문서](https://developer.chrome.com/blog/new-in-devtools-117?hl=ko)
