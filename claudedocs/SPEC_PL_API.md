# 페이레터 BOQV9 REST API 연동 스펙

> **버전**: 1.03 (2024/06/28 기준)
> **작성일**: 2026-01-12
> **목적**: ARS 시나리오 DLL 개발을 위한 페이레터 API 연동 참조 문서

---

## 0. 기존 시스템과의 관계

### ⚠️ 중요: 기존 API와 신규 API 비교

현재 `ALLAT_StockWin_Billkey_Easy_New_Scenario` DLL은 **ASP 기반 레거시 API**를 사용하고 있습니다.
본 문서의 BOQV9 REST API는 **신규 도입 예정 API**입니다.

| 구분 | 기존 API (레거시) | BOQV9 REST API (신규) |
|------|-------------------|----------------------|
| **도메인** | `billadmin.wownet.co.kr` | `swbillapi.wowtv.co.kr` |
| **방식** | ASP 페이지 호출 | RESTful API |
| **엔드포인트 예시** | `orderRequestApi.asp?DNIS=...&HP_NO=...` | `POST /v1/payment/simple/getpaymentinfo_V2` |
| **응답 형식** | XML | JSON |
| **인증** | 없음 (Query Parameter) | HMAC-SHA256 Authorization 헤더 |
| **소스 파일** | `WowTvSocket.cpp` | (신규 구현 필요) |

### 기존 API 엔드포인트 (현재 사용 중)

```
# 주문 정보 조회
https://billadmin.wownet.co.kr/pgmodule/DasomARS/UserCall/orderRequestApi.asp

# 빌키 해지
https://billadmin.wownet.co.kr/pgmodule/DasomARS/UserCall/DirectPayBillTerminate.asp
```

### 마이그레이션 고려사항

1. **인증 모듈 신규 개발 필요**: HMAC-SHA256 서명 생성 로직
2. **응답 파서 변경**: XML → JSON 파싱 로직
3. **에러 처리 변경**: HTTP Status Code 기반 처리
4. **테스트 환경 분리**: QA 서버에서 충분한 테스트 후 운영 전환

### 기능별 API 매핑

| 기능 | 기존 API (레거시) | 신규 BOQV9 REST API |
|------|-------------------|---------------------|
| 간편결제 정보 조회 | `orderRequestApi.asp` | `/v1/payment/simple/getpaymentinfo_V2` |
| 정기결제 정보 조회 | `orderRequestApi.asp` | `/v1/payment/batch/getpayment/arsno_V2` |
| 빌키 해지 | `DirectPayBillTerminate.asp` | `/v1/payment/simple/withdrawal` |
| 결제 동의 등록 | (해당 없음) | `/v1/payment/simple/agree` |
| 캐시 잔액 조회 | (해당 없음) | `/v1/payment/cash/getbalance` |
| 캐시 사용 | (해당 없음) | `/v1/payment/cash/paycash` |
| 상품 지급 | (해당 없음) | `/v1/product/provision/phoneno` |
| 회원 조회 | (해당 없음) | `/v1/member/getmemberstatus` |

### 기존 XML 응답 필드 → 신규 JSON 필드 매핑

```
기존 XML 필드          →  신규 JSON 필드
────────────────────────────────────────
order_no              →  orderNo
shop_id               →  mallIdSimple / mallIdGeneral
cc_name               →  nickName
cc_pord_desc          →  itemName
cc_pord_code          →  (categoryId_2nd 활용)
amount                →  payAmt
partner_nm            →  (별도 조회 필요)
bill_key              →  billKey
ext_data              →  billPassword
renew_flag            →  payAgreeFlag
card_name             →  cardCompany
sub_status            →  batchPayType (0=최초, 1=해지, 2=사용중)
sub_amount            →  payAmt
sub_has_trial         →  (별도 필드 없음, 비즈니스 로직으로 처리)
expire_date           →  serviceEndMonth + serviceEndDay
```

---

## 1. 개요

### 1.1 문서 목적
한국경제TV(WowTV) ARS 결제 시스템에서 페이레터 REST API를 호출하기 위한 개발 참조 스펙 문서입니다.

> **Note**: 본 문서는 신규 BOQV9 REST API 규격을 정리한 것입니다. 기존 ASP 기반 API와 병행 또는 전환하여 사용할 수 있습니다.

### 1.2 서버 환경

| 환경 | 도메인 | 용도 |
|------|--------|------|
| **QA** | `https://devswbillapi.wowtv.co.kr` | 개발/테스트 |
| **LIVE** | `https://swbillapi.wowtv.co.kr` | 운영 |

