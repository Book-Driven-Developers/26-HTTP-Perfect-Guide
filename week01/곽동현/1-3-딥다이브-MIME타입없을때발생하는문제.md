# 1.3 딥다이브 — MIME 타입이 없다면 어떤 문제가 생길까?

> MIME 타입은 단순한 "파일 종류 표시"가 아니다.
> 서버가 클라이언트에게 "이 바이트 덩어리를 어떻게 해석해라"는 **계약서**다.
> 이 계약서가 사라지면 클라이언트는 추측에 의존할 수밖에 없고, 추측은 언제나 보안 구멍이 된다.

---

## 1. MIME 타입이 하는 일 — "해석 명령"

HTTP 응답은 결국 **바이트 스트림**이다.
`<html>...</html>` 도 바이트고, `JFIF...` 로 시작하는 JPEG 이미지도 바이트다.
MIME 타입이 없다면 클라이언트는 이 바이트 덩어리가 무엇인지 알 방법이 없다.

```
HTTP/1.1 200 OK
Content-Length: 12984
                          ← Content-Type 헤더가 없다면?

[바이트 덩어리...]
```

---

## 2. 문제 1 — 브라우저의 "추측 렌더링" (Content Sniffing)

MIME 타입이 없을 때 브라우저는 **콘텐츠 스니핑(Content Sniffing)** 을 시도한다.
바이트 앞부분을 살펴보고 "이게 HTML 같은데?"라고 스스로 판단하는 것이다.

```
바이트 시작이 "<html"  → HTML로 렌더링
바이트 시작이 "JFIF"   → JPEG 이미지로 표시
바이트 시작이 "%PDF"   → PDF로 열기
```

### 추측이 틀릴 때

서버가 텍스트 파일을 내려줬는데 그 텍스트 안에 `<script>` 태그가 포함되어 있다고 가정한다.

```
서버 의도:  이건 사용자가 업로드한 일반 텍스트 파일이야. 그냥 화면에 글자로 보여줘.
브라우저:   Content-Type 없음 → 스니핑 → "이거 HTML 같은데?" → JavaScript 실행!
```

공격자가 `<script>alert('XSS')</script>` 가 담긴 텍스트 파일을 업로드하면,
Content-Type 없이 응답하는 서버는 브라우저가 그것을 HTML로 해석해 스크립트를 실행하게 만든다.
이것이 **MIME 스니핑 기반 XSS 공격**이다.

> **인터넷 익스플로러(IE)** 는 Content-Type이 있어도 내용을 보고 재분류하는 공격적 스니핑 정책 때문에
> 수많은 보안 사고의 원인이 됐다. 이 문제를 막기 위해 등장한 헤더가 바로 `X-Content-Type-Options: nosniff`다.

---

## 3. 문제 2 — API 클라이언트의 역직렬화 실패

브라우저는 그나마 스니핑이라도 한다.
하지만 **REST API를 호출하는 클라이언트 코드**는 스니핑을 하지 않는다.

```java
// Spring의 RestTemplate은 Content-Type을 보고 역직렬화 방법을 결정한다
ResponseEntity<User> response = restTemplate.getForEntity(url, User.class);
```

내부적으로 `RestTemplate`은 다음 흐름으로 동작한다:

```
응답 수신
  → Content-Type 헤더 읽기
  → "application/json" → Jackson으로 역직렬화
  → "application/xml"  → JAXB로 역직렬화
  → 헤더 없음          → HttpMessageConverter 선택 불가 → 예외 발생
```

Content-Type 없이 JSON을 내려보내면:

```
org.springframework.web.client.UnknownContentTypeException:
Could not extract response: no suitable HttpMessageConverter found
for response type [class com.example.User] and content type [application/octet-stream]
```

`application/octet-stream`은 "타입 모름"을 나타내는 MIME 타입의 fallback값이다.
즉, MIME 타입이 없으면 클라이언트 코드는 **JSON인지 XML인지 알 수 없어 역직렬화 자체가 불가능**하다.

---

## 4. 문제 3 — 다운로드 vs 렌더링 판단 실패

브라우저는 MIME 타입을 보고 "화면에 그릴 것인가, 파일로 저장할 것인가"를 결정한다.

```
Content-Type: text/html        → 화면에 렌더링
Content-Type: application/pdf  → PDF 뷰어로 열기
Content-Type: application/octet-stream → 파일로 다운로드
Content-Type 없음              → 브라우저 마음대로
```

예를 들어 PDF를 내려줄 때 Content-Type을 빠뜨리면:
- 어떤 브라우저는 바이트를 분석해 PDF 뷰어로 연다
- 어떤 브라우저는 그냥 파일로 저장한다
- 어떤 브라우저는 깨진 텍스트를 화면에 출력한다

**같은 서버, 같은 응답인데 브라우저마다 다른 결과**가 나온다.
이것이 MIME 타입 없이 멀티 브라우저 지원을 포기하는 것과 같은 이유다.

---

## 5. 문제 4 — 중간 서버(프락시, CDN)의 오동작

프락시나 CDN은 MIME 타입을 보고 **캐싱 전략을 결정**한다.

