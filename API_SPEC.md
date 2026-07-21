# 똑똑용돈 API 명세서 v2.1 (iOS 클라이언트 기준)

> 기준: `knocknockMoney-ios` 목업 코드(화면·데이터 구조)와 기능명세서·유저스토리.
> 백엔드는 미구현 상태이며 이 문서가 **구현 대상 계약 초안**이다. 구현 후에는 서버 `/openapi.json`(OpenAPI 3.1)이 기계 판독 정본이 된다.
> v1 대비 주요 변경: 게임 시드를 미션게임 기획서 최종 6종으로 교체, 성공/실패 판정 도입, 보상 구조를 "가족 지갑 충전 → 시니어 적립 → 출금/상점" 모델로 개편, 교육·알림·상점·출금·주간목표 API 신설, 대시보드를 iOS 화면 단위로 재설계. (근거: 부록 B 화면→API 매핑)
> v2.1 추가: **음성 응원 편지 신설**(§3 — 우체통·편지함, 24시간 휘발/프리미엄 영구 보관), **똑똑이 음성 일기 신설**(§4 — 기기 내 STT, 음성 원본 미수집, 별도 동의), **가족 선물하기**(§6 — 상점 구매에 수령인 지정, 가족 커머스·B2B 접점), **프리미엄 구독**(§11 — 3티어 entitlement + 서버 게이트 402), **케어 푸시 신설**(§8 무활동 룰 확정 — 3일 무활동 alert + §10 디바이스 등록·APNs 발송, 티어 무관), **아바타 꾸미기 신설**(§2 — 얼굴 고정, 머리·옷 2슬롯, G-2 화면, 상점 `deco` 연동), 상점 카테고리·제휴 확장(§6), 교육 콘텐츠 `premium` 플래그(§5), 가족 지갑 충전 내역·충전자 표시(§7), 형제자매(guardian) 초대 명시(§3), 동의 코드 `third_party_research` 예약(§1.7).
> **확정 정책**: 1P = 1원 고정 · 일일 미션 5개(홈 진행률 분모 5) · 초대 코드 4자리 · 로그인은 Apple + 카카오(휴대폰 OTP 미제공) · 활동 달력 제공(§4, 시니어·자녀 공용) · 게임 시드 = 기획서 최종 6종(§1.8) · **걸음 기능 제거** · 반응시간·힌트 계측 필수(§4) · 보상 포인트 = 인지 기여도(연구 연계 순위) 비례 + **AI 기피 게임 보너스**(§1.8·§9) · **음성 응원 편지**(§3 — 24h 휘발/프리미엄 영구, 분석 투입 금지) · **똑똑이 음성 일기**(§4 — 음성 원본 미수집·전사 암호화·원문 기본 비공개(공유는 시니어 본인이 제어), §1.9 특칙) · **가족 선물하기**(§6) · **프리미엄 구독 3티어 + 서버 측 게이트**(§11).
> **확정 스택**: iOS **Swift + SwiftUI** · Backend **Python + FastAPI + PostgreSQL(pgvector)** · AI **Gemini 3.5 Flash(구조화) + gemini-embedding-001(임베딩 768차원) + Qwen 3 32B(추천·리포트, OpenAI 호환 API — 운영 OpenRouter) + 통계 로직(패턴 분석)** — 태스크별 매핑은 §9 · CI/CD **GitHub Actions**.

- Base URL: `{HOST}/api/v1` (로컬: `http://localhost:8000/api/v1`)
- 문서 UI: `{HOST}/docs` (Swagger)
- 모든 요청/응답 본문: `application/json` (UTF-8)

---

## 1. 공통 규약

### 1.1 인증

모든 엔드포인트(운영용 제외)는 Firebase ID 토큰을 요구한다.

```
Authorization: Bearer <Firebase ID Token>
```

- 로그인 수단: **Apple 로그인 + 카카오 로그인** (둘 다 Firebase Auth 경유). 휴대폰 OTP는 제공하지 않는다 — iOS의 "휴대폰 번호로 계속하기" 버튼과 `PhoneLoginFlow` 화면은 **카카오 로그인 버튼으로 대체**.
- 카카오 연동은 **Firebase OIDC 제공자(`oidc.kakao`)** 방식: iOS가 KakaoSDK 네이티브 로그인(카카오톡 앱 전환)으로 OIDC `id_token`을 받아 `OAuthProvider.credential(providerID: "oidc.kakao", idToken:)`으로 Firebase에 로그인 → 이후 흐름은 Apple과 동일(`POST /auth/session`). **카카오 전용 백엔드 엔드포인트는 없다** — 서버는 로그인 수단과 무관하게 Firebase ID 토큰의 `uid`만 신뢰한다.
- 같은 사람이 Apple·카카오로 각각 가입하면 **별도 계정**이 된다(MVP: 계정 통합·연결 미제공 — 가입 화면에 "이전에 쓰던 로그인 방법을 선택하세요" 안내 필요).
- 백엔드는 토큰의 `uid`만 신뢰한다. **클라이언트는 어떤 요청에도 자기 user_id를 인증 목적으로 보내지 않는다.**
- 최초 로그인 시 반드시 `POST /auth/session`을 먼저 호출해 사용자를 등록해야 한다. 미등록 상태로 다른 API 호출 시 `401` (detail: "Unknown user — call POST /v1/auth/session first").
- 개발 환경(`AUTH_MODE=debug`)에서는 `Bearer debug:<uid>` 가짜 토큰 허용. 데모 계정: `debug:demo-senior-1`(시니어), `debug:demo-guardian-1`(자녀).
- 남용 방지 레이트리밋: `POST /auth/session` **30회/분/IP**, `POST /families/join` **10회/10분/사용자**(§3) — 초과 시 429.

### 1.2 역할·권한 모델

- `role`: `senior` | `guardian`. **iOS의 `UserRole.child`가 API의 `guardian`에 대응**한다(코드 매핑 필수, 부록 A-7 참조).
- 역할은 가입 시 1회 확정(변경 API 없음). 현재 iOS는 역할을 기기 로컬(UserDefaults)에 저장하는데, **서버 저장으로 전환**해야 다기기·재설치에서 유지된다.

| 리소스 | 시니어 본인 | 같은 가족 guardian (active) | 외부인 |
|---|---|---|---|
| 시니어 데이터 읽기 (home, wallet, metrics, pattern, history …) | ✅ | ✅ (감사 로그 기록) | ❌ 403 |
| 시니어 데이터 쓰기 (attempts, checkins, steps, edu 완료, 상점 구매, 출금) | ✅ | ❌ 403 | ❌ 403 |
| 가족 리소스 읽기 (members, dashboard, reports, care-alerts, reward-*) | 소속 시 ✅ | 소속 시 ✅ | ❌ 403 |
| 보상 규칙 설정(PUT reward-rules), 포인트 충전, 리포트 수동 생성 | ❌ 403 | ✅ | ❌ 403 |
| 음성 응원 편지 발송 (POST cheers, §3) | ❌ 403 | ✅ | ❌ 403 |
| 편지함 열람 (cheer-letters, §3) | ✅ (수신분) | 자기 발신분만 | ❌ 403 |
| 똑똑이 음성 일기 원문 (voice-diaries, §4) | ✅ (읽기·쓰기·삭제) | 기본 ❌ 403 — **시니어가 공유를 켠 경우에만 읽기 ✅**(감사 로그) | ❌ 403 |
| 프리미엄 게이트 리소스 (guardian 심화 열람 — §11 표) | ✅ (게이트 없음) | 구독 시 ✅ / 무료 ❌ 402 | ❌ 403 |

- 한 가족에 **guardian 여러 명 허용**(형제자매). 초대·참여 플로우는 §3 공용 — `member_role`은 가입 시 확정된 `role`을 따른다.
- 프리미엄 게이트는 **guardian 호출에만 적용**된다. 시니어 본인의 자기 데이터 조회는 티어와 무관하게 항상 허용(§11).

### 1.3 에러 포맷 — RFC 9457 `application/problem+json`

```json
{
  "type": "about:blank",
  "title": "Forbidden",
  "status": 403,
  "detail": "Not a guardian of this senior",
  "instance": "/api/v1/seniors/{id}/home"
}
```

| 코드 | 의미 | 클라이언트 처리 |
|---|---|---|
| 400 | 잘못된 요청(만료/사용된 초대 코드, 잔액 부족 등) | 사용자 안내 |
| 401 | 토큰 없음/만료/미등록 | 토큰 갱신 → 재시도, 실패 시 재로그인 |
| 402 | 프리미엄 구독 필요 (`title: "Payment Required"`, §11) | 페이월 화면(F-13)으로 유도 |
| 403 | 권한 없음 | 화면 접근 차단 |
| 404 | 리소스 없음 | 안내 |
| 409 | 상태 충돌(이미 성공 완료된 미션, 이미 구매한 아이템 등) | **오프라인 큐에서 해당 항목 제거** |
| 422 | 필드 검증 실패 (`errors` 배열 포함) | 개발 단계에서 수정 |
| 429 | 레이트리밋 | 백오프 후 재시도 또는 안내 |
| 5xx | 서버 오류 | 재시도(지수 백오프) |

### 1.4 시간·날짜 규칙

- 모든 타임스탬프는 **ISO 8601 UTC** (`2026-07-21T05:17:26Z`). "방금 전/2시간 전" 등 상대 표기는 클라이언트가 렌더링.
- `date` 필드는 **서버가 시니어 타임존(Asia/Seoul) 기준으로 계산해 내려주는 값**. 클라이언트는 자체 시계로 "오늘"을 판정하지 않는다(홈의 "7월 21일 화요일"도 서버 `date` 렌더링).
- 주(`week_start`)는 월요일 시작.

### 1.5 멱등성 (offline-first 큐)

- **미션 결과 제출·상점 구매·출금 신청·포인트 충전**은 클라이언트 생성 `idempotency_key`(UUID 권장, 8~64자)를 본문에 담는다. `Idempotency-Key` 헤더도 허용(헤더 우선).
- 같은 키 재전송 시 서버는 **기존 결과를 200으로 반환**하고 `replayed: true`로 표시한다. 포인트 이중 처리 서버 차단.
- 체크인·걸음 동기화는 키 없이 `(시니어, 날짜)` upsert(last-write-wins)라 재전송 안전.

### 1.6 페이지네이션 (커서)

- `?cursor=&limit=` 방식. `next_cursor`가 `null`이면 마지막 페이지.

### 1.7 enum 값 (공유 상수)

| 항목 | 값 |
|---|---|
| `role` / `member_role` | `senior`, `guardian` (iOS `child` ↔ `guardian`) |
| `mission_type` | `game`, `checkin` (교육은 별도 리소스 §5) |
| `cognitive_domain` | `memory`, `attention`, `language`, `executive`, `visuospatial`, `null` |
| assignment `status` | `assigned`, `completed`, `expired` |
| attempt `status` | `in_progress`, `completed`, `abandoned` |
| assignment `source` | `rule`, `ai` |
| ledger `reason_type` | `mission_complete`, `engagement_bonus`, `checkin`, `edu_complete`, `goal_bonus`, `streak_bonus`, `redeem`, `withdraw`, `adjust` |
| care `level` / alert `severity` | `safe`·`info`, `caution`, `alert` |
| metric `key` | `accuracy`, `reaction_time`, `participation` |
| notification `type` | `mission`, `reward`, `family`, `care`, `report` |
| consent `code` | `over14`, `tos`, `privacy`, `sensitive_data`, `marketing`, `voice_diary`(선택 — 똑똑이 음성 일기 opt-in, §4 발화 수집·분석 동의. 미동의 시 기능 비활성일 뿐 다른 기능 제약 없음), `third_party_research`(선택 — 온보딩 미노출, B2B/B2G 비식별 집계 제공 개시 시 동의 화면 추가) |
| calendar day `status` | `none`, `partial`, `complete` |
| cheer `visibility` (음성 응원 편지, §3) | `named`(실명 표시), `anonymous`(익명 표시 — 서버는 발신자 기록 유지) |
| subscription `tier` (§11) | `free`, `premium`, `premium_ai` |
| subscription `status` (§11) | `active`, `expired`, `demo`, `trial`(무료 체험 — 도입 확정, 기간·중복 방지 기준은 A-7 ④) |
| shop `category` (§6) | `deco`, `theme`, `widget`, `gifticon`, `edu`, `health`, `partner` |
| shop `fulfillment_type` (§6) | `demo`, `api`, `physical` |