### 1.3 통신 규격

| 항목 | 값 |
|------|-----|
| 프로토콜 | HTTPS (TLS 1.2+) |
| HTTP 버전 | HTTP/1.1 |
| 인코딩 | UTF-8 |
| 데이터 형식 | JSON (기본), XML (선택) |
| HTTP Method | POST |

---

## 2. 인증 규격

### 2.1 인증 헤더 형식

```
Authorization: PLTOKEN {APP_ID}:{Signature}:{Nonce}:{Timestamp}
```

### 2.2 인증 파라미터

| 파라미터 | 설명 | 예시 |
|----------|------|------|
| **APP_ID** | 발급받은 앱 ID (32자) | `3f2e11523b634600bc432fe38ed73e3d` |
| **APP_KEY** | 비밀키 (Base64, 32바이트) | `c5lyf6x4XvTP8xFPtPEL+...` |
| **Signature** | HMAC-SHA256 서명 (Base64) | 아래 생성 로직 참조 |
| **Nonce** | 임의 고유값 (UUID 권장) | `035ae3de-b68b-4d0f-b5fc-401b98a2da7b` |
| **Timestamp** | UNIX 타임스탬프 (UTC, 초) | `1583391560` |

### 2.3 Signature 생성 로직

```
RequestString = APP_ID + UpperCase(HTTP_METHOD) + Timestamp + Nonce

Signature = Base64( HMAC-SHA256( Base64Decode(APP_KEY), UTF8(RequestString) ) )
```

**예시**:
```
APP_ID     = "3f2e11523b634600bc432fe38ed73e3d"
METHOD     = "POST"
Timestamp  = "1583391560"
Nonce      = "20200305571158"

RequestString = "3f2e11523b634600bc432fe38ed73e3dPOST158339156020200305571158"

Signature = Base64(HMAC-SHA256(Decode(APP_KEY), RequestString))
          = "Lu7AmhstnNLghIafRMNIXrswDRblv/D75Kee1BheIGY="
```

**최종 Authorization 헤더**:
```
Authorization: PLTOKEN 3f2e11523b634600bc432fe38ed73e3d:Lu7AmhstnNLghIafRMNIXrswDRblv/D75Kee1BheIGY=:20200305571158:1583391560
```

### 2.4 주의사항

- **시간 동기화**: 클라이언트-서버 시간 차이 5분 이상이면 요청 거부
- **Nonce 재사용 금지**: 동일 Nonce 5분간 거부 (재전송 공격 방지)
- **APP_KEY 변환**: 반드시 Base64 디코딩 후 바이트 배열로 사용

---

## 3. 응답 규격

### 3.1 성공 응답

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

```json
{
  "resultCode": "0",
  "data": {
    // API별 응답 데이터
  }
}
```

### 3.2 실패 응답

```http
HTTP/1.1 4XX/5XX
Content-Type: application/json; charset=utf-8
```

```json
{
  "error": {
    "code": 911,
    "message": "Not existing user.",
    "detail": ""
  }
}
```

### 3.3 공통 오류 코드

| 코드 | HTTP Status | 설명 |
|------|-------------|------|
| 999 | 500 | 시스템 내부 오류 |
| 998 | 401 | 인증 실패 (헤더 없음/오류) |
| 997 | 400 | 요청 파라미터 오류 |
| 996 | 404 | 리소스 없음 |
| 995 | 405 | 허용되지 않는 Method |
| 994 | 501 | 미구현 API |
| 993 | 403 | 권한 없음 / IP 미허용 |
| 911 | 404 | 사용자 없음 |

---

## 4. API 상세 스펙

### 4.1 간편결제 정보 조회

쿠폰 및 보너스 캐시 적용 정보를 포함한 간편결제 등록정보를 조회합니다.

**Endpoint**
```
POST /v1/payment/simple/getpaymentinfo_V2
```

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `reqType` | Byte | O | 요청타입: 1=회선번호, 2=상품코드, 3=종목알파고용 |
| `reqTypeVal` | String(12) | O | 요청데이터 (회선번호 또는 상품코드) |
| `phoneNo` | String(16) | O | 휴대폰번호 (하이픈 제외) |
| `ARSType` | String(1) | O | ARS구분: `VARS`=보는ARS, `ARS`=듣는ARS |

**Request 예시**
```json
{
  "reqType": 1,
  "reqTypeVal": "1234",
  "phoneNo": "01011112222",
  "ARSType": "VARS"
}
```

