# HTTP 메시지

## 1. 메시지 문법

HTTP 메시지는 **요청 메시지**와 **응답 메시지** 두 종류가 있으며, 각각 다음 구성요소로 이루어진다.

```
요청 메시지:
<메서드> <요청 URL> <버전>
<헤더들>
<빈 줄 (CRLF)>
<엔터티 본문>

응답 메시지:
<버전> <상태코드> <사유 구절>
<헤더들>
<빈 줄 (CRLF)>
<엔터티 본문>
```

### 구성요소 설명

| 구성요소 | 설명 |
|---|---|
| **메서드 (Method)** | 클라이언트가 서버에게 요청하는 동작. `GET`, `POST`, `PUT` 등. 요청 메시지에만 존재 |
| **요청 URL** | 요청 대상 리소스의 위치. 전체 URL 또는 절대 경로 형태 |
| **버전 (Version)** | 사용하는 HTTP 버전. 예: `HTTP/1.1` |
| **상태코드 (Status Code)** | 요청 처리 결과를 나타내는 3자리 숫자. 응답 메시지에만 존재 |
| **사유 구절 (Reason Phrase)** | 상태코드를 사람이 읽기 쉽게 설명한 짧은 문자열. 예: `OK`, `Not Found`. 응답 메시지에만 존재 |
| **헤더 (Headers)** | `이름: 값` 형태의 메타데이터. 요청/응답 모두 존재 |
| **엔터티 본문 (Entity Body)** | 실제 전송하는 데이터. 없을 수도 있음 (예: GET 요청) |

---

## 2. CRLF란?

HTTP 메시지에서 **헤더와 본문 사이의 빈 줄**을 흔히 CRLF라고 부른다.

- **CR (Carriage Return, `\r`, 0x0D)**: 타자기에서 커서를 줄의 맨 앞으로 이동시키는 동작에서 유래
- **LF (Line Feed, `\n`, 0x0A)**: 커서를 다음 줄로 이동시키는 동작

HTTP 명세(RFC 7230)는 줄 끝을 반드시 **CRLF(`\r\n`)** 로 구분하도록 정의한다. 헤더의 각 줄은 `\r\n`으로 끝나고, 헤더 블록이 완전히 끝났음을 나타내는 "빈 줄"은 `\r\n\r\n` (두 번 연속)으로 표현된다.

```
HTTP/1.1 200 OK\r\n
Content-Type: text/html\r\n
Content-Length: 1024\r\n
\r\n          <-- 이 빈 줄이 CRLF (헤더 종료 신호)
<body>...
```

유닉스 계열은 `\n`만 사용하지만, HTTP는 이식성을 위해 명시적으로 `\r\n`을 요구한다. 다만 현실에서는 `\n`만으로도 대부분 동작한다.

---

## 3. HTTP 메서드

### 메서드 종류

| 메서드 | 설명 | 안전(Safe) | 멱등(Idempotent) |
|---|---|---|---|
| **GET** | 리소스 조회 | O | O |
| **POST** | 데이터 제출, 리소스 생성 | X | X |
| **PUT** | 리소스 전체 교체/생성 | X | O |
| **DELETE** | 리소스 삭제 | X | O |
| **HEAD** | 응답 헤더만 조회 (본문 없음) | O | O |
| **OPTIONS** | 서버가 지원하는 메서드 조회 | O | O |
| **TRACE** | 요청 루프백 테스트 (디버깅용) | O | O |
| **CONNECT** | 프록시를 통한 터널 연결 (SSL) | X | X |
| **PATCH** | 리소스 일부 수정 | X | X |

> **Safe**: 리소스를 변경하지 않는 메서드  
> **Idempotent**: 동일한 요청을 여러 번 보내도 결과가 같은 메서드

### HTTP 버전별 메서드 확장

| 버전 | 추가된 메서드 | 비고 |
|---|---|---|
| HTTP/0.9 | GET | 단순 문서 전송 전용 |
| HTTP/1.0 | GET, POST, HEAD | RFC 1945 |
| HTTP/1.1 | PUT, DELETE, OPTIONS, TRACE, CONNECT | RFC 2616 (1999) |
| HTTP/1.1 확장 | **PATCH** | RFC 5789 (2010), 별도 RFC로 추가 |
| HTTP/2.0 | 새 메서드 없음 | 전송 계층 최적화에 집중 (멀티플렉싱, 헤더 압축) |
| HTTP/3.0 | 새 메서드 없음 | QUIC(UDP 기반) 전송 계층 교체, 의미론(semantics)은 HTTP/1.1과 동일 |

HTTP/2와 HTTP/3는 **전송 프로토콜 레이어**를 개선한 것으로, 애플리케이션 레이어의 메서드/상태코드 의미는 RFC 9110(HTTP Semantics, 2022)으로 통합·관리된다.

---

## 4. HTTP 상태코드

상태코드는 3자리 숫자로, 첫 번째 자리가 응답의 분류를 나타낸다.

### 분류 범위