### 1.9 데이터·보안 원칙 (보안 문서 반영)

수집·저장 데이터: 계정 연동용 기본 식별 정보, 프로필, 미션 참여·활동 데이터, AI 생성 리포트·추천 결과. **실제 금융결제 정보와 의료기관의 진료·의료 데이터는 수집·저장하지 않는다.**

- **수집 항목(확정)**: 회원가입 시 **실명·이메일·전화번호·성별·생년월일**(+역할·타임존)을 수집한다. 그 외 항목(주소·주민번호 등)은 수집하지 않으며, 새 개인정보 필드 추가 시 보안 검토를 선행한다. 실명·이메일·전화번호·생년월일은 `identity` 스키마에 **암호화 저장**하고 검색은 HMAC 인덱스로만 한다.
- **개인정보·활동 데이터 분리 저장**: DB를 **5개 스키마로 분리**(`identity` / `activity` / `reward` / `ai` / `audit`)하고 교차 스키마 물리 FK 없이 **내부 식별자(UUID)로만 논리 연결**한다. `activity`·`reward`·`ai` 스키마에는 이름·연락처 컬럼 자체를 두지 않는다. 활동 리소스 응답(attempts, checkins, metrics …)에도 식별 정보를 포함하지 않는다.
- **토큰·코드 취급**: 초대 코드는 HMAC 해시만 저장(평문은 발급 응답 1회), 감사 로그의 IP는 HMAC 해시만 보관(원본 미저장), LLM 프롬프트는 원문을 저장하지 않고 입력 해시만 기록한다.
- **인증·권한**: Firebase Authentication JWT(§1.1) + 가족 연결 기반 열람 권한(§1.2). 대시보드·리포트는 연결된 가족에게만 열람 허용하며 guardian의 시니어 데이터 열람은 **감사 로그**를 남긴다.
- **암호화**: 전 구간 HTTPS(TLS) 필수. 전화번호 등 주요 개인정보는 저장 시 암호화하고, 탈퇴 시 암호화 키를 폐기한다(§2 `DELETE /me`).
- **결제 정보 미수집**: 포인트 충전(§7)·프리미엄 구독(§11)은 App Store 인앱 결제로 처리 — 카드 정보는 Apple이 보관하고 서버는 StoreKit 트랜잭션 ID만 저장한다. 출금 화면의 계좌 정보는 데모 표시이며 **실계좌 수집은 MVP 범위 외**(부록 A-7 ③).
- **AI 개인정보 보호**: 외부 LLM(Google AI Studio·Groq 등)에는 이름·연락처 등 직접 식별 정보를 전달하지 않고, 가명 처리된 활동 데이터만 전송한다(§9).
- **음성 응원 편지(§3) 특칙**: 자녀 음성은 **전달 콘텐츠**라서 시니어 음성 일기와 달리 서버 저장이 필요하다 — 단, ① 오디오·전사는 **전달 목적으로만** 암호화 저장하고 **어떤 AI·통계 분석 파이프라인에도 투입하지 않는다**(발화 분석은 시니어 일기 전용). 유일한 예외는 발송 시점 1회의 **안전 필터**(§3 — 서버 내 금칙어·패턴 검사, 외부 AI 미전송, 통과/차단 여부만 기록). ② 보존은 **24시간 휘발 또는 영구 보관 둘 중 하나** — 발송 시점의 발신자 티어로 확정되며 이후 변하지 않는다(무료 발신 24시간 후 자동 파기·사후 전환 불가, premium 발신 영구 — 구독 해지 후에도 유지, §11). ③ 접근은 **수신 시니어 + 발신자**로 한정하고 오디오는 단기 서명 URL로만 제공. ④ 전사는 DEK 암호화, LLM 입력 제외. ⑤ 익명 발송은 **표시상 익명일 뿐 서버는 발신자를 기록**한다(남용 대응 — 개인정보처리방침에 명시).
- **똑똑이 음성 일기(§4) 특칙 — 서비스 최대 민감 데이터**: ① **음성 원본은 서버로 전송·저장하지 않는다** — STT는 기기 내(on-device) 처리 원칙, 오디오 업로드 API 자체를 두지 않는다. ② 전사 텍스트는 `voice_diary` **별도 명시 동의**(§1.7) 후에만 수집하며 DEK **암호화 저장**, 보존 기간 만료 시 원문 파기(파생 지표만 보존). ③ **전사 원문은 기본 비공개 — 공개 여부는 시니어 본인이 결정한다**: 시니어가 공유 설정(`voice_diary_share_with_family`, §2)을 켠 경우에만 같은 가족 guardian이 원문을 열람할 수 있고(열람마다 감사 로그), 언제든 끌 수 있으며 끄는 즉시 접근이 회수된다. 공유가 꺼진 상태(기본값)에서는 가족에게 참여 여부·정서 신호·발화 특징 변화 요약만 제공한다. ④ 외부 LLM 전송 전 **PII 스크러빙 필수**(이름·연락처·주소·기관명 마스킹 + 가명 ID). ⑤ 발화 특징 분석(반복·조리성·단어 회상 등)은 인지 수행 데이터와 동일 등급의 민감 자산으로 취급하고, 결과는 의료적 진단이 아닌 **참여·변화 참고 신호**로만 표현한다(비진단 원칙). ⑥ 시니어 본인은 자기 일기 원문을 열람·삭제할 수 있고, 동의 철회 시 전사·지표를 삭제한다. — 체크인 `note_text`는 종전대로 LLM 입력 제외 유지(이 특칙은 음성 일기 전사에만 적용).
- **자금 흐름 남용 방어**: 앱은 부모-자식 관계를 공식 증명하지 않으므로 자금 이동 채널(세탁·보이스피싱·탈세)로 악용될 수 있다 — 실결제 전환 시 ⓐ 출금은 시니어 본인 명의 계좌만 ⓑ 충전·출금·현금성 선물 한도 ⓒ 신규 연결 냉각 기간 ⓓ 이상 거래 탐지(FDS) ⓔ 환불은 원결제수단만을 적용한다. 통제 정본은 보안 문서.md §4.7(M-1~M-8), 수치는 부록 A-7 ⑩.
- MVP 이후 단계적 강화: 접근 로그 관리, 비식별화 처리, 데이터 암호화 고도화, 개인정보 접근 이력 관리.
- 세부 통제 항목·등급·보강 로드맵은 [SECURITY.md](./SECURITY.md)(**목표 보안 설계 체크리스트**)를 따른다.

### 1.8 미션 시드 (미션게임 기획서 최종 6종 + 체크인)

| code | 이름 | type | domain | 표시 라벨 | 기본 보상 | 예상 시간 | 기획서 순위 | iOS 상태 |
|---|---|---|---|---|---|---|---|---|
| `digit_symbol` | 숫자·기호 짝맞추기 | game | attention | **순발력** | **7,000P** | 180s | 1위 | ✅ 구현 완료(계측 포함) |
| `spot_difference` | 다른 그림 찾기 | game | visuospatial | **관찰력** | **6,600P** | 240s | 2위 | ✅ 구현 완료(계측 포함) |
| `sequence_recall` | 순서 기억하기 | game | memory | **기억력** | **6,200P** | 180s | 3위 | ✅ 구현 완료(계측 포함) |
| `signal_tap` | 신호에 맞춰 누르기 | game | attention | **집중력** | **5,800P** | 120s | 4위 | ✅ 구현 완료(계측 포함) |
| `word_sort` | 단어·그림 분류하기 | game | language | **언어력** | **5,400P** | 240s | 5위 | ✅ 구현 완료(계측 포함) |
| `daily_order` | 일상 순서 맞추기 | game | executive | **판단력** | **5,000P** | 240s | 6위 | ✅ 구현 완료(계측 포함) |
| `daily_checkin` | 기분·수면 체크인 | checkin | null | — | 1,000P | 60s | — | ✅ 구현 완료 |

- 시드는 **미션게임 기획서 최종 6종으로 확정**. iOS의 같은 그림 찾기·소리 퀴즈(구현 완료분)·길 찾기(목업)는 시드에서 제외 — 해당 화면·게임 코드는 제거 대상.
- **`cognitive_label`(표시 라벨, 확정)**: 미션 카드 옆에 붙는 짧은 인지 활동 라벨("기억력"·"순발력" 등). mission 객체가 포함되는 **모든 응답**(홈·미션 목록·카탈로그·추천·히스토리)에 서버가 내려준다. 라벨은 기획서 §5의 게임별 관찰 데이터(연관 인지 활동 영역 — 예: 숫자·기호=처리 속도·시각 주의 → 순발력, 신호 탭=반응 억제·지속 주의 → 집중력)에서 도출한 값이며, "이 활동이 어떤 인지 영역과 관련되는지"의 안내다 — 능력 평가·진단 표현 금지 원칙(§9) 유지.
- **일일 미션 배정은 5개 고정** — 게임 6종 풀에서 규칙/AI 추천이 5종을 선택한다. 홈 진행률 분모(`missions_total`)는 항상 5.
- iOS 게임 6종 전부 구현 완료(문항별 반응시간·힌트 계측 포함) — 일일 5개 배정을 임시 축소 없이 운영 가능. 신규 5종의 서버 성공 판정 기준·`raw_payload` 스키마 확정만 남음(부록 A-7 ①).
- `daily_checkin`은 `POST /checkins` 성공 시 서버가 자동 완료 처리한다(별도 attempt 불필요). **배정 5개·홈 진행률 분모에는 포함되지 않는** 별도 일일 활동이다.
- **보상 원칙(확정)**: 게임 1회 성공 보상은 **최소 5,000P ~ 최대 10,000P 구간**이다. 기본 보상은 **인지능력 향상 기여도(기획서 연구 연계 순위)에 비례** — 1위 7,000P부터 6위 5,000P까지 400P 단위 하향(구간·순위 비례 원칙이 정본, 개별 사다리 수치는 잠정). 10,000P에 근접하는 보상은 아래 기피 게임 보너스가 얹힐 때만 발생한다 — 상한 가중치는 "안 하려는 게임을 하게 만드는 유인" 전용이다.
- **AI 기피 게임 보너스(확정)**: 통계 로직이 게임별 기피 지표(최근 14일 배정 대비 미시작률·완료율·중도 이탈률)를 산출하고, 일일 추천 배치(Qwen, §9)가 기피 게임에 `bonus_points`를 책정해 당일 배정에 포함한다. **상한: 기본+보너스 총액 10,000P(보너스 최대 = 10,000P − 기본 보상 — 기피되기 쉬운 저보상 게임일수록 보너스 여지가 커진다), 동시 최대 2개 게임.** 보너스는 성공 완료 시에만 지급되며 원장에 `engagement_bonus`로 별도 기록된다. UI 표시: "6,200P + 보너스 1,800P".

---

## 2. 인증 / 계정 / 약관

### POST `/auth/session` — 로그인 세션 (사용자 upsert)

Firebase 로그인 직후 항상 호출. 기존 사용자면 프로필·약관 필드는 무시된다.

요청 — 최초 가입 시 필수: `role` + **실명·이메일·전화번호·성별·생년월일(수집 항목 확정)** + `consents`(필수 약관 4종). 누락 시 400. `display_name`은 선택(미지정 시 실명 사용):
```json
{
  "role": "senior",
  "real_name": "김순자",
  "display_name": "김순자",
  "email": "soonja.kim@example.com",
  "phone": "010-1234-5678",
  "birth_date": "1954-03-02",
  "gender": "female",
  "timezone": "Asia/Seoul",
  "consents": [
    {"code": "over14", "agreed": true},
    {"code": "tos", "agreed": true},
    {"code": "privacy", "agreed": true},
    {"code": "sensitive_data", "agreed": true},
    {"code": "marketing", "agreed": false}
  ]
}
```

응답 200:
```json
{
  "user_id": "019f8313-…",
  "role": "senior",
  "status": "active",
  "created": true,
  "families": [
    {"family_id": "019f831b-…", "family_name": "김씨네", "member_role": "senior", "status": "active"}
  ]
}
```

