# 네트워크 통신의 과정

---

# 1. URL 입력 및 DNS 조회 (응용 계층)
> 클라이언트가 브라우저 주소창에 `www.google.com`을 입력하면 시스템은 이 문자를 컴퓨터가 이해할 수 있는 IP 주소로 변환하기 위해 DNS 서비스를 이용합니다.

## 1.1 로컬 캐시 확인 및 DNS 서버 접속

```text
Browser Cache: 내부 IP 주소 확인
Client → DNS: 서버 접속
```
- 먼저 PC 내부 Cache의 기록을 확인하여 기록이 없다면 DNS 서버와 통신을 시작합니다. DNS는 오버헤드가 적고 빠른 응답이 가능한 **UDP 프로토콜(포트 53)** 를 사용합니다.

## 1.2 DNS 질의 수행

### 1.2.1 Recursive Query (재귀적 질의)

```text
Client → DNS: "최종 IP 주소 요청"
```
- 클라이언트는 DNS 서버에게 최종 IP 주소를 요청합니다.

### 1.2.2 Iterative Query (반복적 질의)

```text
DNS → Root DNS: ".com 네임 서버 주소 반환"
DNS → TLD DNS: "google.com 네임 서버 주소 반환"
DNS → Authoritative DNS: "www.google.com의 최종 IP 주소 반환"
```
- DNS 서버가 전 세계 계층화된 하위 DNS 서버를 하나씩 접속해 정보를 수집하는 과정입니다. 각 서버는 자신이 아는 범위 내에서 다음 목적지를 알려줍니다.

## 1.3 IP 주소 반환

```text
"DNS → Client: "최종 IP 주소 반환"
```
- 모든 질의를 마친 DNS 서버가 최종 IP 주소를 클라이언트에게 반환합니다.

# 2. TCP 연결 확립 (전송 계층)

> 데이터의 안전한 전송을 위해 `TCP 프로토콜(포트 80)`을 사용하여 서버와 신뢰성 있는 연결을 맺습니다.

## 2.1 세션 수립

```text
3-Way Handshake
Client → Server: SYN(연결 요청)
Server → Client: SYN-ACK(연결 요청 확인 및 응답)
Client → Server: ACK(연결 확립 완료)
```
- **HTTP 프로토콜(포트 80)** 요청을 위해 3번의 패킷 교환을 통해 두 장치 간의 논리적인 통신 통로가 완전히 개방됩니다.

# 3. HTTP 요청 (응용 계층)
> 연결된 통로를 통해 사용자가 원하는 웹 페이지를 가져오기 위한 요청 메시지를 작성하는 단계입니다.

## 3.1 HTTP 요청 메시지 생성
```http
GET / HTTP/1.1
Host: [www.google.com](https://www.google.com)
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...
Accept: text/html,application/xhtml+xml ...
Connection: keep-alive
```