**Response Body**

| 필드 | 타입 | 설명 |
|------|------|------|
| `memberId` | String(36) | 회원 ID (UUID) |
| `orderNo` | Int64 | 주문번호 |
| `nickName` | String(20) | 필명 |
| `packageId` | Int32 | 월정액 패키지 ID |
| `itemName` | String(100) | 구매 상품명 |
| `pgCode` | String(1) | PG 코드: `A`=올앳, `P`=페이레터 |
| `categoryId_2nd` | String(6) | 중분류 카테고리ID |
| `merchantId` | String(40) | 가맹점 ID |
| `mallIdSimple` | String(40) | 상점 ID (간편) |
| `mallIdGeneral` | String(40) | 상점 ID (일반) |
| `payAmt` | Int64 | 실제 결제 금액 (할인 적용 후) |
| `CouponUseFlag` | String(1) | 쿠폰 존재 여부: `Y`/`N` |
| `CouponName` | String(50) | 쿠폰 이름 |
| `BonusCashUseFlag` | String(1) | 보너스 캐시 존재 여부: `Y`/`N` |
| `BonusCashUseAmt` | Int64 | 보너스 캐시 사용 금액 |
| `purchaseAmt` | Int64 | 상품 원가 (할인 전) |
| `notiUrlSimple` | String(200) | Notification URL (간편) |
| `notiUrlGeneral` | String(200) | Notification URL (일반) |
| `billKey` | String(128) | 정기결제 키 |
| `billPassword` | String(128) | 정기결제 비밀번호 |
| `cardCompany` | String(30) | 카드사명 |
| `payAgreeFlag` | String(1) | 결제 동의 여부 |
| `purchaseLimitFlag` | String(1) | 구매 제한 여부 (아래 표 참조) |
| `memberState` | Byte | 고객상태: 1=비회원, 2=유료회원, 3=기구매자 |
| `serviceCheckFlag` | String(1) | 서비스 점검여부: `Y`/`N` |
| `checkCompleteTime` | String(16) | 점검완료일시 (yyyy-MM-dd hh:mm) |

**purchaseLimitFlag 값**

| 값 | 설명 |
|----|------|
| 1 | 정상 (구매가능) |
| 2 | 불량사용자 등록 |
| 3 | 구매 가능 횟수 초과 |
| 4 | 판매 시작전 상품 |
| 5 | 판매 종료 상품 |
| 6 | 판매 중지 상품 |

**Response 예시**
```json
{
  "resultCode": "0",
  "data": {
    "memberId": "604ab16a-e525-49d8-984d-b06662ff8c46",
    "orderNo": 202001011222154,
    "nickName": "테스터",
    "packageId": 4,
    "itemName": "주식비타민 3일 패키지(정기)",
    "categoryId_2nd": "AAAAAA",
    "pgCode": "A",
    "merchantId": "testmerchantid",
    "mallIdSimple": "testmallidsimple",
    "mallIdGeneral": "testmallidgeneral",
    "payAmt": 5000,
    "CouponUseFlag": "Y",
    "CouponName": "정액 50%할인 쿠폰",
    "BonusCashUseFlag": "Y",
    "BonusCashUseAmt": 8500,
    "purchaseAmt": 27000,
    "notiUrlSimple": "http://swbill.wowtv.co.kr/noti/simple",
    "notiUrlGeneral": "http://swbill.wowtv.co.kr/noti/general",
    "billKey": "kefweerwav21aslkd3f",
    "billPassword": "800101",
    "cardCompany": "신한카드",
    "payAgreeFlag": "Y",
    "purchaseLimitFlag": "1",
    "memberState": 2,
    "serviceCheckFlag": "N",
    "checkCompleteTime": ""
  }
}
```

---

### 4.2 캐시 잔액 조회

종목알파고 내 캐시 정보를 조회합니다.

**Endpoint**
```
POST /v1/payment/cash/getbalance
```

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `phoneNo` | String(16) | O | 휴대폰번호 (하이픈 제외) |

**Response Body**

| 필드 | 타입 | 설명 |
|------|------|------|
| `cashBalance` | Int64 | 캐시 잔액 |
| `totalInCashAmt` | Int64 | 충전 캐시 총 금액 (총 입금액) |
| `totalOutCashAmt` | Int64 | 충전 캐시 총 사용금액 (총 출금액) |

---

### 4.3 캐시 사용

종목알파고 내 상품 금액만큼 캐시를 차감합니다.