- 약관 동의는 시각·버전과 함께 서버에 이력 저장한다(민감정보 처리 근거). 선택 동의(`marketing`, `third_party_research`)는 이후 `PATCH /me`로 변경 가능. `third_party_research`는 온보딩에서 노출하지 않는 **예약 코드**다(§1.7).
- 동의 화면에 제공하는 전문의 정본: `tos` = [이용약관.md](./이용약관.md), `privacy`·`sensitive_data` = [개인정보처리방침.md](./개인정보처리방침.md)(민감정보에 준하는 정보는 방침 §3 — 별도 체크박스). `voice_diary` 동의 문구도 방침 §2·§6과 일치해야 한다.

### GET `/me` — 내 프로필 + 소속 가족

응답 200: `user_id, role, status, real_name, display_name, email, phone, birth_date, gender, timezone, families[], consents[], entitlements, avatar`

- `avatar`: `{"hair": "hair_03", "outfit": "outfit_01"}` — 아바타 설정(G-2). **얼굴은 전 사용자 공통 고정**이며 머리 모양·옷차림 코드 2개만 저장한다(개인정보 아님 — 평문 저장). 미설정 시 기본값(`hair_01`/`outfit_01`).

- `entitlements`: `{"tier": "free" | "premium" | "premium_ai"}` — 유효 티어(§11). guardian은 본인 구독, 시니어는 **같은 가족 guardian의 활성 구독이 있으면 그 티어를 상속**한다(프리미엄 교육 콘텐츠 잠금 해제용, §5).

### PATCH `/me` — 프로필·선택 동의 수정

요청(모두 선택): `real_name`, `display_name`, `email`, `phone`, `birth_date`, `gender`, `timezone`, `marketing_consent`, `voice_diary_consent`, `voice_diary_share_with_family`, `third_party_research_consent`, `avatar` → 응답은 GET `/me`와 동일.

- `avatar` 저장 시 서버 검증: 카탈로그에 없는 코드 400, **소유하지 않은 유료 아이템 코드 400**(무료 기본 세트는 항상 허용).

- `voice_diary_consent: false`로 철회 시 **기존 음성 일기 전사·파생 지표를 전부 삭제**한다(§4 — 철회 = 삭제, 비가역 안내 필요).
- `voice_diary_share_with_family`(기본 `false`, **시니어 본인만 변경 가능** — guardian이 요청하면 403): 켜면 같은 가족 guardian이 일기 원문을 열람할 수 있다(§4, 열람 감사 로그). 끄는 즉시 guardian 접근 회수.

### GET `/me/avatar/catalog` — 아바타 카탈로그 (G-2)

```json
{
  "hair":   [{"code": "hair_01", "name": "단정한 머리", "free": true,  "owned": true,  "shop_item_id": null},
             {"code": "hair_07", "name": "뽀글 파마",   "free": false, "owned": false, "shop_item_id": "…"}],
  "outfit": [{"code": "outfit_01", "name": "기본 옷",   "free": true,  "owned": true,  "shop_item_id": null},
             {"code": "outfit_05", "name": "한복",      "free": false, "owned": false, "shop_item_id": "…"}]
}
```
- 슬롯은 **`hair`(머리 모양)·`outfit`(옷차림) 2개 고정** — 얼굴·체형 커스터마이징 없음.
- 무료 기본 세트(각 슬롯 6종 내외)는 전원 `owned: true`. 유료 아이템은 상점 `deco` 상품과 매핑(`shop_item_id`) — 구매(§6) 시 `owned: true`로 전환. 미소유 아이템은 G-2에서 잠금 표시 + 탭 시 상점 이동.
- 에셋(이미지)은 앱 번들 또는 CDN — 카탈로그 `code`가 에셋 키. 신규 아이템 추가는 서버 카탈로그+에셋 배포만으로 가능(앱 강제 업데이트 없이 — 미보유 에셋 코드는 기본값 렌더링 폴백).

### DELETE `/me` — 탈퇴 → 204

soft delete + 암호화 키 폐기. 이후 모든 호출은 401.

---

## 3. 가족 (초대·연결·응원)

온보딩 플로우: 자녀가 가족 생성 → 초대 코드 발급·공유(카카오톡/복사/QR) → 시니어가 코드 입력으로 참여 → 자녀 화면은 초대 상태를 폴링해 "연결 완료" 전환.

초대 플로우는 **형제자매(guardian) 초대에도 동일하게 사용**한다 — 코드로 참여한 사용자의 `member_role`은 가입 시 확정된 `role`을 따르므로, guardian이 참여하면 guardian 구성원이 된다(F-6 "추가 연결" 재사용). 분담결제·N분의1 정산은 실결제 확정(부록 A-7 ②) 전 범위 외 — 현 단계는 충전 기여 내역 표시(§7)까지만.

### POST `/families` — 가족 생성 (guardian만, 시니어는 403) → 201
```json
// 요청
{"name": "김씨네", "relation_label": "첫째"}
// 응답
{"family_id": "…", "name": "김씨네", "created_by": "…"}
```

### POST `/families/{family_id}/invites` — 초대 코드 발급 → 201
```json
{"invite_id": "…", "code": "7Q9K", "qr_payload": "ddkyd://join?code=7Q9K", "expires_at": "2026-07-22T05:17:26Z"}
```
- 코드는 **4자리 영대문자+숫자**(혼동 문자 0/O/1/I/L 제외), TTL 24시간, 1회용. 시니어 입력 UI(4칸)와 일치 — 자녀 화면의 6자 하드코딩 표시("7Q9K2M")는 4자로 수정 필요.
- 4자리는 조합 수가 작으므로 `POST /families/join` 코드 검증에 **레이트리밋 필수**(계정·IP당 10회/10분, 초과 시 429) + 실패 누적 시 해당 초대 무효화.
- **평문 코드는 이 응답에만 존재** — 화면 표시·공유 후 폐기(로컬 영속 저장 금지).

### GET `/families/{family_id}/invites/{invite_id}` — 초대 상태 확인 (자녀 대기 화면 폴링)
```json
{"invite_id": "…", "status": "used", "joined_member": {"user_id": "…", "display_name": "김순자", "member_role": "senior"}}
```
- `status`: `pending` | `used` | `expired`. 폴링 간격 3~5초 권장.

### POST `/families/join` — 코드로 참여
```json
// 요청
{"code": "7Q9K", "relation_label": "어머니"}
// 응답
{"family_id": "…", "family_name": "김씨네", "member_role": "senior", "status": "active"}
```
- 오류: 404(잘못된 코드) / 400(만료·사용됨) / 409(이미 구성원).

### GET `/families/{family_id}/members`

F-6 부모님 프로필·S-10 마이페이지의 연결 정보 원천.
```json
{"family_id": "…", "members": [
  {"user_id": "…", "member_role": "senior", "status": "active",
   "display_name": "김순자", "relation_label": "어머니", "joined_at": "2026-03-12T…",
   "avatar": {"hair": "hair_03", "outfit": "outfit_01"}}
]}
```
- 한 guardian이 여러 시니어(어머니·아버지)와 연결 가능 — 대시보드 상단 부모 선택 칩은 이 목록의 `member_role == "senior"` 항목으로 구성.

### DELETE `/families/{family_id}/members/{user_id}` → 204
본인 탈퇴 또는 guardian에 의한 해제.

### POST `/families/{family_id}/cheers` — 음성 응원 편지 보내기 (guardian만) → 201

자녀가 **똑똑이를 통해 음성으로** 부모님께 보내는 응원 편지. 자녀 기기에서 녹음 + on-device STT 전사 후 함께 업로드한다. 부모님 홈의 **우체통**이 흔들리며 도착을 알리고, 열면 편지지 UI에 전사 편지가 펼쳐지고 음성이 재생된다(S-21).

요청 `multipart/form-data`:

| 필드 | 내용 |
|---|---|
| `senior_id` | 수신 시니어 |
| `audio` | 음성 파일 — **AAC(m4a), 최대 60초·5MB** (형식·길이·크기 서버 검증, 초과 400). **선택(권장)** — iOS는 항상 첨부한다(음성이 이 기능의 본질). 미첨부 시 전사만 편지로 수신되며, 이는 장애·폴백 경로다(S-21은 `audio_url: null`이면 재생 버튼 없이 렌더링) |
| `transcript` | 자녀 기기 on-device STT 전사, ≤ 1,000자 (편지지 텍스트) — 필수 |
| `visibility` | `named` \| `anonymous` (§1.7) |

```json
// 응답 201
{"cheer_id": "…", "senior_id": "…", "visibility": "named",
 "sent_at": "…", "expires_at": "2026-07-22T…", "retention": "24h"}
```
- **보존(§1.9 특칙)**: 편지의 보존 상태는 **`24h` 또는 `permanent` 둘 중 하나뿐이며, 발송 시점의 발신자 티어로 확정되어 이후 변하지 않는다.** 무료 발신 → `retention: "24h"`(24시간 후 오디오·전사 자동 파기, 사후 보관 전환 불가). 발송 시점에 발신 guardian이 premium 이상 → `retention: "permanent"`, `expires_at: null` — **이후 구독을 해지해도 영구 보관이 유지된다**(§11 혜택).
- 오디오는 오브젝트 스토리지에 **암호화 저장**, 접근은 단기 서명 URL(발급 후 15분)로만. 전사는 DEK 암호화 저장, LLM 입력 제외.
- **분석 금지**: 응원 오디오·전사는 발화 특징·정서 분석 등 어떤 프로파일링 파이프라인에도 투입하지 않는다 — 전달 콘텐츠일 뿐이다. **유일한 예외는 아래 안전 필터.**
- **안전 필터(콘텐츠 모더레이션, 발송 시점 1회)**: 서버가 `transcript`를 금칙어·패턴 사전(욕설·모욕·비하 / 위협·협박 / 위험 행위 조장 / 금전 요구·사기 의심 표현)으로 검사한다 — **서버 내 코드 기반, 외부 AI 미전송, 오디오는 검사하지 않음**. 감지 시 **400 거부(오디오·전사 저장하지 않음)**, problem `detail`은 사유 카테고리만 담는다. 필터 로그는 통과/차단 여부만 기록(문구 원문 미기록). 클라이언트는 전사 미리보기 단계에서 로컬 사전으로 1차 검사해 발송 전에 안내한다(서버 판정이 정본).
- `anonymous`는 수신 화면에서 "익명의 응원"으로 표시될 뿐, 서버는 발신자를 기록한다(§1.9 특칙 ⑤).
- 수신 측 노출: 홈 `mailbox`(§4) + 알림함 `type: "family"`("응원 편지가 도착했어요" — 익명이면 발신자명 미포함).
- 레이트리밋: **guardian당 시니어별 3회/일** — 초과 429. 시니어 호출·비가족 호출 403.

### GET `/seniors/{senior_id}/cheer-letters?cursor=&limit=20` — 편지함 (S-21)

수신 시니어 본인 전용(guardian은 자기 발신분만 필터 조회 가능 — `?sent=me`).

```json
{"letters": [
  {"cheer_id": "…", "from_display_name": "유정민", "visibility": "named",
   "transcript": "엄마, 오늘도 화이팅이에요! …", "audio_url": "https://…(15분 서명 URL)",
   "listened": false, "sent_at": "…", "expires_at": null, "retention": "permanent"}
], "next_cursor": null}
```
- `visibility: "anonymous"`면 `from_display_name: null`(UI는 "익명의 응원" 표시).
- `audio_url: null`이면 전사만 수신된 편지(발신 시 오디오 미첨부 폴백) — 재생 버튼 없이 렌더링한다.
- 만료(24h) 지난 편지는 목록에서 제외된다(오디오·전사는 파기 완료 상태).

### POST `/seniors/{senior_id}/cheer-letters/{cheer_id}:listened` — 청취 처리 → 200

우체통 흔들림 해제용. 응답 `{"listened": true}`. 수신 시니어 본인만.

---

## 4. 시니어 참여 (홈·미션·체크인·음성 일기·달력)

### GET `/seniors/{senior_id}/home` — **홈 1콜 (S-1)**

홈 화면 전체를 한 번에 내려준다. 서버 캐시 30초(쓰기 후 무효화).

