# 유튜브 채널 등록 기능에서 검색과 등록을 분리한 이유: HMAC 기반 등록 토큰 설계

유튜브 채널을 등록하는 기능을 구현하면서 처음에는 단순하게 생각했다.

관리자가 유튜브 채널 핸들을 입력하면 서버에서 YouTube API를 호출하고, 응답받은 채널 정보를 바로 DB에 저장하도록 생각했다.

```
POST /api/admin/youtube-channels
→ YouTube API 호출
→ 채널 정보 조회
→ DB 저장
```

기능만 놓고 보면 하나의 API로 충분히 처리할 수 있다. 하지만 구현을 구체화하면서 고민이 생겼다.

외부 API 조회와 내부 DB 저장은 같은 “등록” 흐름 안에 있지만, 성격이 다르다. YouTube API 호출은 외부 시스템에 의존하는 조회 작업이고, DB 저장은 우리 시스템의 상태를 변경하는 작업이다. 이 둘을 하나의 요청에 묶으면 구현은 단순해질 수 있지만, 실패 지점과 책임이 한 API 안에 섞이게 된다.

그래서 유튜브 채널 등록 기능을 다음과 같이 두 단계로 분리했다.


```
1. 유튜브 채널 검색
   → YouTube API 호출
   → 검색 결과와 registrationToken 반환

2. 유튜브 채널 등록
   → registrationToken 검증
   → DB 중복 검사
   → DB 저장
```

실제 컨트롤러에서도 검색 API와 등록 API는 분리했다. 검색 API는 `forHandle`을 받아 유튜브 채널을 검색하고, 검색된 `channelId`, `forHandle`을 기반으로 `registration token`을 발급한다. 
등록 API는 클라이언트가 보낸 `registration token`을 검증한 뒤, 토큰 안의 값을 사용해 등록을 수행한다.


## 1. 하나의 API로 처리할 수도 있었다

가장 단순한 방식은 검색과 등록을 하나의 API에서 처리하는 것이다.

```
POST /api/admin/youtube-channels

요청:
{
  "forHandle": "@channel",
  "channelName": "...",
  "playlistId": "...",
  "country": "KR"
}
```

서버는 요청을 받으면 YouTube API를 호출해 채널 정보를 가져오고, 그 결과를 DB에 저장한다.

이 방식의 장점은 명확하다.

```
- API가 하나라서 구조가 단순하다.
- 프론트엔드 구현도 상대적으로 쉽다.
- registration token이나 임시 저장소가 필요 없다.
```

하지만 단점도 있었다.

관리자가 채널명, 국가, playlistId 등 등록에 필요한 정보를 모두 입력한 뒤 등록 버튼을 눌렀는데, 그 시점에 YouTube API 호출이 실패하면 등록 전체가 실패한다.

```
관리자 입력 완료
→ 등록 요청
→ YouTube API 장애 또는 quota 초과
→ DB 저장 단계까지 가지 못하고 실패
```

물론 이 실패 원인은 로그로 구분할 수 있다. 하지만 로그는 장애가 발생한 뒤 개발자가 원인을 분석하기 위한 수단이다. 내가 고민한 지점은 단순히 “로그로 구분 가능한가”가 아니라, **API 자체가 어떤 책임을 가져야 하는가**였다.

하나의 등록 API가 다음 책임을 모두 갖게 된다.

```text
- 외부 API 조회
- 외부 API 응답 파싱
- 채널 존재 여부 확인
- 사용자 입력값 검증
- DB 중복 검사
- DB 저장
```

이렇게 되면 등록 요청 하나가 외부 시스템 조회와 내부 상태 변경을 모두 책임지게 된다. 기능이 작을 때는 문제가 없어 보이지만, 장애 대응이나 재시도 전략을 생각하면 책임이 분리되어 있는 편이 더 명확하다고 판단했다.

## 2. 검색과 등록을 분리했다

그래서 검색과 등록을 분리했다.

```text
GET /api/admin/youtube-channels/search?forHandle=@channel
→ YouTube API 호출
→ 검색 결과 반환

POST /api/admin/youtube-channels
→ 검증된 검색 결과 기반으로 DB 저장
```

이렇게 나누면 각 API의 책임이 명확해진다.