**Endpoint**
```
POST /v1/payment/cash/paycash
```

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `phoneNo` | String(16) | O | 휴대폰번호 (하이픈 제외) |
| `cpItemId` | String(6) | O | 업체상품코드 (종목코드) |

**Response Body**

| 필드 | 타입 | 설명 |
|------|------|------|
| `usedCashAmt` | Int64 | 사용된 캐시금액 |
| `cashBalance` | Int64 | 구매 후 캐시잔액 |

---

### 4.4 ARS 정기결제 정보 조회

ARS 정기결제 진행을 위한 주문 및 결제 정보를 조회합니다. 쿠폰 및 보너스 캐시 적용 정보를 포함합니다.

**Endpoint**
```
POST /v1/payment/batch/getpayment/arsno_V2
```

**Request Body** - 4.1과 동일

**Response Body**

| 필드 | 타입 | 설명 |
|------|------|------|
| `memberId` | String(36) | 회원 ID (UUID) |
| `orderNo` | Int64 | 주문번호 |
| `nickName` | String(20) | 필명 |
| `packageId` | Int32 | 월정액 패키지 ID |
| `itemName` | String(100) | 구매 상품명 |
| `pgCode` | String(1) | PG 코드: `A`=올앳, `P`=페이레터 |
| `categoryId_2nd` | String(6) | 중분류 카테고리ID |
| `merchantId` | String(40) | 가맹점 ID |
| `mallIdSimple` | String(20) | 상점 ID (간편) |
| `mallIdGeneral` | String(20) | 상점 ID (일반) |
| `payAmt` | Int64 | 실제 결제 금액 (할인 적용 후) |
| `CouponUseFlag` | String(1) | 쿠폰 존재 여부: `Y`/`N` |
| `CouponName` | String(50) | 쿠폰 이름 |
| `BonusCashUseFlag` | String(1) | 보너스 캐시 존재 여부: `Y`/`N` |
| `BonusCashUseAmt` | Int64 | 보너스 캐시 사용 금액 |
| `purchaseAmt` | Int64 | 상품 원가 (할인 전) |
| `payAgreeFlag` | String(1) | 결제 동의 여부 |
| `purchaseLimitFlag` | String(1) | 구매 제한 여부 (4.1 참조) |
| `batchPayType` | Byte | 정기결제 사용구분: 0=최초가입, 1=해지, 2=사용중 |
| `serviceEndMonth` | String(2) | 서비스 종료 월 |
| `serviceEndDay` | String(2) | 서비스 종료 일 |
| `serviceDivDay` | String(2) | 남은 기간 (일) |
| `serviceCheckFlag` | String(1) | 서비스 점검여부: `Y`/`N` |
| `checkCompleteTime` | String(16) | 점검완료일시 (yyyy-MM-dd hh:mm) |

**Response 예시**
```json
{
  "resultCode": "0",
  "data": {
    "memberId": "604ab16a-e525-49d8-984d-b06662ff8c46",
    "orderNo": 202001011222154,
    "nickName": "테스터",
    "packageId": 4,
    "itemName": "주식비타민 3일 패키지(정기)",
    "categoryId_2nd": "AAAAAA",
    "pgCode": "A",
    "merchantId": "testmerchantid",
    "mallIdSimple": "testmallidsimple",
    "mallIdGeneral": "testmallidgeneral",
    "payAmt": 34000,
    "CouponUseFlag": "Y",
    "CouponName": "정액 50%할인 쿠폰",
    "BonusCashUseFlag": "Y",
    "BonusCashUseAmt": 8500,
    "purchaseAmt": 27000,
    "payAgreeFlag": "Y",
    "purchaseLimitFlag": "1",
    "batchPayType": 2,
    "serviceEndMonth": "03",
    "serviceEndDay": "31",
    "serviceDivDay": "10",
    "serviceCheckFlag": "N",
    "checkCompleteTime": ""
  }
}
```

---

### 4.5 간편결제 해지 처리

**Endpoint**
```
POST /v1/payment/simple/withdrawal
```

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `memberId` | String(16) | O | 회원ID |
| `billKey` | String(128) | O | 정기결제 키 |
| `pgCode` | String(20) | O | PG코드 |

**Response Body**
```json
{
  "resultCode": "0"
}
```

---

### 4.6 상품 지급

특정 휴대폰 번호로 패키지 상품을 지급합니다.

**Endpoint**
```
POST /v1/product/provision/phoneno
```

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `phoneNo` | String(16) | O | 휴대폰번호 (하이픈 제외) |
| `packageId` | Int32 | O | 패키지ID |
| `provider` | String(50) | O | 지급처 (고정값: `MSPAY`) |
| `provisionReason` | String(200) | O | 지급사유 |