```json
{
  "date": "2026-07-21",
  "display_name": "김순자",
  "featured_mission": {
    "assignment_id": "…", "status": "assigned", "source": "ai", "rank": 1, "bonus_points": 1800,
    "mission": {"mission_id": "…", "code": "sequence_recall", "mission_type": "game",
                "title": "순서 기억하기", "emoji": "🧠", "difficulty": 2,
                "cognitive_domain": "memory", "cognitive_label": "기억력",
                "reward_points": 6200, "estimated_seconds": 180}
  },
  "missions_total": 5,
  "missions_completed": 3,
  "checkin": {"done": false, "reward_points": 1000},
  "voice_diary": {"enabled": true, "done": false},
  "extra_activities": [
    {"kind": "mission", "code": "digit_symbol", "emoji": "🔢", "title": "숫자·기호 짝맞추기", "meta": "인지 게임 · 약 3분", "reward_points": 7000},
    {"kind": "edu", "content_id": "…", "emoji": "📖", "title": "오늘의 건강 상식", "meta": "교육 콘텐츠 · 약 2분", "reward_points": 1500}
  ],
  "balance": 84500,
  "weekly_goal": {"week_start": "2026-07-20", "target_points": 150000, "earned_points": 98000},
  "streak_days": 2,
  "mailbox": {"unread_count": 1, "latest_from_display_name": "유정민", "latest_sent_at": "…"}
}
```
- `featured_mission`: 홈 "오늘의 미션" 카드(추천 rank 1).
- `mailbox`: 음성 응원 편지 우체통(§3) — `unread_count > 0`이면 우체통 흔들림 애니메이션, 탭 시 편지함(S-21) 진입. 익명 편지만 있으면 `latest_from_display_name: null`.
- `voice_diary`: 똑똑이 음성 일기 상태 — `enabled`는 `voice_diary` 동의 여부(false면 홈 똑똑이 탭 시 동의 안내), `done`은 오늘 참여 여부.

### GET `/seniors/{senior_id}/missions/today` — 오늘 미션 목록 (S-3 "오늘의 미션" 탭)
```json
{"date": "2026-07-21", "missions": [AssignedMission…]}
```
- 각 항목에 `emoji`, `estimated_seconds`, `reward_points`, `cognitive_label`(인지 활동 라벨 — 카드 옆 칩으로 표시), assignment `status`·`bonus_points`(기피 게임 보너스, 기본 0) 포함 — 카드의 "완료" 체크 표시는 `status == "completed"`, 보너스가 있으면 "+보너스 n,nnnP" 배지 표시.
- "인지 게임" 탭은 미션 카탈로그 전체가 필요: `GET /missions?type=game` (code·이름·규칙 문구 `rule`·보상 포함).

### POST `/seniors/{senior_id}/missions/{mission_id}/attempts` — 미션 시작 → 201
```json
{"attempt_id": "…", "assignment_id": "…", "mission_id": "…", "started_at": "…"}
```
- 오늘 배정되지 않은 미션이면 404. **이미 성공 완료(assignment `completed`)한 미션이면 409.** 실패 후 재도전은 새 attempt 생성 허용.
- 시작 후 당일 자정(Asia/Seoul)까지 미제출 attempt는 서버가 `abandoned` 처리.

### POST `/attempts/{attempt_id}/complete` — 결과 제출 (멱등)

성공/실패 판정은 **서버가 `raw_payload`로 재계산**한다(클라이언트 하드코딩 제거 — 판정 기준은 아래 표).

```json
// 요청
{
  "idempotency_key": "B2F1A6E4-…",
  "duration_ms": 65000,
  "reaction_time_ms_avg": 2100,
  "hint_count": 1,
  "raw_payload": {
    "version": 2,
    "game": "sequence_recall",
    "rounds": [
      {"length": 3, "passed": true,  "mean_rt_ms": 1500, "hints": 0},
      {"length": 4, "passed": true,  "mean_rt_ms": 2100, "hints": 0},
      {"length": 5, "passed": false, "mean_rt_ms": 2700, "hints": 1}
    ]
  }
}
// 응답 200
{
  "attempt_id": "…", "status": "completed",
  "success": true,
  "score": 92,
  "accuracy": 0.67,
  "earned_points": 8000,
  "bonus_points": 1800,
  "balance": 92500,
  "today": {"missions_completed": 4, "missions_total": 5},
  "weekly_goal": {"target_points": 150000, "earned_points": 106000},
  "replayed": false
}
```

게임별 `raw_payload`와 서버 판정 기준(현재 iOS 클라이언트 로직을 서버 정본으로 이관):

| game | raw_payload 핵심 필드 | 성공 기준 |
|---|---|---|
| `sequence_recall` | `rounds[{length, passed, mean_rt_ms, hints}]` (3라운드) | 통과 라운드 ≥ 2 |
| `digit_symbol` · `spot_difference` · `signal_tap` · `word_sort` · `daily_order` | 기획서 §5 저장 필드를 snake_case로 매핑 (`problem_count`, `correct_count`, `error_count`, `mean_rt_ms`, `hints`, `retries`, `skipped` …) | 구현 시 확정 (부록 A-7 ①) |

- **`success: false`면 `earned_points: 0`, assignment는 `assigned` 유지(재도전 가능).** 실패 attempt도 기록되어 대시보드 지표에 반영된다.
- `earned_points`는 **기본 보상 + 기피 게임 보너스 합계**, `bonus_points`는 그중 보너스 몫(성공 시에만 지급). 원장에는 `mission_complete`와 `engagement_bonus` 두 엔트리로 기록된다.
- `reaction_time_ms_avg`·`hint_count`는 **필수 필드**(확정) — iOS 전 게임이 문항별 반응시간 기록과 힌트 동작을 구현해 전송한다. 서버는 `raw_payload`의 문항별 `mean_rt_ms`·`hints`로 재계산해 클라이언트 값을 덮어쓴다.
- 크기 제약: `raw_payload` **32KB·문항 50개 상한**(초과 400), 전역 요청 본문 **256KB**(초과 413). 터치 좌표·센서 원본 스트림 전송 금지 — 문항 단위 요약만.
- 응답의 `today`·`weekly_goal`로 S-7 보상 화면의 "오늘 목표 4/5" 진행 표시를 갱신한다(하드코딩 제거).
- 이미 성공 완료된 attempt에 **다른 키**로 재요청 시 409 → 큐에서 제거.

### POST `/seniors/{senior_id}/checkins` — 일일 체크인 (S-2, 하루 1회 upsert)

iOS 체크인은 기분(5단계)·수면 품질(3단계) 2스텝. 수면은 시간이 아닌 **품질 코드**로 수집한다.

```json
// 요청
{"mood": 4, "sleep_quality": 3, "note_text": null}
// 응답 200
{"checkin_id": "…", "checkin_date": "2026-07-21", "created": true, "earned_points": 1000, "balance": 85500}
```
- `mood` 1~5 (😣힘들어요=1 … 😄아주 좋아요=5), `sleep_quality` 1~3 (잘 못 잤어요=1, 보통=2, 푹 잤어요=3), `note_text` ≤ 500자(선택 — 임의 개인정보 입력 가능성이 있어 **서버에서 암호화 저장**, LLM 입력 제외).
- 보상 **+1,000P는 최초 1회만**. 같은 날 재호출은 값 덮어쓰기(`created: false`, `earned_points: 0`).
- 성공 시 `daily_checkin` 미션 자동 완료.

### POST `/seniors/{senior_id}/voice-diaries` — 똑똑이 음성 일기 (S-20, 하루 1회 upsert)

"오늘 하루 어땠는지 똑똑이에게 들려주세요" — 시니어가 똑똑이를 눌러 음성으로 이야기하면 **기기 내(on-device) STT**가 전사하고, 앱은 **전사 텍스트 + 발화 특징 지표만** 전송한다. **음성 원본 업로드 API는 존재하지 않는다**(§1.9 특칙). `voice_diary` 동의(§1.7) 없으면 403.

```json
// 요청
{
  "transcript": "오늘은 복지관에 가서 친구들을 만났고…",
  "speech_metrics": {
    "version": 1,
    "duration_ms": 47000,
    "word_count": 138,
    "speech_rate_wpm": 176,
    "pause_count": 9,
    "pause_ms_avg": 800,
    "restart_count": 3,
    "stt_confidence_avg": 0.91
  }
}
// 응답 200
{"diary_id": "…", "diary_date": "2026-07-21", "created": true, "replayed": false}
```
- `transcript` ≤ 4,000자. **DEK 암호화 저장**. **보존(확정)**: 시니어의 유효 티어가 `free`면 30일 경과 시 원문 파기·파생 지표만 보존, `premium` 이상(가족 guardian 구독 상속)이면 영구 보존 — 일 배치가 파기를 수행한다.
- `speech_metrics`는 기기 STT 타임스탬프에서 산출한 요약 수치만(터치·오디오 원본 스트림 금지 — attempt `raw_payload`와 동일 원칙). 서버 분석 지표(반복·어휘 다양성·조리성 신호 등)는 일 배치가 전사에서 별도 산출한다(§9).
- 같은 날 재호출은 덮어쓰기(`created: false`) — 체크인과 동일한 `(시니어, 날짜)` upsert.
- 참여 보상 지급 여부·포인트는 결정 대기(부록 A-7 ⑦) — 확정 전 응답에 `earned_points` 없음.

### GET `/seniors/{senior_id}/voice-diaries?cursor=&limit=20` — 일기 목록
```json
{"entries": [
  {"diary_id": "…", "diary_date": "2026-07-21", "transcript": "…", "created_at": "…"}
], "next_cursor": null}
```
- 시니어 본인: 항상 열람 가능.
- **guardian 호출: 시니어의 `voice_diary_share_with_family`가 켜진 경우에만 허용**(열람마다 감사 로그) — 꺼져 있으면(기본값) 403(§1.9 특칙 ③). 공유 여부와 무관하게 가족에게는 대시보드 참여 표시·주간 리포트의 정서/발화 변화 요약이 제공된다(§8·§9).

### DELETE `/seniors/{senior_id}/voice-diaries/{diary_id}` → 204 (**시니어 본인만**)
- 원문·해당 건 파생 지표 즉시 삭제. `voice_diary` 동의 철회(PATCH /me) 시 전체 삭제.

### GET `/seniors/{senior_id}/activity-calendar?month=2026-07` — 활동 달력 (월간)

시니어(내 참여 기록)와 guardian(부모님 활동 한눈에 보기)이 같은 엔드포인트를 사용한다. **신규 화면** — 현재 iOS 목업에 없음.

```json
{
  "month": "2026-07",
  "days": [
    {"date": "2026-07-01", "status": "complete",
     "missions_completed": 5, "missions_total": 5,
     "checked_in": true, "earned_points": 32000},
    {"date": "2026-07-02", "status": "partial",
     "missions_completed": 2, "missions_total": 5,
     "checked_in": true, "earned_points": 13400},
    {"date": "2026-07-03", "status": "none",
     "missions_completed": 0, "missions_total": 5,
     "checked_in": false, "earned_points": 0}
  ],
  "summary": {"active_days": 18, "perfect_days": 6, "total_points": 412000, "longest_streak": 7}
}
```
- `status` 판정(달력 셀 스탬프 렌더링용): `complete` = 배정 5개 전부 완료 / `partial` = 미션 1개 이상 완료 또는 체크인 / `none` = 활동 없음.
- **프리미엄 게이트(§11)**: guardian 호출(F-12)은 `premium` 이상 필요 — 미달 402. 시니어 본인(S-19)은 게이트 없음.
- `days`는 해당 월 1일부터 **오늘까지만** 포함(미래 날짜 없음). 가입 이전 월 요청 시 `days: []`.
- `month`는 시니어 타임존(Asia/Seoul) 기준. 서버 캐시 5분(당일 셀은 쓰기 후 무효화).
- `summary.active_days` = `partial` 이상인 날 수, `perfect_days` = `complete`인 날 수, `longest_streak` = 해당 월 내 최장 연속 활동일.

### GET `/seniors/{senior_id}/activity-calendar/{date}` — 일별 활동 상세 (달력 셀 탭)

```json
{
  "date": "2026-07-21",
  "missions": [
    {"code": "sequence_recall", "emoji": "🧠", "title": "순서 기억하기",
     "status": "completed", "score": 92, "earned_points": 6200, "completed_at": "…"}
  ],
  "checkin": {"done": true, "mood": 4, "sleep_quality": 3},
  "earned_points_total": 7200
}
```
- 그날 배정된 미션 5개 전체가 `status`와 함께 내려온다(미완료는 `assigned`·`score: null`).