```text
검색 API
→ 유튜브 채널이 실제로 존재하는지 확인하고, 관리자에게 후보 정보를 보여준다.

등록 API
→ 검증된 채널 정보를 우리 시스템에 저장한다.
```

이 구조는 관리자 UX에도 더 자연스럽다.

관리자는 핸들을 입력하고 검색 결과를 먼저 확인할 수 있다.

```text
핸들 입력
→ 검색 결과 확인
→ 이 채널이 맞는지 판단
→ 등록
```

유튜브 핸들은 잘못 입력될 수 있고, 관리자가 실제 등록될 채널명과 썸네일을 확인해야 할 수도 있다. 따라서 “입력 즉시 등록”보다 “검색 후 확인, 그리고 등록” 흐름이 더 안전하다고 보았다.


## 3. 그런데 2단계로 나누면 새로운 문제가 생긴다

검색과 등록을 분리하면, 검색 단계에서 확인한 정보를 등록 단계까지 어떻게 전달할지가 문제가 된다.

단순하게 생각하면 검색 API에서 `channelId`를 내려주고, 등록 API에서 다시 받으면 된다.

```text
검색 응답:
{
  "channelId": "UC_A",
  "forHandle": "@aaa",
  "channelName": "A Channel"
}

등록 요청:
{
  "channelId": "UC_A",
  "forHandle": "@aaa",
  "channelName": "A Channel",
  "playlistId": "...",
  "country": "KR"
}
```

하지만 이 방식은 서버가 클라이언트가 보낸 `channelId`를 그대로 신뢰하게 된다.

클라이언트는 요청 값을 얼마든지 바꿀 수 있다.

```text
검색은 A 채널로 수행
→ 등록 요청에서는 B 채널의 channelId로 변경
```

관리자 API라 하더라도, 서버 설계 원칙상 중요한 식별자를 클라이언트 입력에만 의존하는 것은 좋은 구조가 아니라고 판단했다.

검색 단계에서 서버가 확인한 `channelId`와 `forHandle`이 등록 단계에서도 그대로 사용되었는지 검증할 방법이 필요했다.

## 4. 대안 1: 검색 결과를 Redis나 DB에 임시 저장하기

첫 번째 대안은 검색 결과를 서버에 임시 저장하는 것이다.

```text
검색 API
→ YouTube API 호출
→ 검색 결과를 Redis 또는 임시 테이블에 TTL 5분으로 저장
→ searchResultId 반환

등록 API
→ searchResultId로 서버 저장소 조회
→ DB 등록
```

이 방식은 안전하다. 클라이언트가 `channelId`를 직접 조작할 여지가 줄어든다. 서버가 검색 결과를 직접 관리하고, 등록 시에는 `searchResultId`만 받아 서버 저장소에서 원본 검색 결과를 조회하면 된다.

하지만 이 기능에 적용하기에는 다소 무겁다고 판단했다.

```text
- Redis 또는 임시 테이블이 필요하다.
- TTL 관리가 필요하다.
- 검색 결과 저장 실패를 처리해야 한다.
- 만료된 검색 결과에 대한 정책이 필요하다.
- 짧은 시간 필요한 중간 상태를 위해 저장소 운영 포인트가 늘어난다.
```

검색 결과는 장기적으로 보관할 데이터가 아니었다. 등록 단계로 넘어가기 위한 짧은 연결 정보에 가까웠다. 이 정도의 일시적 데이터를 위해 별도의 저장소와 만료 관리 책임을 추가하는 것은 기능 규모 대비 복잡도가 크다고 보았다.

---

## 5. 대안 2: 등록 시 YouTube API를 다시 호출하기

두 번째 대안은 등록 시점에 YouTube API를 다시 호출하는 것이다.

```text
검색 API
→ YouTube API 호출
→ 결과 확인

등록 API
→ forHandle로 YouTube API 재호출
→ channelId 재확인
→ DB 저장
```

이 방식도 안전하다. 등록 시점에 서버가 다시 YouTube API를 호출하므로, 클라이언트가 보낸 `channelId`를 신뢰하지 않아도 된다.

하지만 이 방식은 다른 문제가 있었다.