**Response Body**

| 필드 | 타입 | 설명 |
|------|------|------|
| `memberId` | String(36) | 회원ID |
| `chargeNo` | Int64 | 지급된 상품의 구매번호 |
| `packageId` | Int32 | 패키지ID |
| `expireYmd` | String(8) | 만료일 (yyyyMMdd) |

**지급 가능 패키지 목록**

| 패키지ID | 상품명 |
|----------|--------|
| 100391 | 김종철증권알파고 무료 1주일 |
| 100698 | 이헌상수급박스 무료 1주일 |
| 100988 | 박영호단타왕 무료 1주일 |
| 101114 | 주식비타민 무료 1주일 TV 이벤트 |
| 101231 | 권태민오늘승부주 앱 무료 1주일 |

---

### 4.7 간편결제 동의 등록

**Endpoint**
```
POST /v1/payment/simple/agree
```

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `phoneNo` | String(16) | O | 휴대폰번호 (하이픈 제외) |
| `pgCode` | String(20) | O | PG 코드: `A`=올앳, `P`=페이레터 |
| `agreeFlag` | String(1) | O | 동의여부: `Y`=동의 |

**Response Body**
```json
{
  "resultCode": "0"
}
```

---

### 4.8 종목알파고 사용가능 여부 조회

**Endpoint**
```
POST /v1/members/stocksalphago/chkavailphoneno
```

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `phoneNo` | String(16) | O | 전화번호 (하이픈 제외) |
| `stocksCode` | String(6) | O | 업체상품ID (종목코드) |

**Response Body**

| 필드 | 타입 | 설명 |
|------|------|------|
| `memberId` | String(36) | 회원 ID (UUID) |
| `packageId` | String(10) | 월정액 패키지 ID |
| `itemId` | String(4) | 단품 ID |
| `itemName` | String(100) | 구매 상품명 |
| `purchaseYmd` | String(8) | 구매일 (yyyyMMdd) |
| `expireYmd` | String(8) | 만료일 (yyyyMMdd) |
| `availFlag` | String(1) | 조회여부: `Y`=존재, `N`=미존재 |
| `memberState` | String(1) | 고객상태: 1=비회원, 2=유료회원, 3=기구매자 |

---

### 4.9 ARS 예약 캐시 롤백

**Endpoint**
```
POST /v1/payment/simple/reserverollback
```

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `OrderNo` | Int64 | O | 주문번호 |
| `MemberId` | String(50) | O | 회원ID |

**Response Body**
```json
{
  "resultCode": "0"
}
```

---

### 4.10 Google Analytics 등록용 사용자 번호 조회

**Endpoint**
```
POST /v1/member/getmemberno
```

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `Uuid` | String(36) | O | 회원 UUID |

**Response Body**

| 필드 | 타입 | 설명 |
|------|------|------|
| `MemberNo` | Int64 | 사용자번호 (미존재시 -1) |

---

### 4.11 회원 및 비회원 조회

ARS 무료상품 지급 시나리오상 회원/비회원 조회

**Endpoint**
```
POST /v1/member/getmemberstatus
```

**Request Body**

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `phoneNo` | String(16) | O | 휴대폰번호 (하이픈 제외) |
| `packageId` | Int32 | O | 서비스명 검색할 패키지ID |

**Response Body**

| 필드 | 타입 | 설명 |
|------|------|------|
| `UserState` | Byte | 1=비회원, 2=회원 |
| `PackageName` | String(100) | 패키지명 (시나리오상 서비스명) |

---

## 5. 개발 체크리스트

### 5.1 인증 구현 체크리스트

- [ ] APP_ID, APP_KEY 발급 및 보안 저장
- [ ] HMAC-SHA256 서명 생성 로직 구현
- [ ] Nonce UUID 생성 구현
- [ ] UTC 기준 UNIX Timestamp 생성
- [ ] Authorization 헤더 조립 함수 구현
- [ ] 시간 동기화 확인 (NTP 권장)

### 5.2 API 호출 체크리스트

- [ ] HTTPS 통신 구현 (TLS 1.2+)
- [ ] JSON 요청/응답 파싱 구현
- [ ] HTTP Status Code 기반 성공/실패 판단
- [ ] 오류 응답 처리 (`error` 객체 파싱)
- [ ] 타임아웃 설정 (권장: 30초)
- [ ] 재시도 로직 구현 (네트워크 오류시)