---

## 5. 교육 콘텐츠 (S-3 교육 탭 · S-15 상세)

### GET `/edu-contents` — 목록
```json
{"contents": [
  {"content_id": "…", "emoji": "🩺", "title": "혈압, 이렇게 관리해요", "format": "article",
   "estimated_seconds": 120, "reward_points": 1500, "completed": false, "premium": false}
]}
```
- `format`: `article` | `video`. `completed`는 요청자(시니어) 기준.
- `premium: true` 콘텐츠는 유효 티어(§2 `entitlements` — 시니어는 가족 guardian 구독 상속)가 `premium` 이상일 때만 본문 열람·완료 가능. 목록에는 항상 노출(프리미엄 배지 + 잠금 UI).

### GET `/edu-contents/{content_id}` — 상세
```json
{"content_id": "…", "title": "…", "format": "article",
 "body_md": "## 혈압 관리\n\n…", "video_url": null, "reward_points": 1500, "completed": false, "premium": false}
```
- 본문은 서버 제공(`body_md` 마크다운 또는 `video_url`). 현재 iOS의 하드코딩 본문 제거 대상.
- `premium` 콘텐츠를 무료 티어가 조회하면 402(본문 미포함) — 상세·완료 모두 서버 게이트(§11).

### POST `/edu-contents/{content_id}/complete` — 완료 제출 (멱등)
```json
// 요청
{"idempotency_key": "…"}
// 응답 200
{"content_id": "…", "earned_points": 1500, "balance": 86000, "replayed": false}
```
- 콘텐츠당 보상 1회. 재완료 시 `earned_points: 0`. `premium` 콘텐츠는 유효 티어 미달 시 402.

---

## 6. 보상 — 시니어 지갑·출금·상점

포인트 경제(**1P = 1원 고정, 확정**):
**자녀가 가족 지갑에 충전(§7) → 시니어가 미션으로 적립 → 시니어가 출금(데모) 또는 상점 사용.**

### GET `/seniors/{senior_id}/wallet` — 용돈함 (S-8)
```json
{
  "balance": 84500,
  "krw_value": 84500,
  "withdrawable": true,
  "weekly_goal": {"week_start": "2026-07-20", "target_points": 150000, "earned_points": 98000}
}
```

### GET `/seniors/{senior_id}/points/ledger?cursor=&limit=20` — 적립·사용 내역
```json
{
  "entries": [
    {"id": "…", "delta": 6200, "balance_after": 84500, "reason_type": "mission_complete",
     "emoji": "🧠", "title": "순서 기억하기 완료", "ref_type": "mission_attempt", "created_at": "…"}
  ],
  "next_cursor": null
}
```
- `emoji`·`title`은 서버가 렌더링해 내려준다(iOS 목록 그대로 표시). "오늘/어제" 상대 표기는 클라이언트가 `created_at`으로 계산.

### GET `/seniors/{senior_id}/weekly-goal` — 주간 목표·보상 조건 (S-17)
```json
{
  "week_start": "2026-07-20",
  "target_points": 150000,
  "earned_points": 98000,
  "conditions": [
    {"type": "streak", "label": "3일 연속 참여", "achieved": true, "bonus_points": 5000,
     "progress": {"current": 3, "target": 3}},
    {"type": "weekly_points", "label": "주간 목표 150,000P", "achieved": false,
     "progress": {"current": 98000, "target": 150000}}
  ]
}
```
- 목표·보너스 수치는 guardian이 설정한 보상 규칙(§7 reward-rules)에서 파생.

### POST `/seniors/{senior_id}/withdrawals` — 출금 신청 (S-12, 데모·멱등)
```json
// 요청
{"idempotency_key": "…", "amount_krw": 50000}
// 응답 201
{"withdrawal_id": "…", "amount_krw": 50000, "fee_krw": 500, "status": "requested",
 "account": {"bank": "국민은행", "number_masked": "···123", "holder": "김순자"},
 "expected": "1~2 영업일 이내 입금", "balance": 34500}
```
- MVP: 실이체 없이 신청 상태만 관리(`requested` → 운영 처리). 잔액 부족 시 400. 계좌 등록/변경 API는 결정 대기(부록 A-7).
- **실출금 전환의 전제(확정 원칙 — 보안 문서.md §4.7)**: ⓐ 출금 계좌는 **시니어 본인 명의만**(1원 인증/계좌실명조회로 가입 실명 대조 — 검증 없이 실출금 출시 금지) ⓑ 출금 일/월 한도 적용 ⓒ 신규 가족 연결 후 냉각 기간·계좌 변경 후 48시간 보류 ⓓ 선불전자지급수단 해당 여부·전금법 등록·AML 검토 선행. 한도 수치는 부록 A-7 ⑩.
- `GET /seniors/{senior_id}/withdrawals` — 신청 내역 목록.

### GET `/shop/items` — 상점 목록 (S-13)
```json
{"items": [
  {"item_id": "…", "emoji": "☕", "name": "카페 기프티콘", "description": "5,000원권",
   "price_points": 5000, "category": "gifticon", "fulfillment_type": "demo",
   "partner_id": null, "purchased": false}
]}
```
- `price_points: 0` = 무료. `category`(§1.7): `deco` | `theme` | `widget` | `gifticon` | `edu`(교육 콘텐츠 연계 상품) | `health`(건기식·보청기 등 시니어 특화) | `partner`(제휴 상품).
- **`deco` 상품 중 아바타 아이템**은 `avatar_slot`(`hair`|`outfit`)·`avatar_code` 필드를 가진다 — 구매 시 아바타 카탈로그(§2)에서 `owned: true`로 전환되어 G-2에서 착용 가능. 아바타 아이템은 계정 귀속(선물 가능 — recipient의 소유로 등록).
- **제휴 상품·광고 지면은 별도 광고 시스템 없이 `partner` 카테고리 상품으로 흡수**한다 — `partner_id`는 제휴사 식별자(비제휴 상품은 `null`). S-13 상점 그리드에 카테고리 섹션으로 노출.
- `fulfillment_type`(§1.7): `demo`(상태 관리만) | `api`(기프티콘 발급 API 연동) | `physical`(실물 배송). **실발급·실배송은 실결제 범위 확정(부록 A-7 ②) 전까지 전 상품 `demo` 고정.**

### POST `/shop/items/{item_id}/purchase` — 구매·선물 (멱등)
```json
// 요청 (recipient_user_id 없으면 본인 사용)
{"idempotency_key": "…", "recipient_user_id": "…"}
// 응답 200
{"purchase_id": "…", "item_id": "…", "price_points": 5000, "balance": 29500,
 "recipient": {"user_id": "…", "display_name": "유정민"},
 "fulfillment": {"type": "demo", "status": "demo"}, "replayed": false}
```
- 잔액 부족 400, 중복 구매 불가 아이템 재구매 409. 기프티콘 실발급은 MVP 범위 외(데모 처리, 부록 A-7 ②) — `fulfillment.status`는 `demo` 외 값(`issued`·`shipped` 등)을 실발급 도입 시 확정한다.
- **가족 선물하기**: `recipient_user_id`에 **같은 가족 구성원**을 지정하면 선물 구매가 된다(비가족 403). 재원은 호출자 역할로 결정 — **시니어 = 본인 지갑 포인트**(미션으로 모은 용돈으로 자녀·손주에게 선물), **guardian = 가족 지갑 충전 잔액**(§7 `charged_balance`) 차감. 수령인에게 `type: "family"` 알림("가족이 선물을 보냈어요") 발송.
- 현물(`gifticon`·`health`·`partner`) 선물이 B2B 제휴 커머스의 접점이다 — 제휴 상품을 가족 간 선물로 순환시키는 구조. **실물 배송 주소는 수집하지 않는다**(§1.9 — demo 이행 한정. 실배송 도입 시 주소 수집 보안 검토 선행, 부록 A-7 ⑧).
- **현금성 선물 유속 제한(확정 원칙 — 보안 문서.md §4.7 M-3)**: 실발급 전환 시 `gifticon`·`health`·`partner` 카테고리의 선물 구매에 **월 한도(소액)**를 적용한다 — 충전→선물→현금화("깡")가 주간 보상 상한을 우회하는 유일한 대량 경로이기 때문. 비현금성(`deco`·`theme`·`widget`)은 제외. 신규 가족 연결 후 냉각 기간에는 현금성 선물 불가. 수치는 부록 A-7 ⑩.

### GET `/families/{family_id}/gifts?cursor=&limit=20` — 가족 선물 내역

가족 구성원 공용(구성원 외 403).
```json
{"gifts": [
  {"gift_id": "…", "item": {"name": "카페 기프티콘", "emoji": "☕", "category": "gifticon"},
   "from_display_name": "김순자", "to_display_name": "유정민",
   "fulfillment": {"type": "demo", "status": "demo"}, "created_at": "…"}
], "next_cursor": null}
```

---

## 7. 보상 — 가족 지갑·충전·규칙 (guardian 화면)

### GET `/families/{family_id}/reward-wallet?senior_id={sid}` — 충전 잔액·매칭 현황 (F-4·F-8)
```json
{
  "charged_balance": 120000,
  "monthly_matched": {"amount_krw": 12400, "target_krw": 20000, "month": "2026-07"},
  "point_to_krw_rate": 1.0
}
```
- `charged_balance`: 자녀가 충전해 둔 재원(P). 시니어 적립 포인트는 이 잔액에서 차감된다.
- `monthly_matched`: 이번 달 시니어에게 지급된 누적액/목표(진행바 렌더링용).

### POST `/families/{family_id}/reward-wallet/charges` — 포인트 충전 (인앱 결제, 멱등)
```json
// 요청
{"idempotency_key": "…", "product_id": "chg_100000", "transaction_id": "<StoreKit2 트랜잭션 ID>", "amount_krw": 100000}
// 응답 201
{"charge_id": "…", "status": "completed", "amount_krw": 100000, "charged_balance": 220000}
```
- 서버가 App Store 영수증(트랜잭션)을 검증 후 적립. MVP는 `status: "demo"`로 결제 없이 시뮬레이션 가능(부록 A-7).
- 상품: 30,000 / 50,000 / 100,000 / 300,000 / 600,000 / 1,000,000원 6종(`chg_30000` …).
- 충전 건에는 **충전자(`charged_by`)가 기록**된다 — 형제자매 여러 guardian이 충전하는 가족에서 기여 내역 표시의 원천.
- **실결제 전환 시 충전 월 한도**(가족당·충전자당)를 적용한다(보안 문서.md §4.7 M-3 — 수치는 부록 A-7 ⑩). 미사용 충전 포인트의 환불은 **원결제수단으로만**(제3자 계좌 환불 경로 없음, M-2).

### GET `/families/{family_id}/reward-wallet/charges?cursor=&limit=20` — 충전 내역 (F-4·F-8)
```json
{
  "entries": [
    {"charge_id": "…", "amount_krw": 100000, "status": "completed",
     "charged_by": {"user_id": "…", "display_name": "유정민", "relation_label": "첫째"},
     "created_at": "…"}
  ],
  "next_cursor": null
}
```
- 가족 구성원 누구나 조회 가능(guardian 화면 기준). "누가 얼마 충전했는지"를 F-4 용돈 관리·F-8 충전 화면에 노출한다.

### GET · PUT `/families/{family_id}/reward-rules?senior_id={sid}` — 보상 규칙 (F-4·F-10, guardian만 PUT)
```json
{
  "senior_id": "…",
  "weekly_goal_points": 150000,
  "goal_bonus": {"enabled": true, "bonus_points": 5000},
  "streak_bonus": {"enabled": true, "days": 3, "bonus_points": 5000},
  "weekly_cap": {"enabled": true, "cap_points": 150000},
  "monthly_target_krw": 20000
}
```
- v1의 "용돈 플랜(monthly_amount·환산율·cycle)"을 대체한다. 시니어 S-17 주간 목표 화면과 같은 데이터 원천.
- 보너스 포인트는 충전 잔액에서 자동 지급. PUT 시 유효 범위: `weekly_goal_points` 20,000~200,000(10,000 단위).
- **환산율은 1P=1원으로 확정** — `point_to_krw_rate`는 항상 `1.0`(응답 호환용 상수). 자녀 보상 관리 화면(F-4)의 "포인트 단가 100P당 N원" 조절 UI는 제거 대상.