```text
- YouTube API를 중복 호출한다.
- quota 사용량이 증가한다.
- 등록 시점에도 외부 API 장애에 영향을 받는다.
- 검색 시점의 결과와 등록 시점의 결과가 달라질 수 있다.
- 등록 API가 다시 외부 API 조회 책임을 갖게 된다.
```

물론 구현을 잘하면 YouTube API 호출을 DB 트랜잭션 밖으로 뺄 수 있다. 따라서 “트랜잭션 범위가 무조건 커진다”고 말할 수는 없다.

하지만 등록 API가 다시 외부 API 호출에 의존하게 되는 것은 사실이다. 그러면 등록 실패 원인이 다시 YouTube API 문제인지, quota 문제인지, DB 문제인지 섞이게 된다.

검색과 등록을 분리한 이유 중 하나가 외부 조회와 내부 상태 변경의 책임을 나누기 위함이었는데, 등록 시 YouTube API를 다시 호출하면 그 장점이 약해진다고 판단했다.

---

## 6. 최종 선택: HMAC 기반 Registration Token

그래서 선택한 방식이 HMAC 기반 registration token이다.

검색 단계에서 서버가 확인한 값을 payload로 구성한다.

```text
payload:
{
  "channelId": "...",
  "forHandle": "...",
  "expiresAtEpochSecond": ...
}
```

그리고 이 payload를 HMAC-SHA256으로 서명한 토큰을 클라이언트에 내려준다.

```text
registrationToken = base64url(payload) + "." + signature
```

등록 단계에서는 클라이언트가 보낸 registration token을 검증한다.

```text
1. payload와 signature 분리
2. 서버 secret key로 signature 재계산
3. 클라이언트가 보낸 signature와 비교
4. 만료 시간 검증
5. 검증된 payload의 channelId, forHandle 사용
```

실제 구현에서도 `channelId`, `forHandle`, `expiresAtEpochSecond`를 payload로 만들고, `HmacSHA256`으로 signature를 생성한다. 등록 시에는 동일한 방식으로 expected signature를 다시 계산한 뒤 `MessageDigest.isEqual()`로 비교하고, 이후 토큰 만료 시간도 검증한다. 

---

## 7. HMAC은 무엇인가?

HMAC은 Hash-based Message Authentication Code의 약자다.

쉽게 말하면 서버만 알고 있는 secret key와 메시지를 함께 사용해 서명값을 만드는 방식이다.

개념적으로는 다음과 같이 볼 수 있다.

```text
signature = HMAC-SHA256(secretKey, payload)
```

일반적인 SHA-256 해시는 payload만 있으면 누구나 계산할 수 있다.

```text
SHA-256(payload)
```

따라서 클라이언트가 payload를 바꾸고, 바뀐 payload로 다시 SHA-256 값을 계산해 보내는 것도 가능하다.

하지만 HMAC-SHA256은 secret key가 필요하다.

```text
HMAC-SHA256(secretKey, payload)
```

secret key는 서버만 알고 있으므로, 클라이언트는 payload를 조작하더라도 올바른 signature를 만들 수 없다.

즉, HMAC은 데이터를 숨기기 위한 암호화가 아니라, 데이터가 서버가 만든 이후 변경되지 않았는지 검증하기 위한 서명 방식이다.

```text
암호화
→ 내용을 못 보게 하는 목적

HMAC 서명
→ 내용이 바뀌지 않았는지 검증하는 목적
```

이 기능에서 필요한 것은 `channelId`를 숨기는 것이 아니었다. 중요한 것은 검색 단계에서 서버가 확인한 `channelId`가 등록 단계에서 다른 값으로 바뀌지 않았음을 검증하는 것이었다.

따라서 암호화보다 서명이 더 적합했다.

---

## 8. 왜 HMAC-SHA256인가?

HMAC은 내부 해시 알고리즘으로 여러 가지를 사용할 수 있다.

```text
HmacMD5
HmacSHA1
HmacSHA256
HmacSHA512
```

이 중 `HmacSHA256`을 선택한 이유는 안전성과 표준성 때문이다.

MD5와 SHA-1은 서로 다른 입력이 같은 해시값을 갖는 충돌을 현실적으로 만들 수 있다는 점에서 보안 용도로 권장되지 않는다. 물론 HMAC은 secret key를 함께 사용하기 때문에 단순 해시의 충돌 문제와 완전히 동일하게 보기는 어렵다.