### 5.3 데이터 검증 체크리스트

- [ ] 휴대폰번호 하이픈 제거 처리
- [ ] 필수 필드 검증
- [ ] 응답 데이터 타입 검증 (Int64 금액 필드 주의)
- [ ] `purchaseLimitFlag` 값에 따른 분기 처리
- [ ] `serviceCheckFlag` 점검 여부 확인

---

## 6. C++ 구현 참고

### 6.1 인증 헤더 생성 예시 (의사코드)

```cpp
#include <openssl/hmac.h>
#include <openssl/evp.h>

std::string GenerateSignature(
    const std::string& appId,
    const std::string& appKeyBase64,
    const std::string& method,
    const std::string& timestamp,
    const std::string& nonce)
{
    // 1. RequestString 생성
    std::string requestString = appId + ToUpper(method) + timestamp + nonce;

    // 2. APP_KEY Base64 디코딩
    std::vector<unsigned char> appKey = Base64Decode(appKeyBase64);

    // 3. HMAC-SHA256 계산
    unsigned char hmacResult[EVP_MAX_MD_SIZE];
    unsigned int hmacLen;
    HMAC(EVP_sha256(),
         appKey.data(), appKey.size(),
         (unsigned char*)requestString.c_str(), requestString.length(),
         hmacResult, &hmacLen);

    // 4. Base64 인코딩
    return Base64Encode(hmacResult, hmacLen);
}

std::string BuildAuthHeader(
    const std::string& appId,
    const std::string& signature,
    const std::string& nonce,
    const std::string& timestamp)
{
    return "PLTOKEN " + appId + ":" + signature + ":" + nonce + ":" + timestamp;
}
```

### 6.2 API 호출 예시 (의사코드)

```cpp
bool CallPaymentAPI(const std::string& endpoint, const std::string& jsonBody)
{
    // 1. 인증 정보 생성
    std::string nonce = GenerateUUID();
    std::string timestamp = GetUnixTimestamp();
    std::string signature = GenerateSignature(APP_ID, APP_KEY, "POST", timestamp, nonce);
    std::string authHeader = BuildAuthHeader(APP_ID, signature, nonce, timestamp);

    // 2. HTTP 헤더 설정
    headers["Authorization"] = authHeader;
    headers["Content-Type"] = "application/json; charset=utf-8";

    // 3. HTTPS POST 요청
    HttpResponse response = HttpPost(BASE_URL + endpoint, jsonBody, headers);

    // 4. 응답 처리
    if (response.statusCode == 200) {
        // JSON 파싱 및 resultCode 확인
        return ParseSuccessResponse(response.body);
    } else {
        // 오류 처리
        return HandleErrorResponse(response.body);
    }
}
```

---

## 7. 변경 이력

| 버전 | 일자 | 내용 |
|------|------|------|
| 0.9 | 2020/01/08 | 최초 작성 |
| 0.91 | 2020/03/05 | 인증헤더 Signature 생성 시 Request-Uri, Content 항목 제외 |
| 0.92 | 2020/03/20 | ARS구분, 종목코드 요청 데이터 추가 |
| 0.93 | 2020/04/02 | resultCode 필드 추가, URI 버전 관리 |
| 0.94 | 2020/04/07 | 금액 필드 데이터형 Double→Int64 변환 |
| 0.95 | 2020/04/10 | 간편결제 동의 등록, 종목알파고 사용가능 여부 조회 추가 |
| 0.96 | 2020/04/13 | ARS 결제정보 조회 삭제, 응답 파라미터 변경 |
| 0.97 | 2020/04/17 | PG코드 데이터 변경 (A:올앳, P:페이레터) |
| 0.98 | 2020/04/22 | 보이는 ARS 관련 중분류 카테고리ID 추가 |
| 0.99 | 2020/07/27 | 미사용 API 삭제 |
| 1.00 | 2022/11/28 | 간편/정기 1회차 결제 시 쿠폰, 보너스 캐시 사용 처리 |
| 1.02 | 2024/02/15 | 간편/정기결제 정보 조회 매뉴얼 수정, UUID 사용자 정보조회 |
| 1.03 | 2024/06/28 | 회원 및 비회원 조회 추가 |

---

## 8. 참고 자료

- 원본 문서: `docs/BOQV9 REST API Protocol.docx`
- 관련 소스: `ALLAT_Access.cpp`, `WowTvSocket.cpp`
- 프로젝트: ALLAT StockWin Billkey Easy New Scenario DLL