---

## 8. 가족 대시보드 (guardian 화면)

> ⚠️ guardian 앱은 이 섹션 응답을 **디스크에 영속화하지 않는다**(메모리 캐시만).

### GET `/families/{family_id}/dashboard?senior_id={sid}` — **대시보드 1콜 (F-1)**

시니어 홈과 동일하게 guardian 홈도 1콜로 구성한다. 서버 캐시 60초.

```json
{
  "senior": {"senior_id": "…", "display_name": "김순자", "relation_label": "어머니",
             "avatar": {"hair": "hair_03", "outfit": "outfit_01"}},
  "care_state": {
    "level": "safe",
    "message": "이번 주 특별한 변화가 없어요 :)",
    "sub": "평소와 비슷한 활동 패턴을 유지하고 있어요"
  },
  "week": {"participation_rate": 0.86, "delta_ratio": 0.08},
  "today": {"missions_completed": 3, "missions_total": 5},
  "last_active_at": "2026-07-21T02:11:00Z",
  "recent_activities": [
    {"type": "mission", "emoji": "🧠", "title": "순서 기억하기 완료", "meta": "정답률 92% · 빠른 반응", "occurred_at": "…"},
    {"type": "checkin", "emoji": "☀️", "title": "오늘 기분 체크인", "meta": "\"좋아요\" 선택", "occurred_at": "…"}
  ]
}
```
- `care_state.level`: `safe`(안심) | `caution`(주의) | `alert`(확인필요) — 이상 신호 규칙(참여 급감·정답률 연속 하락·반응 지연)을 **통계 로직(코드)으로 산출**(AI 미사용, §9 표). `message`·`sub`는 서버 렌더링 문구. **진단이 아닌 관찰 신호** — 비진단 톤 유지.
- `last_active_at`: 유저스토리(자녀 US1)의 "최근 활동 시각". `recent_activities`는 최근 10건.

### GET `/seniors/{senior_id}/metrics/daily?metric=accuracy&days=7` — 지표별 시계열 (F-2 차트)
```json
{
  "metric": "accuracy",
  "series": [
    {"date": "2026-07-15", "value": 0.78},
    {"date": "2026-07-16", "value": 0.84}
  ],
  "summary": {"current": 0.86, "delta_ratio": 0.08, "avg_reaction_ms": 1800, "week_participation_count": 18}
}
```
- `metric`: `accuracy`(정답률 0~1) | `reaction_time`(ms) | `participation`(일별 완료 수). 세 지표 각각 호출(차트 탭 전환 시).
- `days`: 7 | 30. 미참여일은 `value: null`.
- **프리미엄 게이트(§11)**: guardian 호출 시 `days=30`은 `premium` 이상 필요(무료는 7일만) — 미달 402. 시니어 본인 호출은 게이트 없음.

### GET `/seniors/{senior_id}/pattern-summary` — 참여 패턴 요약 (F-9)
```json
{
  "stats": [
    {"key": "frequency", "label": "참여 빈도", "value": "주 5.2회", "note": "지난주보다 늘었어요", "tone": "good"},
    {"key": "favorite", "label": "선호 활동", "value": "기억 게임", "note": "가장 자주 참여해요", "tone": "good"},
    {"key": "reaction", "label": "평균 반응시간", "value": "1.8초", "note": "안정적으로 유지 중", "tone": "good"},
    {"key": "afternoon", "label": "오후 활동량", "value": "낮은 편", "note": "오전보다 참여 적어요", "tone": "warn"}
  ],
  "care_points": [
    {"tone": "good", "text": "수면 관련 체크인 응답이 안정적이에요"},
    {"tone": "warn", "text": "오후 활동량이 오전보다 낮은 편이에요"}
  ]
}
```
- `afternoon` 지표를 위해 서버는 attempt `started_at`의 **시간대(오전/오후) 집계**를 유지한다.
- 패턴 통계·`care_points` 문구는 **통계 로직(코드)** 산출 — 외부 AI 호출 없음(§9 표).
- **프리미엄 게이트(§11)**: guardian 호출은 `premium` 이상 필요 — 미달 402.

### GET `/seniors/{senior_id}/missions/{mission_id}/history?cursor=&limit=20` — 미션 회차 기록 (F-7)
```json
{
  "mission": {"mission_id": "…", "code": "sequence_recall", "title": "순서 기억하기", "cognitive_label": "기억력"},
  "summary": {"avg_score": 86, "total_count": 18},
  "entries": [
    {"attempt_id": "…", "date": "2026-07-21", "score": 92, "success": true, "comment": "빠른 반응, 오류 없음"}
  ],
  "next_cursor": null
}
```
- `comment`는 결과 제출 시 활동 데이터 구조화 단계(**Gemini 3.5 Flash**, §9 표)가 생성해 저장한 한 줄 요약 — 식별 정보 없이 활동 데이터만 전송(§1.9). v1에서 누락됐던 attempt 단위 이력 조회를 이 엔드포인트가 담당.

### GET `/families/{family_id}/care-alerts?senior_id={sid}&cursor=` — 케어 알림 (F-5)
```json
{"alerts": [
  {"alert_id": "…", "severity": "alert", "emoji": "⚠️", "title": "3일 연속 게임 정답률이 평소보다 낮아요", "occurred_at": "…"},
  {"alert_id": "…", "severity": "caution", "emoji": "📉", "title": "오후 활동량이 지난주보다 줄었어요", "occurred_at": "…"},
  {"alert_id": "…", "severity": "info", "emoji": "✅", "title": "오늘 미션 3개를 완료했어요", "occurred_at": "…"}
], "next_cursor": null}
```
- `severity` → 태그 매핑: `alert`=확인 필요 / `caution`=주의 / `info`=정상.
- **프리미엄 게이트(§11)**: 무료 티어는 **최신 5건만**(`next_cursor: null` 고정), `premium` 이상은 전체 히스토리 페이지네이션.

**케어 알림 생성 룰(확정 — 일 배치 통계 로직, AI 미사용)**:

| 룰 | 조건 | 생성 |
|---|---|---|
| **무활동** | "무활동일" = 시니어 타임존 기준 그날 미션 완료·체크인·음성 일기 **모두 없음**. 연속 2일 | `caution` "이틀째 활동이 없어요" |
| | 연속 3일 | **`alert` "3일째 활동이 없어요 — 안부 전화 어떠세요?" + 푸시 발송(§10)**. 지속 시 3일 간격 재알림(6일·9일…, 매일 발송 금지) |
| | 활동 재개 | 카운터 리셋 + `info` "오늘 다시 활동을 시작했어요" |
| 이상 신호 | 정답률 연속 하락·평균 반응시간 지연·오후 활동량 감소 (케어 상태 카드 §8 상단과 동일 룰 공유) | `caution`/`alert` — 임계값은 서버 설정값(운영 튜닝, 코드 배포 없이 조정) |

- **케어 알림의 생성과 푸시 발송은 구독 티어와 무관하다**(안전 기능 — 과금 게이트 금지). 프리미엄 게이트는 과거 히스토리 열람에만 적용되며, `alert`급 무활동 알림은 무료 티어의 "최신 5건" 안에서 항상 확인 가능하다.

---

## 9. AI (리포트·추천)

태스크별 처리 방식(확정 스택):

| 태스크 | 주기 | 처리 |
|---|---|---|
| 활동 데이터 구조화 (attempt 한 줄 요약 등) | 결과 제출 시마다 | **Gemini 3.5 Flash** (Google AI Studio — 2.5는 신규 발급 키에서 생성 불가 실측) |
| 변화 패턴 분석 (케어 상태·패턴 통계·이상 신호) | 일 배치 | **통계 로직(코드)** — AI 미사용 |
| 맞춤 활동 추천 | 하루 1회 배치 | **Qwen 3 32B** (OpenAI 호환 API — 운영 OpenRouter, 설정으로 Groq 전환 가능) |
| 기피 게임 보너스 책정 | 하루 1회 (추천 배치에 포함) | **통계 로직**(기피 지표 산출) + **Qwen 3 32B**(보너스 대상·금액 결정, §1.8 상한 내) |
| 가족 주간 리포트 | 주 1회 배치 + 수동 생성 | **Qwen 3 32B** (OpenAI 호환 API) — 비진단 톤 프롬프트 강제 |
| **음성 일기 발화 특징 산출** (반복 문장 비율·어휘 다양성·필러 빈도·발화 속도 추이) | 일 배치 | **통계 로직(코드)** — 전사 텍스트에서 수치만 산출, 외부 전송 없음 |
| **음성 일기 정서·주제 구조화** (하루 이야기 요약·기분 신호·개연성 신호) | 일 배치 | **Gemini 3.5 Flash** — **PII 스크러빙 후** 가명 ID로만 전송(§1.9 특칙 ④), 원문·해시 외 저장 없음. **활성화 전제: 입력 데이터를 학습에 쓰지 않는 유료 티어/약관 확인**(부록 A-7 ⑥) — 그 전까지 이 태스크는 비활성, 발화 특징 산출(통계 로직)만 운영 |

- **개인정보 분리 전송**(§1.9): 외부 LLM에는 식별 정보 없이 활동 데이터만 전달한다. 리포트 문단은 `{name}` 플레이스홀더로 생성한 뒤 서버가 실명으로 치환해 응답한다.
- **음성 일기 산출물의 노출 경계**: 전사 원문은 어떤 guardian API에도 나가지 않는다. 주간 리포트·패턴 요약에는 "대화 참여 일수, 정서 신호, 발화 특징 **변화** 요약"만 비진단 문구로 포함한다(예: "이야기 중 같은 내용을 반복하는 날이 늘었어요 — 대화 나눠보시면 좋아요". 진단 표현 금지: "인지 저하", "치매 의심" 류).
- **사람 검수**: 배치 생성 리포트는 게시 전 사람이 검수한다(내부 운영 절차) — 목록·상세 API는 검수 완료본만 노출. 수동 생성(`:generate`)은 데모용으로 즉시 노출된다.

### GET `/families/{family_id}/reports?senior_id={sid}&limit=10` — 주간 리포트 목록
```json
{"reports": [
  {"report_id": "…", "senior_id": "…", "period_start": "2026-07-15", "period_end": "2026-07-21", "generated_at": "…", "locked": false}
]}
```
- **프리미엄 게이트(§11)**: 무료 티어는 **최신 1건만 열람 가능** — 목록에는 전체가 내려오되 최신 외 항목은 `locked: true`(아카이브 잠금 UI 렌더링용). `premium` 이상은 전부 `locked: false`.

### GET `/families/{family_id}/reports/{report_id}` — 리포트 본문 (F-3)
```json
{
  "report_id": "…", "senior_id": "…",
  "period_start": "2026-07-15", "period_end": "2026-07-21",
  "generated_by": "qwen3-32b",
  "paragraphs": [
    "이번 주 김순자님은 5일 동안 꾸준히 미션에 참여하셨어요. …",
    "인지 게임 중 \"순서 기억하기\"에서 특히 좋은 반응 속도를 보이셨고 …",
    "다만 오후 시간대 참여가 오전보다 적은 편이에요. …"
  ],
  "care_points": [{"tone": "warn", "text": "오후 활동량이 오전보다 낮은 편이에요"}],
  "recommendations": [
    {"emoji": "🚶", "title": "오후 산책 미션", "desc": "하루 5,000보 목표를 오후에 나눠 걸어보세요"},
    {"emoji": "🧩", "title": "단어 분류하기", "desc": "언어 인지에 도움이 되는 게임을 추천해요"}
  ],
  "generated_at": "…"
}
```
- v1의 `summary_md` 단일 마크다운 대신 **`paragraphs[]` 평문 문단 배열**로 변경 — iOS 렌더링 구조(문단 카드)와 일치, 마크다운 렌더러 불필요.
- 리포트 톤: 진단·위험 판정 금지, 참여·변화 관찰 중심(기능명세서 안전 기준). 프롬프트로 강제 + 사람 검수(§9 상단).
- `generated_by`는 실제 생성 모델 식별자. **iOS 리포트 화면의 "Claude Sonnet 생성" 하드코딩은 제거**하고 "AI 분석 리포트" 등 범용 문구로 표시(모델명 노출 여부는 UI 결정).
- **프리미엄 게이트(§11)**: 무료 티어는 최신 리포트만 조회 가능 — 그 외(`locked: true` 항목) 조회 시 402.