```
Content-Type: image/jpeg  → 공격적으로 캐시, 오래 유지
Content-Type: text/html   → 짧게 캐시 (자주 바뀌므로)
Content-Type: application/json → 보통 캐시 안 함 (동적 데이터)
Content-Type 없음          → CDN이 기본 전략 적용 (보통 캐시 안 함, 또는 임의 적용)
```

이미지 파일이 MIME 타입 없이 전달되면 CDN이 캐시를 안 해버릴 수 있다.
반대로 API 응답(JSON)이 MIME 타입 없이 전달되면 CDN이 임의로 캐시해버려
**오래된 데이터를 사용자에게 계속 내려보내는** 버그가 생길 수 있다.

---

## 6. 문제 5 — 콘텐츠 협상(Content Negotiation) 붕괴

HTTP는 클라이언트가 원하는 형식을 서버에게 요청하는 **콘텐츠 협상** 메커니즘을 갖고 있다.

```
GET /api/users/1 HTTP/1.1
Accept: application/json          ← 나는 JSON으로 줘
```

```
GET /api/users/1 HTTP/1.1
Accept: application/xml           ← 나는 XML로 줘
```

서버는 요청의 `Accept` 헤더를 보고 응답 포맷을 결정하고, 응답의 `Content-Type`으로 실제 포맷을 알린다.

```
HTTP/1.1 200 OK
Content-Type: application/json    ← 요청대로 JSON으로 줬어
```

**Content-Type 없이 응답하면 클라이언트는 자신이 요청한 형식으로 왔는지조차 확인할 수 없다.**
협상이 성공했는지 실패했는지 알 방법이 없어지는 것이다.

---

## 7. 실제 사례 — `X-Content-Type-Options: nosniff`

위에서 설명한 MIME 스니핑 공격이 실제로 심각한 문제가 되자,
마이크로소프트가 IE 8에서 이 헤더를 도입했다.

```
HTTP/1.1 200 OK
Content-Type: text/plain
X-Content-Type-Options: nosniff   ← 이거 텍스트야. 스니핑하지 마.

[<script>alert('XSS')</script> 가 담긴 텍스트 파일]
```

이 헤더가 있으면 브라우저는 Content-Type을 **그대로 믿고** 스니핑을 하지 않는다.
스크립트가 있어도 "텍스트"로만 보여준다.

Spring Security는 기본적으로 이 헤더를 응답에 포함시킨다:

```java
// Spring Security 기본 설정에서 자동 포함
// SecurityHeadersConfigurer → ContentTypeOptionsHeaderWriter
response.addHeader("X-Content-Type-Options", "nosniff");
```

---

## 8. Java 개발자 관점 — 실수하기 쉬운 케이스

```java
// 잘못된 예 — Content-Type 없이 응답
@GetMapping("/download")
public ResponseEntity<byte[]> download() {
    byte[] data = fileService.read();
    return ResponseEntity.ok(data);  // Content-Type 지정 안 함
}

// 올바른 예
@GetMapping("/download")
public ResponseEntity<byte[]> download() {
    byte[] data = fileService.read();
    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_PDF)
        .body(data);
}
```

또는 `produces` 속성으로 명시:

```java
@GetMapping(value = "/api/users", produces = MediaType.APPLICATION_JSON_VALUE)
public List<User> getUsers() { ... }
```

`produces`를 명시하면 두 가지 효과가 있다:
1. 응답에 `Content-Type: application/json` 이 자동으로 붙는다
2. 클라이언트가 `Accept: application/xml`로 요청하면 **406 Not Acceptable** 로 거절한다 (잘못된 협상을 명확히 차단)

---

## 9. 전체 흐름 정리

```
MIME 타입 없음
  ├─ 브라우저 → 콘텐츠 스니핑 → XSS 공격 가능
  ├─ API 클라이언트 → 역직렬화 실패 → 예외 발생
  ├─ 브라우저 → 렌더링/다운로드 판단 불가 → 브라우저마다 다른 동작
  ├─ 프락시/CDN → 캐싱 전략 오판 → 오래된 데이터 제공 또는 캐시 미적용
  └─ 콘텐츠 협상 → 성공 여부 확인 불가 → 클라이언트-서버 계약 붕괴
```

MIME 타입은 "있으면 좋고 없어도 되는" 정보가 아니다.
**클라이언트가 바이트를 어떻게 해석할지 결정하는 유일한 근거**이며,
이것이 없으면 클라이언트는 추측에 의존하고, 추측은 보안 사고와 버그의 원인이 된다.

---

## 아직 열린 질문 — 다음 탐구의 시작점

> **서버가 `Content-Type: application/json`을 내려줬는데, 실제 바디는 XML이라면 어떻게 될까?**
>
> MIME 타입이 "계약서"라면, 계약 내용과 실제 이행이 다른 경우다.
> 브라우저는 어떻게 반응하고, `RestTemplate`은 어떻게 반응할까?
> 그리고 이런 실수를 방지하는 서버 측 장치는 무엇인가?

*관련 섹션: 1.3 리소스 / 1.3.1 미디어 타입*