하지만 새로 설계하는 토큰 서명 방식에서 레거시 알고리즘을 선택할 이유는 없었다.

`HmacSHA256`은 Java 표준 `Mac` API에서 바로 사용할 수 있고, 현재 일반적인 서명 및 토큰 검증 용도로 널리 쓰이는 방식이다. 이 기능의 목적은 짧은 registration token의 무결성 검증이므로 SHA-256 기반 HMAC이면 충분하다고 판단했다.

---

## 9. 이 방식의 장점

HMAC 기반 registration token을 사용하면 다음 장점이 있다.

```text
1. 검색 결과를 서버에 임시 저장하지 않아도 된다.
2. 등록 시 YouTube API를 다시 호출하지 않아도 된다.
3. 클라이언트가 channelId를 조작하면 signature 검증에 실패한다.
4. expiresAt을 payload에 넣어 오래된 검색 결과 사용을 제한할 수 있다.
5. 검색과 등록 단계를 stateless하게 연결할 수 있다.
```

즉, 이 방식은 Redis나 임시 테이블 없이도 검색 단계와 등록 단계를 안전하게 연결할 수 있다.

```text
검색 API
→ 외부 API 조회
→ 서버가 확인한 결과를 서명 토큰으로 전달

등록 API
→ 서명 토큰 검증
→ 검증된 channelId로 DB 저장
```

---

## 10. 이 방식의 한계

물론 이 방식에도 주의할 점은 있다.

첫 번째, HMAC은 암호화가 아니다. payload는 Base64Url로 인코딩되어 있을 뿐이므로 클라이언트가 내용을 볼 수 있다.

따라서 token payload에는 민감한 정보를 넣으면 안 된다. 이 기능에서는 `channelId`, `forHandle`, `expiresAtEpochSecond` 정도만 넣었기 때문에 문제가 크지 않다고 보았다.

두 번째, secret key 관리가 중요하다.

secret key가 노출되면 공격자가 유효한 registration token을 직접 만들 수 있다. 따라서 운영 환경에서는 반드시 환경변수나 secret 관리 도구를 통해 주입해야 한다.

세 번째, 중복 등록 방지는 애플리케이션 레벨 검사만으로는 부족하다.

등록 서비스에서 `existsByChannelId()`로 중복 여부를 확인하더라도, 동시에 같은 채널을 등록하는 요청이 들어오면 race condition이 발생할 수 있다.

```text
A 요청: existsByChannelId false
B 요청: existsByChannelId false
A 요청: save 성공
B 요청: save 시도
```

따라서 DB에도 `channel_id`에 대한 unique 제약조건이 필요하고, unique constraint violation을 적절한 예외로 변환하는 처리가 필요하다.

---

## 11. 결론

유튜브 채널 등록 기능은 하나의 API로도 구현할 수 있었다.

하지만 외부 API 조회와 내부 DB 저장은 성격이 다르다. YouTube API 호출은 외부 시스템의 가용성에 영향을 받는 조회 작업이고, DB 저장은 우리 시스템의 상태를 변경하는 작업이다.

그래서 두 작업을 하나의 등록 요청에 모두 묶기보다, 검색과 등록을 분리했다.

검색과 등록을 분리하면 검색 결과를 등록 단계까지 어떻게 안전하게 전달할지가 문제가 된다. Redis나 DB에 임시 저장하는 방법도 있고, 등록 시 YouTube API를 다시 호출하는 방법도 있었다.

하지만 이 기능에서는 별도의 임시 저장소를 두지 않고, 등록 시 외부 API를 다시 호출하지 않으면서도 검색 결과 위변조를 막고 싶었다.

그래서 최종적으로 HMAC-SHA256 기반 registration token을 선택했다.

```text
검색 단계에서 서버가 확인한 channelId, forHandle, 만료 시간을 토큰으로 서명
→ 등록 단계에서 토큰의 서명과 만료 시간 검증
→ 검증된 값으로 DB 저장
```

이 설계의 핵심은 단순히 토큰을 사용했다는 것이 아니다.

외부 API 조회와 내부 상태 변경을 분리하고, 두 단계 사이의 신뢰 문제를 stateless한 서명 토큰으로 해결했다는 점이다.