| 범위 | 분류 | 의미 |
|---|---|---|
| 100–199 | 정보 (Informational) | 요청 수신, 처리 진행 중 |
| 200–299 | 성공 (Success) | 요청 정상 처리 |
| 300–399 | 리다이렉션 (Redirection) | 요청 완료를 위해 추가 동작 필요 |
| 400–499 | 클라이언트 오류 (Client Error) | 요청 문법 오류 또는 처리 불가 |
| 500–599 | 서버 오류 (Server Error) | 서버가 요청 처리 실패 |

### 1xx — 정보

| 코드 | 사유 구절 | 의미 |
|---|---|---|
| 100 | Continue | 요청의 첫 부분을 받았으니 계속 보내도 됨 |
| 101 | Switching Protocols | 서버가 프로토콜 전환을 수락함 (예: WebSocket 업그레이드) |
| 102 | Processing *(WebDAV)* | 서버가 요청을 처리 중이나 아직 응답이 없음 |

### 2xx — 성공

| 코드 | 사유 구절 | 의미 |
|---|---|---|
| 200 | OK | 요청 성공 |
| 201 | Created | 요청 성공 및 새 리소스 생성됨 |
| 202 | Accepted | 요청은 수신했으나 처리가 완료되지 않음 |
| 204 | No Content | 요청 성공, 반환할 본문 없음 |
| 206 | Partial Content | 범위 요청(Range)의 일부 성공 |

### 3xx — 리다이렉션

| 코드 | 사유 구절 | 의미 |
|---|---|---|
| 301 | Moved Permanently | 리소스가 영구 이동. 이후 요청은 새 URL 사용 |
| 302 | Found | 리소스가 임시 이동. 원래 URL 유지 |
| 303 | See Other | 다른 URL을 GET으로 조회하라 |
| 304 | Not Modified | 캐시된 리소스가 최신 상태. 본문 없이 헤더만 반환 |
| 307 | Temporary Redirect | 메서드를 유지한 채 임시 이동 |
| 308 | Permanent Redirect | 메서드를 유지한 채 영구 이동 *(RFC 7538, 2015)* |

### 4xx — 클라이언트 오류

| 코드 | 사유 구절 | 의미 |
|---|---|---|
| 400 | Bad Request | 잘못된 요청 문법 |
| 401 | Unauthorized | 인증 필요 (미인증 상태) |
| 403 | Forbidden | 인증했지만 접근 권한 없음 |
| 404 | Not Found | 리소스를 찾을 수 없음 |
| 405 | Method Not Allowed | 해당 메서드 미지원 |
| 409 | Conflict | 리소스 충돌 (예: 중복 생성 시도) |
| 410 | Gone | 리소스가 영구 삭제됨 |
| 422 | Unprocessable Entity | 문법은 올바르나 의미상 처리 불가 *(WebDAV / RFC 4918)* |
| 429 | Too Many Requests | 클라이언트 요청 횟수 초과 *(RFC 6585, 2012)* |
| 451 | Unavailable For Legal Reasons | 법적 이유로 리소스 제공 불가 *(RFC 7725, 2016)* |

### 5xx — 서버 오류

| 코드 | 사유 구절 | 의미 |
|---|---|---|
| 500 | Internal Server Error | 서버 내부 오류 |
| 501 | Not Implemented | 서버가 해당 기능을 지원하지 않음 |
| 502 | Bad Gateway | 게이트웨이/프록시가 잘못된 응답 수신 |
| 503 | Service Unavailable | 서버 일시 중단 또는 과부하 |
| 504 | Gateway Timeout | 게이트웨이/프록시 응답 시간 초과 |

### 책 이후 추가된 주요 상태코드

| 코드 | RFC | 연도 | 설명 |
|---|---|---|---|
| 308 Permanent Redirect | RFC 7538 | 2015 | 301의 메서드 유지 버전 |
| 421 Misdirected Request | RFC 7540 | 2015 | HTTP/2 연결이 잘못된 서버로 전달됨 |
| 422 Unprocessable Content | RFC 4918 | 2007 | 의미적으로 처리 불가능한 요청 |
| 425 Too Early | RFC 8470 | 2018 | TLS 1.3 Early Data 재전송 공격 방지 |
| 429 Too Many Requests | RFC 6585 | 2012 | 요청 속도 제한 초과 |
| 451 Unavailable For Legal Reasons | RFC 7725 | 2016 | 법적 차단 (소설 『화씨 451』에서 유래) |

### 302 vs 303 vs 307 혼선

#### 배경: HTTP/1.0의 302 문제

HTTP/1.0에서 302는 "Found (임시 이동)"을 의미했다. 명세상으로는 리다이렉트 시 **원래 메서드를 유지**해야 했다. 그런데 브라우저들이 실용적인 이유로 `POST` 요청 후 302 응답을 받으면 **GET으로 바꿔서 재요청**하는 동작을 비공식적으로 구현했고, 이것이 사실상 표준이 되어버렸다.

```
POST /submit HTTP/1.0      →  302 Found (Location: /result)
                           →  GET /result    ← 브라우저가 임의로 메서드 변경
```