### POST `/families/{family_id}/reports:generate` — 수동 생성 (guardian만)
- 응답은 리포트 본문과 동일. **가족당 3회/10분 레이트리밋(429)**. 최대 수십 초 소요 — 로딩 UI 필요.
- **프리미엄 게이트(§11)**: `premium` 이상 필요 — 미달 402.

### GET `/seniors/{senior_id}/recommendations?date=` — 오늘의 맞춤 추천
```json
{
  "senior_id": "…", "for_date": "2026-07-21", "source": "batch",
  "recommendations": [
    {"mission_id": "…", "code": "word_sort", "title": "단어·그림 분류하기", "emoji": "🧩",
     "mission_type": "game", "cognitive_domain": "language", "cognitive_label": "언어력",
     "reward_points": 5400, "bonus_points": 1800, "score": 0.87,
     "reason": "요즘 잘 안 하시는 활동이에요 — 오늘 완료하면 보너스 1,800P를 더 드려요"}
  ]
}
```
- 하루 1회 배치(Qwen 3 32B, OpenAI 호환 API)가 사전 계산한 결과 조회(항상 빠름). 홈 `featured_mission`과 리포트 `recommendations`의 원천. **기피 게임 보너스(`bonus_points`) 책정도 이 배치에서 함께 수행**되어 당일 assignment에 반영된다(§1.8).

---

## 10. 알림 (인앱 알림함·설정)

### GET `/me/notifications?cursor=&limit=20` — 알림함 (S-18)
```json
{"notifications": [
  {"notification_id": "…", "type": "mission", "emoji": "🎯", "title": "오늘의 미션이 준비됐어요", "occurred_at": "…", "read": false},
  {"notification_id": "…", "type": "reward", "emoji": "🪙", "title": "어제 미션 보상 7,200P가 저금통에 담겼어요", "occurred_at": "…", "read": true},
  {"notification_id": "…", "type": "family", "emoji": "💌", "title": "유정민 자녀가 응원을 보냈어요", "occurred_at": "…", "read": true}
], "next_cursor": null}
```

### POST `/me/notifications:mark-read` — 읽음 처리
```json
{"notification_ids": ["…"], "all": false}
```
응답 200: `{"updated": 3}`. `all: true`면 전체 읽음.

### GET · PUT `/me/notification-settings` — 알림 설정 (F-11, 시니어·guardian 공용)
```json
{"mission": true, "reward": true, "weekly_report": true, "care_alert": true, "marketing": false}
```
- 현재 iOS는 로컬 `@State` — 서버 저장으로 전환. **이 설정이 푸시 발송 필터의 원천이다**(아래).

### POST `/me/devices` — 푸시 디바이스 토큰 등록 (upsert) → 200
```json
// 요청
{"device_token": "<APNs device token>", "platform": "ios"}
// 응답
{"device_id": "…", "registered": true}
```
- 로그인 후 앱이 자동 호출(토큰 갱신 시 재호출 — 같은 기기는 upsert). 사용자당 다기기 허용.
- **로그아웃·탈퇴 시 클라이언트가 `DELETE /me/devices/{device_id}` 호출**(탈퇴는 서버도 일괄 삭제). 무효 토큰(APNs 피드백)은 서버가 자동 정리.

### 푸시 발송 규칙 (서버)

- 알림함 알림(§10) 생성 시, 수신자의 `notification-settings`에서 **해당 타입 토글이 켜진 경우에만** APNs 푸시를 발송한다(알림함 기록은 토글과 무관하게 항상 남음). `marketing` 푸시는 마케팅 동의(§1.7)까지 이중 확인.
- **케어 푸시(핵심 유스케이스)**: 무활동 3일 `alert`(§8 생성 룰) 등 `care` 타입 알림은 같은 가족의 **모든 guardian**(care_alert 토글 on)에게 발송된다 — 자녀가 앱을 열지 않아도 도달하는 것이 이 기능의 목적. 티어 무관(§8).
- **페이로드 최소화(§1.9와 일관)**: 푸시 본문에는 관계 호칭 수준만 담는다 — 예: "어머님의 활동이 3일째 없어요". 실명·활동 상세·점수는 포함 금지(잠금화면 노출 표면 최소화), 상세는 앱 진입 후 조회.
- 발송 채널(확정): **FCM 경유** — 기존 Firebase 프로젝트에 APNs 인증 키(.p8)·Team ID·번들 ID를 등록하고 서버는 FCM API만 호출한다(설정 일원화). 키 등록 전까지 발송은 no-op(알림함 기록·대상 산출은 동작).

---

## 11. 구독 (프리미엄 — 자녀 대상 결제 게이트)

수익 모델의 1차 검증 대상: **payer(자녀)·user(부모)가 분리된 구조에서 자녀에게 심화 인사이트를 반복과금**한다. 기존 대시보드·리포트 화면 위에 결제 게이트를 씌우는 구조라 신규 화면은 페이월(F-13) 하나다.

### 티어·게이트 정책 (확정)

| 티어 | 열람 범위 |
|---|---|
| `free` | F-1 대시보드 1콜 전체, F-2 7일 추이, 최신 주간 리포트 1건, 케어알림 최신 5건 |
| `premium` | + F-2 30일 추이, F-3 리포트 아카이브·수동 생성, F-9 참여 패턴, F-12 부모님 활동 달력, F-5 케어알림 전체 히스토리, `premium` 교육 콘텐츠(§5 — 가족 시니어에게 상속), **음성 응원 편지 영구 보관**(§3 — 구독 중 발송한 본인 발신분, 발송 시점에 확정되어 구독 해지 후에도 유지. 무료 발신분은 24시간 휘발) |
| `premium_ai` | **슬롯만 예약** — 추천 고도화·심화 리포트 등 상위 기능은 활동 데이터 축적 후 확정(부록 A-7 ④). 게이트 기준은 `premium`과 동일하게 시작 |

- **게이트는 서버가 강제**한다(402, §1.3) — 클라이언트 잠금 UI는 보조 수단일 뿐 서버 게이트 없이 배포 금지("서버가 확정" 원칙과 동일 선상).
- 게이트는 **guardian 호출에만 적용**. 시니어 본인의 자기 데이터 조회는 항상 허용(§1.2). 시니어의 `premium` 교육 콘텐츠 접근은 가족 guardian 구독 상속(§2 `entitlements`).
- 구독은 guardian 계정 귀속. 한 가족에 구독 guardian이 1명이라도 있으면 시니어 상속이 성립한다. guardian 간 상속은 없음(형제자매 각자 구독).

### GET `/me/subscription` — 현재 구독 상태
```json
{"tier": "premium", "status": "active", "product_id": "sub_premium_monthly",
 "started_at": "…", "expires_at": "…", "auto_renew": true}
```
- 무구독이면 `{"tier": "free", "status": null, …}`. `/me`의 `entitlements.tier`는 이 상태에서 파생된 **유효 티어**(시니어 상속 반영).
- **무료 체험(trial) — 도입 확정**: 체험 중에는 `status: "trial"`로 premium과 동일한 게이트 혜택을 받는다. 기간·시작 트리거(가입 시 자동 vs 페이월에서 시작)·**중복 방지 기준**(phone_hmac 기반 계정당/가족당 1회 등)은 A-7 ④ 결정 대기 — 확정 전까지 페이월에 체험 UI를 노출하지 않는다.

### POST `/me/subscription` — 구독 등록·갱신 (StoreKit2 영수증 검증, 멱등)
```json
// 요청
{"idempotency_key": "…", "product_id": "sub_premium_monthly", "transaction_id": "<StoreKit2 트랜잭션 ID>"}
// 응답 200
{"tier": "premium", "status": "active", "expires_at": "…", "replayed": false}
```
- 상품: `sub_premium_monthly`(→ `premium`), `sub_premium_ai_monthly`(→ `premium_ai`, 출시 시점은 부록 A-7 ④). 가격·연간 상품 구성은 결정 대기(부록 A-7 ④).
- 서버가 App Store 트랜잭션을 검증 후 티어 부여. 갱신·만료·환불 반영은 App Store Server Notification 연동(실결제 도입 시 — 부록 A-7 ②).
- **MVP는 데모 상태 관리만**: 결제 없이 `status: "demo"`로 티어를 부여해 게이트 플로우를 검증한다(충전 `status: "demo"`와 동일 원칙). 카드 정보는 Apple 보관 — 서버는 트랜잭션 ID만 저장(§1.9).
- guardian만 호출 가능(시니어 403).

---

## 12. 운영 (앱에서 직접 사용 안 함)

`GET /healthz`, `GET /readyz`, `GET /metrics`, `POST /api/v1/internal/jobs/*`(내부 토큰 필요 — 앱에서 호출 금지)

---

## 부록 A. iOS 팀과 사전 합의 필요사항 체크리스트

### A-1. 인증·계정 (블로킹 — 가장 먼저)
1. **Firebase 프로젝트 공유**: 동일 프로젝트에 iOS 앱 등록(`GoogleService-Info.plist`), 백엔드에 프로젝트 ID 설정. 로그인 수단은 **Apple + 카카오 확정** — 휴대폰 OTP 미제공, `PhoneLoginFlow`·"휴대폰 번호로 계속하기" 버튼은 카카오 버튼으로 대체.
   - **카카오 사전 준비**: Firebase Authentication을 **Identity Platform으로 업그레이드**(OIDC 제공자 기능 필요) → 제공자 `oidc.kakao` 등록. Kakao Developers에 앱 등록 + **OpenID Connect 활성화** + iOS 번들 ID·URL 스킴 설정. iOS에 KakaoSDK(Auth) 의존성 추가.
2. **세션 플로우**: Firebase 로그인 성공 → `POST /auth/session` 순서 고정. 온보딩의 **약관 동의 5종 + 역할 선택 + 가입 정보 입력(실명·이메일·전화번호·성별·생년월일) 화면 신설** 후 session 요청에 포함(현재 전부 미수집/로컬 → 서버 저장 전환). Apple·카카오 프로필 값으로 프리필 가능하되, Apple "Hide My Email" 사용 시 릴레이 주소가 수집될 수 있음(실이메일 요구 여부 정책 결정). `RootView`의 온보딩 완료 판정도 서버 `/me` 기준으로 변경.
3. **401 처리 규약**: 토큰 만료 시 갱신 → 1회 재시도, 실패 시 재로그인. `Unknown user` 401은 `/auth/session` 재호출로 복구.
4. **`bypassAuth` 제거**: DEBUG 기본 우회 플래그는 서버 연동 시작 시점에 debug 토큰 방식(§1.1)으로 대체.

### A-2. 데이터 계약 (스키마 고정)
5. **게임별 `raw_payload` 스키마**: 시드 6종(§1.8)의 필드를 미션게임 기획서 §5 저장 데이터 필드 기준으로 확정(§4 표). `version: 2`로 하위 호환 관리.
6. **성공 판정은 서버 정본**: 현재 클라 하드코딩된 기준(순서 2/3, 짝맞추기 실수≤4, 소리퀴즈 3/4)을 서버로 이관 — 화면 표시는 항상 `complete` 응답 기준.
7. **enum 상수 공유**: §1.7·§1.8 표를 Swift enum으로 코드젠 or 수동 동기화. `/openapi.json` 기반 클라이언트 생성(swift-openapi-generator) 여부 결정.
8. **Mock 모델 → API 모델 치환**: `Mission.points`(String "40P") → `reward_points`(Int), `AllowanceEntry.date`(String "오늘") → `created_at`(timestamp), `ShopItem.price`(String) → `price_points`(Int) 등 표시용 String 필드를 전부 수치·타임스탬프로 교체.

### A-3. 오프라인·멱등성 (offline-first 큐)
9. **idempotency_key 생성 주체 = 클라이언트**: 미션 완료·구매·출금·충전 페이로드 생성 시 UUID 발급, 큐에 함께 저장. 2xx 수신 시에만 삭제.
10. **재전송 규칙**: 네트워크 복구 시 순서대로 재전송. `replayed: true`는 정상 처리. **409 수신 시 큐에서 제거.**
11. **미전송 상태 UX**: 포인트 적립·차감은 서버 트랜잭션에서만 발생 — 큐 대기 중에는 "적립 예정" 표시, 로컬 잔액 가감 금지.

### A-4. 시간·표시 규칙
12. **"오늘" 판정은 서버**: 홈 `date` 그대로 사용. 자정 넘김 시 홈 재조회 타이밍 합의.
13. **포인트·잔액·진행률은 서버 값만 표시**: `balance`, `krw_value`, `today.missions_completed` 등 계산 금지. 현재 하드코딩된 값들(84,500P·3/5·4,820보·340/500P 등) 전부 API 값으로 대체.

### A-5. 캐싱·보안
14. **guardian 데이터 디스크 저장 금지**: 대시보드·지표·리포트·케어알림·활동 달력 응답은 메모리 캐시만. 시니어 홈은 SwiftData 캐시 허용(30s~5min TTL). 활동 달력은 시니어 앱에서 조회할 때만 디스크 캐시 허용.
15. **초대 코드 취급**: 평문 코드는 발급 응답에만 존재 — 표시·공유 후 영속 저장 금지.
16. **서버 캐시 TTL 인지**: 홈 30초·대시보드 60초(쓰기 후 무효화). pull-to-refresh 정책 합의.
17. **걸음 기능 제거(확정)**: HealthKit 연동 없음. iOS의 `StepsViews`(권한·상세 화면)·홈 걸음 카드·미션 목록 걷기 항목·주간 목표 "걸음수 5일" 조건은 모두 제거 대상.

### A-6. 환경·운영
18. **환경별 Base URL**: 로컬/스테이징/프로덕션 URL 및 빌드 설정 전환. HTTPS 필수.
19. **에러 UX 매핑**: §1.3 표(특히 409·429·5xx 백오프)를 공통 네트워크 레이어에 구현.
20. **스펙 변경 통지 절차**: 계약 변경 시 `docs/openapi.json` 커밋 + 채널 공지. Breaking change는 `/v1` 내 하위호환 유지.

### A-7. 미결정 사항 (기획·정책 결정 대기)

> **확정 완료**(본문 반영): ⓐ 1P=1원 고정(§6·§7) ⓑ 일일 미션 5개·홈 분모 5(§1.8) ⓒ 초대 코드 4자리+검증 레이트리밋(§3) ⓓ 휴대폰 OTP 미제공·로그인 Apple+카카오(§1.1) ⓔ 게임 시드=기획서 최종 6종·같은그림/소리퀴즈/길찾기 폐기(§1.8) ⓕ 걸음 기능 전면 제거 ⓖ 반응시간·힌트 계측 필수(§4) ⓗ SECURITY.md 통제 반영(§1.9) ⓘ **음성 응원 편지 신설**(§3 — 우체통·24h 휘발/프리미엄 영구, 익명 표시 옵션, 분석 투입 금지, 3회/일) ⓙ **프리미엄 구독 3티어(free/premium/premium_ai)·서버 게이트 402·시니어 상속**(§11) ⓚ **똑똑이 음성 일기**(§4·§1.9 특칙) ⓛ **가족 선물하기**(§6 — recipient 지정 구매, 시니어=지갑/guardian=충전 잔액).

21. **① 신규 게임 5종 서버 판정 확정**: iOS는 6종 전부 구현 완료(계측 포함) — 배정 임시 운영 방안은 불필요해짐. 남은 결정: 신규 5종(digit_symbol·spot_difference·signal_tap·word_sort·daily_order)의 서버 성공 판정 기준·`raw_payload` 스키마 확정(현재 iOS는 `problem_count`·`correct_count`·`error_count`·`mean_rt_ms`·`hints`·`rt_ms[]` 요약을 전송, sequence_recall은 `rounds[]`)과 보상 포인트 확정(기획서 §10).
22. **② 실결제·실이체 범위**: IAP 영수증 검증(충전 §7·구독 §11), 출금 실이체, 기프티콘 실발급·실물 배송(`fulfillment_type` §6) — MVP는 전부 데모 상태 관리만. 실결제 전환 시 App Store Server Notification(구독 갱신·환불) 연동 포함.
23. **③ 시니어 계좌 등록/변경 API**: 출금 화면의 계좌 정보 원천 결정. 단, 보안 원칙상 **금융결제 정보 미수집**(§1.9) — MVP는 계좌 수집 없이 데모 표시만 유지하고, 실출금 도입 시 별도 보안 검토 필수.
24. **④ 구독 상품 구성·가격**: 확정 — **무료 체험(트라이얼) 도입 자체는 확정**(발표자료 정본의 전환 깔때기 전제 — 체험 시작 → 유료 전환이 SOM 가설의 1단). 결정 대기: `sub_premium_monthly` 가격, 연간 상품 여부, **무료 체험 기간·중복 방지 기준**(계정당/가족당 1회 등), `premium_ai` 티어의 차별 기능(추천 고도화·심화 리포트)과 출시 시점 — 게이트 경계(§11 표)는 확정.
25. **⑤ OpenAI 용도**: 확정 스택에 OpenAI가 포함되나 §9 태스크 매핑에 없음 — 임베딩(pgvector 기반 추천 유사도)·폴백 등 용도 확정 필요.
26. **⑥ 무료 티어 약관 검토**: Google AI Studio·Groq 무료 티어는 입력 데이터가 모델 개선에 활용될 수 있음 — 가명화 전송 원칙(§1.9·§9) 준수 확인 + 정식 서비스 전 유료 티어 전환 검토.
27. **⑦ 똑똑이 음성 일기 세부 정책**: 방향 확정(§4 — 음성으로 하루 이야기 → 기기 내 STT 전사 → 내용·발화 특징 분석, §1.9 특칙). 결정 대기: ⒜ **STT 엔진** — iOS Speech(ko-KR) on-device 우선, on-device 미지원 기기의 처리(기능 비활성 vs Apple 서버 STT 허용 — 허용 시 개인정보처리방침 명시 필요) ⒝ **전사 원문 보존 기간 — 확정**: 유효 티어 free 30일·premium 이상 영구(구독 혜택) ⒞ 참여 보상 지급 여부·포인트 ⒟ 발화 특징 지표 목록·산식 확정(반복·어휘 다양성·조리성·단어 회상 신호) ⒠ 가족 리포트 노출 문구 가이드(비진단 표현 사전 + 사람 검수 범위) ⒡ 시니어 본인 동의 UX(가입 후 첫 진입 시 안내 — 온보딩 필수 동의에 포함 금지).
28. **⑧ 제휴 상품·광고·선물 이행 운영**: `partner` 카테고리(§6)의 제휴사 계약·정산·실발급 프로세스, 광고성 상품 표시 의무(광고 표기), **선물 실물 배송 도입 시 배송 주소 수집 보안 검토**(§1.9 신규 필드 원칙 — 현 단계 미수집·demo 이행) — 유저 베이스 성장 후 계약 단위로 확정.
29. **⑨ 음성 응원 편지 스토리지 — 확정: AWS S3**(암호화 at rest·24h 수명주기 규칙·서명 URL — L-1·L-2 통제 준수). 오디오 형식·상한은 현 계약 유지(AAC 60초·5MB·서명 URL 15분).
30. **⑩ 자금 흐름 남용 방어 수치·절차**(실결제 전환 ②③의 전제 — 원칙은 확정, 보안 문서.md §4.7): 충전 월 한도(가족당·충전자당), 출금 일/월 한도, 현금성 선물 월 한도, 신규 가족 연결 냉각 기간(후보 72시간~7일), FDS 룰 임계값, 본인 명의 계좌 검증 방식(1원 인증 vs 계좌실명조회 — 오픈뱅킹/펌뱅킹 사업자 선정), 선불전자지급수단 해당 여부·전금법 등록·AML 법률 검토 결과, 선택적 관계 인증(고한도 티어) 도입 여부.

---

## 부록 B. 화면 → API 매핑

| 화면 | API |
|---|---|
| O-4 로그인 (Apple·카카오) | Firebase Auth(Apple / `oidc.kakao`) → `POST /auth/session` |
| O-5b 약관 동의 | `POST /auth/session` (consents 포함) |
| O-3 역할 선택 | `POST /auth/session` (role 포함) |
| O-6 가족 연결(자녀) | `POST /families` → `POST /invites` → `GET /invites/{id}` 폴링 (형제자매 초대도 동일 플로우, F-6 재사용) |
| O-7 가족 연결(시니어) | `POST /families/join` |
| S-1 시니어 홈 | `GET /seniors/{id}/home` (우체통 = `mailbox`) |
| S-2 체크인 | `POST /seniors/{id}/checkins` |
| S-20 똑똑이 음성 일기 (**신규 화면**) | `POST·GET·DELETE /seniors/{id}/voice-diaries` (원문 기본 비공개 — 시니어가 공유 설정 시 guardian 읽기 허용) |
| S-21 우체통·편지함 (**신규 화면**) | `GET /seniors/{id}/cheer-letters`, `POST …/{cheer_id}:listened` |
| S-3 미션 목록 | `GET /missions/today`, `GET /missions?type=game`, `GET /edu-contents` |
| S-4~6 게임 인트로·플레이·결과 | `POST /attempts` → `POST /attempts/{id}/complete` |
| S-7 보상 획득 | `complete` 응답의 `earned_points`·`today` |
| S-8 용돈함 | `GET /wallet`, `GET /points/ledger` |
| S-9 걸음수 | **기능 제거 확정** — 화면·API 없음 |
| S-10 마이페이지 | `GET /me`, `GET /families/{id}/members` |
| S-12 출금 | `POST /withdrawals`, `GET /withdrawals` |
| S-13 상점 | `GET /shop/items`, `POST /shop/items/{id}/purchase`(선물 = `recipient_user_id`), `GET /families/{id}/gifts` |
| S-15 교육 상세 | `GET /edu-contents/{id}`, `POST …/complete` (`premium` 콘텐츠는 유효 티어 필요 — §5) |
| S-17 주간 목표 | `GET /seniors/{id}/weekly-goal` |
| S-18 알림함 | `GET /me/notifications`, `POST …:mark-read` |
| (화면 없음) 푸시 등록 | `POST /me/devices` (로그인 후 자동), `DELETE /me/devices/{id}` (로그아웃 시) |
| S-19 활동 달력 (**신규 화면**) | `GET /seniors/{id}/activity-calendar`, `GET …/{date}` |
| F-1 대시보드 | `GET /families/{id}/dashboard` + 음성 응원 녹음 시트 `POST /families/{id}/cheers`(multipart) |
| F-2 활동 추이 | `GET /seniors/{id}/metrics/daily` ×3 지표 (30일 = premium) |
| F-3 주간 리포트 | `GET /reports`, `GET /reports/{id}`, `POST /reports:generate` (아카이브·수동 생성 = premium) |
| F-4 용돈 관리 | `GET /reward-wallet`, `GET /reward-wallet/charges`(충전자 표시), `GET·PUT /reward-rules` |
| F-5 케어 알림 | `GET /families/{id}/care-alerts` (전체 히스토리 = premium) |
| F-6 부모님 프로필 | `GET /families/{id}/members` (추가 연결 = 형제자매 초대 겸용) |
| F-7 미션 히스토리 | `GET /seniors/{id}/missions/{mid}/history` |
| F-8 포인트 충전 | StoreKit + `POST /reward-wallet/charges`, `GET /reward-wallet/charges` |
| F-9 참여 패턴 | `GET /seniors/{id}/pattern-summary` (premium) |
| F-10 보상 조건 | `GET·PUT /reward-rules` |
| F-11 알림 설정 | `GET·PUT /me/notification-settings` |
| F-12 부모님 활동 달력 (**신규 화면**) | S-19와 동일 엔드포인트 (guardian 읽기·감사 로그, premium) |
| F-13 프리미엄 페이월 (**신규 화면**) | `GET·POST /me/subscription` + StoreKit2 구독 |
| G-1 설정 | `GET /me`, `PATCH /me`, 로그아웃(Firebase) |
| G-2 아바타 꾸미기 (**신규 화면**, S-10·G-1에서 진입) | `GET /me/avatar/catalog`, `PATCH /me`(avatar) — 유료 아이템은 상점 `deco` 구매 연동(§6) |