이 동작은 명세 위반이었지만 너무 많이 퍼져 되돌릴 수 없었다.

#### HTTP/1.1에서의 정리

HTTP/1.1(RFC 2616)은 혼선을 해소하기 위해 두 개의 코드를 새로 정의했다:

| 코드 | 이름 | 동작 | 사용 시점 |
|---|---|---|---|
| **302** Found | 임시 이동 | 메서드 변경 여부가 **불명확** (레거시 호환) | 사용 자제 |
| **303** See Other | 다른 곳 조회 | **항상 GET**으로 재요청 | POST 처리 후 결과 페이지로 이동 시 |
| **307** Temporary Redirect | 임시 리다이렉트 | **메서드 유지** | POST → POST 유지 등, 메서드를 바꾸면 안 될 때 |

#### HTTP/2, HTTP/3에서의 입장

HTTP/2와 HTTP/3는 상태코드의 **의미를 변경하지 않는다**. 다만 영구 리다이렉트 계열에 308이 추가되어 대칭성이 완성됐다:

| 임시 | 영구 | 메서드 변경 여부 |
|---|---|---|
| 302 Found | 301 Moved Permanently | 불명확 (레거시) |
| 303 See Other | — | 항상 GET |
| 307 Temporary Redirect | **308 Permanent Redirect** | 메서드 유지 |

```
메서드를 GET으로 바꾸고 싶다  →  303 (임시)
메서드를 유지하고 싶다       →  307 (임시) / 308 (영구)
302, 301은 레거시 호환용
```

---

## 5. HTTP 헤더

헤더는 역할에 따라 **일반 헤더**, **요청 헤더**, **응답 헤더**, **엔터티 헤더**로 분류된다.

### 헤더 분류

| 분류 | 적용 대상 | 설명 |
|---|---|---|
| **일반 헤더 (General)** | 요청 + 응답 양쪽 | 메시지 자체에 관한 정보 |
| **요청 헤더 (Request)** | 요청 메시지 | 클라이언트 정보, 선호 형식 등 |
| **응답 헤더 (Response)** | 응답 메시지 | 서버 정보, 응답 조건 등 |
| **엔터티 헤더 (Entity)** | 요청 + 응답 양쪽 | 본문 데이터 자체에 관한 정보 |

### 일반 헤더 (General Headers)

요청/응답 메시지 모두에서 사용되며, 메시지의 전송 방식, 연결 상태, 날짜 등을 기술한다.

#### 연결 관련

| 헤더 | 설명 | 예시 |
|---|---|---|
| **Connection** | 현재 연결 후 네트워크 연결을 유지할지 끊을지 제어 | `Connection: keep-alive` / `Connection: close` |
| **Upgrade** | 현재 프로토콜에서 다른 프로토콜로 전환 요청 | `Upgrade: websocket` |
| **Via** | 메시지가 거쳐온 중간 노드(프록시, 게이트웨이) 기록 | `Via: 1.1 proxy.example.com` |

#### 날짜

| 헤더 | 설명 | 예시 |
|---|---|---|
| **Date** | 메시지가 생성된 날짜/시간 (RFC 1123 형식) | `Date: Tue, 15 Nov 1994 08:12:31 GMT` |

#### 캐시 관련

| 헤더 | 설명 | 예시 |
|---|---|---|
| **Cache-Control** | 캐싱 동작 지시. `no-cache`, `max-age` 등 다양한 디렉티브 | `Cache-Control: max-age=3600` |
| **Pragma** | HTTP/1.0 호환 캐시 제어. `no-cache`만 실질적으로 사용됨 | `Pragma: no-cache` |
| **Warning** | 메시지 변환 또는 캐시 상태에 대한 경고 (HTTP/1.1, 현재는 deprecated) | `Warning: 199 - "Miscellaneous warning"` |

#### 전송 인코딩

| 헤더 | 설명 | 예시 |
|---|---|---|
| **Transfer-Encoding** | 본문의 전송 인코딩 방식 (hop-by-hop). `chunked`가 대표적 | `Transfer-Encoding: chunked` |
| **Trailer** | chunked 인코딩 사용 시 본문 뒤에 오는 헤더 목록 사전 선언 | `Trailer: Expires` |

#### MIME

| 헤더 | 설명 | 예시 |
|---|---|---|
| **MIME-Version** | MIME 프로토콜 버전. HTTP에서는 `1.0`이 관례적으로 쓰이지만 의미는 없음 | `MIME-Version: 1.0` |

### Hop-by-Hop vs End-to-End

일반 헤더 중 일부는 **hop-by-hop 헤더**로, 현재 연결 구간에만 적용되고 프록시를 통해 전달되지 않는다.

| 구분 | 헤더 | 설명 |
|---|---|---|
| **Hop-by-Hop** | Connection, Keep-Alive, Transfer-Encoding, Trailer, Upgrade, Proxy-Authorization, TE | 중간 노드에서 소비되고 전달 안 됨 |
| **End-to-End** | 나머지 대부분 (Date, Cache-Control, Via 등) | 최종 수신자까지 전달됨 |
