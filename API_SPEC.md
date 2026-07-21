# 똑똑용돈 API 명세서 v2 (iOS 클라이언트 기준)

> 기준: `knocknockMoney-ios` 목업 코드(화면·데이터 구조)와 기능명세서·유저스토리.
> 백엔드는 미구현 상태이며 이 문서가 **구현 대상 계약 초안**이다. 구현 후에는 서버 `/openapi.json`(OpenAPI 3.1)이 기계 판독 정본이 된다.
> v1 대비 주요 변경: 게임 시드를 미션게임 기획서 최종 6종으로 교체, 성공/실패 판정 도입, 보상 구조를 "가족 지갑 충전 → 시니어 적립 → 출금/상점" 모델로 개편, 교육·알림·상점·출금·주간목표 API 신설, 대시보드를 iOS 화면 단위로 재설계. (근거: 부록 B 화면→API 매핑)
> **확정 정책**: 1P = 1원 고정 · 일일 미션 5개(홈 진행률 분모 5) · 초대 코드 4자리 · 로그인은 Apple + 카카오(휴대폰 OTP 미제공) · 활동 달력 제공(§4, 시니어·자녀 공용) · 게임 시드 = 기획서 최종 6종(§1.8) · **걸음 기능 제거** · 반응시간·힌트 계측 필수(§4) · 보상 포인트 = 인지 기여도(연구 연계 순위) 비례 + **AI 기피 게임 보너스**(§1.8·§9).
> **확정 스택**: iOS **Swift + SwiftUI** · Backend **Python + FastAPI + PostgreSQL(pgvector)** · AI **Gemini 2.5 Flash(구조화) + Qwen 3 32B@Groq(추천·리포트) + 통계 로직(패턴 분석)** — 태스크별 매핑은 §9 · CI/CD **GitHub Actions**.

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
| consent `code` | `over14`, `tos`, `privacy`, `sensitive_data`, `marketing` |
| calendar day `status` | `none`, `partial`, `complete` |

### 1.9 데이터·보안 원칙 (보안 문서 반영)

수집·저장 데이터: 계정 연동용 기본 식별 정보, 프로필, 미션 참여·활동 데이터, AI 생성 리포트·추천 결과. **실제 금융결제 정보와 의료기관의 진료·의료 데이터는 수집·저장하지 않는다.**

- **수집 항목(확정)**: 회원가입 시 **실명·이메일·전화번호·성별·생년월일**(+역할·타임존)을 수집한다. 그 외 항목(주소·주민번호 등)은 수집하지 않으며, 새 개인정보 필드 추가 시 보안 검토를 선행한다. 실명·이메일·전화번호·생년월일은 `identity` 스키마에 **암호화 저장**하고 검색은 HMAC 인덱스로만 한다.
- **개인정보·활동 데이터 분리 저장**: DB를 **5개 스키마로 분리**(`identity` / `activity` / `reward` / `ai` / `audit`)하고 교차 스키마 물리 FK 없이 **내부 식별자(UUID)로만 논리 연결**한다. `activity`·`reward`·`ai` 스키마에는 이름·연락처 컬럼 자체를 두지 않는다. 활동 리소스 응답(attempts, checkins, metrics …)에도 식별 정보를 포함하지 않는다.
- **토큰·코드 취급**: 초대 코드는 HMAC 해시만 저장(평문은 발급 응답 1회), 감사 로그의 IP는 HMAC 해시만 보관(원본 미저장), LLM 프롬프트는 원문을 저장하지 않고 입력 해시만 기록한다.
- **인증·권한**: Firebase Authentication JWT(§1.1) + 가족 연결 기반 열람 권한(§1.2). 대시보드·리포트는 연결된 가족에게만 열람 허용하며 guardian의 시니어 데이터 열람은 **감사 로그**를 남긴다.
- **암호화**: 전 구간 HTTPS(TLS) 필수. 전화번호 등 주요 개인정보는 저장 시 암호화하고, 탈퇴 시 암호화 키를 폐기한다(§2 `DELETE /me`).
- **결제 정보 미수집**: 포인트 충전은 App Store 인앱 결제로 처리 — 카드 정보는 Apple이 보관하고 서버는 StoreKit 트랜잭션 ID만 저장한다(§7). 출금 화면의 계좌 정보는 데모 표시이며 **실계좌 수집은 MVP 범위 외**(부록 A-7 ④).
- **AI 개인정보 보호**: 외부 LLM(Google AI Studio·Groq 등)에는 이름·연락처 등 직접 식별 정보를 전달하지 않고, 가명 처리된 활동 데이터만 전송한다(§9).
- MVP 이후 단계적 강화: 접근 로그 관리, 비식별화 처리, 데이터 암호화 고도화, 개인정보 접근 이력 관리.
- 세부 통제 항목·등급·보강 로드맵은 [SECURITY.md](./SECURITY.md)(**목표 보안 설계 체크리스트**)를 따른다.

### 1.8 미션 시드 (미션게임 기획서 최종 6종 + 체크인)

| code | 이름 | type | domain | 표시 라벨 | 기본 보상 | 예상 시간 | 기획서 순위 | iOS 상태 |
|---|---|---|---|---|---|---|---|---|
| `digit_symbol` | 숫자·기호 짝맞추기 | game | attention | **순발력** | **50P** | 180s | 1위 | ❌ 신규 구현 |
| `spot_difference` | 다른 그림 찾기 | game | visuospatial | **관찰력** | **45P** | 240s | 2위 | ❌ 신규 구현 |
| `sequence_recall` | 순서 기억하기 | game | memory | **기억력** | **40P** | 180s | 3위 | ✅ 구현 완료 |
| `signal_tap` | 신호에 맞춰 누르기 | game | attention | **집중력** | **35P** | 120s | 4위 | ❌ 신규 구현 |
| `word_sort` | 단어·그림 분류하기 | game | language | **언어력** | **30P** | 240s | 5위 | ⚠️ 목업(완성 필요) |
| `daily_order` | 일상 순서 맞추기 | game | executive | **판단력** | **25P** | 240s | 6위 | ⚠️ 목업(완성 필요) |
| `daily_checkin` | 기분·수면 체크인 | checkin | null | — | 10P | 60s | — | ✅ 구현 완료 |

- 시드는 **미션게임 기획서 최종 6종으로 확정**. iOS의 같은 그림 찾기·소리 퀴즈(구현 완료분)·길 찾기(목업)는 시드에서 제외 — 해당 화면·게임 코드는 제거 대상.
- **`cognitive_label`(표시 라벨, 확정)**: 미션 카드 옆에 붙는 짧은 인지 활동 라벨("기억력"·"순발력" 등). mission 객체가 포함되는 **모든 응답**(홈·미션 목록·카탈로그·추천·히스토리)에 서버가 내려준다. 라벨은 기획서 §5의 게임별 관찰 데이터(연관 인지 활동 영역 — 예: 숫자·기호=처리 속도·시각 주의 → 순발력, 신호 탭=반응 억제·지속 주의 → 집중력)에서 도출한 값이며, "이 활동이 어떤 인지 영역과 관련되는지"의 안내다 — 능력 평가·진단 표현 금지 원칙(§9) 유지.
- **일일 미션 배정은 5개 고정** — 게임 6종 풀에서 규칙/AI 추천이 5종을 선택한다. 홈 진행률 분모(`missions_total`)는 항상 5.
- ⚠️ 완성된 게임이 현재 `sequence_recall` 1종뿐이므로, 신규·목업 5종 구현 전까지의 배정 운영 방안(중복 배정 허용 or 임시 축소)은 결정 필요(부록 A-7 ①).
- `daily_checkin`은 `POST /checkins` 성공 시 서버가 자동 완료 처리한다(별도 attempt 불필요). **배정 5개·홈 진행률 분모에는 포함되지 않는** 별도 일일 활동이다.
- **보상 원칙(확정)**: 기본 보상은 **인지능력 향상 기여도(기획서 연구 연계 순위)에 비례**한다 — 1위 50P부터 6위 25P까지 5P 단위 하향. 순위 비례 원칙이 정본이며 개별 수치는 잠정(기획서 §10에서 확정).
- **AI 기피 게임 보너스(확정)**: 통계 로직이 게임별 기피 지표(최근 14일 배정 대비 미시작률·완료율·중도 이탈률)를 산출하고, 일일 추천 배치(Qwen, §9)가 기피 게임에 `bonus_points`를 책정해 당일 배정에 포함한다. **상한: 기본 보상의 50%, 동시 최대 2개 게임.** 보너스는 성공 완료 시에만 지급되며 원장에 `engagement_bonus`로 별도 기록된다. UI 표시: "40P + 보너스 20P".

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
  "birth_date": "1949-03-02",
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

- 약관 동의는 시각·버전과 함께 서버에 이력 저장한다(민감정보 처리 근거). `marketing`은 이후 `PATCH /me`로 변경 가능.

### GET `/me` — 내 프로필 + 소속 가족

응답 200: `user_id, role, status, real_name, display_name, email, phone, birth_date, gender, timezone, families[], consents[]`

### PATCH `/me` — 프로필·마케팅 동의 수정

요청(모두 선택): `real_name`, `display_name`, `email`, `phone`, `birth_date`, `gender`, `timezone`, `marketing_consent` → 응답은 GET `/me`와 동일.

### DELETE `/me` — 탈퇴 → 204

soft delete + 암호화 키 폐기. 이후 모든 호출은 401.

---

## 3. 가족 (초대·연결)

온보딩 플로우: 자녀가 가족 생성 → 초대 코드 발급·공유(카카오톡/복사/QR) → 시니어가 코드 입력으로 참여 → 자녀 화면은 초대 상태를 폴링해 "연결 완료" 전환.

### POST `/families` — 가족 생성 (guardian만, 시니어는 403) → 201
```json
// 요청
{"name": "김씨네", "relation_label": "큰딸"}
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
   "display_name": "김순자", "relation_label": "어머니", "joined_at": "2026-03-12T…"}
]}
```
- 한 guardian이 여러 시니어(어머니·아버지)와 연결 가능 — 대시보드 상단 부모 선택 칩은 이 목록의 `member_role == "senior"` 항목으로 구성.

### DELETE `/families/{family_id}/members/{user_id}` → 204
본인 탈퇴 또는 guardian에 의한 해제.

---

## 4. 시니어 참여 (홈·미션·체크인·걸음)

### GET `/seniors/{senior_id}/home` — **홈 1콜 (S-1)**

홈 화면 전체를 한 번에 내려준다. 서버 캐시 30초(쓰기 후 무효화).

```json
{
  "date": "2026-07-21",
  "display_name": "김순자",
  "featured_mission": {
    "assignment_id": "…", "status": "assigned", "source": "ai", "rank": 1, "bonus_points": 20,
    "mission": {"mission_id": "…", "code": "sequence_recall", "mission_type": "game",
                "title": "순서 기억하기", "emoji": "🧠", "difficulty": 2,
                "cognitive_domain": "memory", "cognitive_label": "기억력",
                "reward_points": 40, "estimated_seconds": 180}
  },
  "missions_total": 5,
  "missions_completed": 3,
  "checkin": {"done": false, "reward_points": 10},
  "extra_activities": [
    {"kind": "mission", "code": "digit_symbol", "emoji": "🔢", "title": "숫자·기호 짝맞추기", "meta": "인지 게임 · 약 3분", "reward_points": 30},
    {"kind": "edu", "content_id": "…", "emoji": "📖", "title": "오늘의 건강 상식", "meta": "교육 콘텐츠 · 약 2분", "reward_points": 15}
  ],
  "balance": 84500,
  "weekly_goal": {"week_start": "2026-07-20", "target_points": 500, "earned_points": 340},
  "streak_days": 2
}
```
- `featured_mission`: 홈 "오늘의 미션" 카드(추천 rank 1).

### GET `/seniors/{senior_id}/missions/today` — 오늘 미션 목록 (S-3 "오늘의 미션" 탭)
```json
{"date": "2026-07-21", "missions": [AssignedMission…]}
```
- 각 항목에 `emoji`, `estimated_seconds`, `reward_points`, `cognitive_label`(인지 활동 라벨 — 카드 옆 칩으로 표시), assignment `status`·`bonus_points`(기피 게임 보너스, 기본 0) 포함 — 카드의 "완료" 체크 표시는 `status == "completed"`, 보너스가 있으면 "+보너스 nP" 배지 표시.
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
  "earned_points": 60,
  "bonus_points": 20,
  "balance": 84560,
  "today": {"missions_completed": 4, "missions_total": 5},
  "weekly_goal": {"target_points": 500, "earned_points": 380},
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
- `reaction_time_ms_avg`·`hint_count`는 **필수 필드**(확정) — iOS 전 게임에 문항별 반응시간 기록과 힌트 동작을 구현한다(현재 3종 모두 미측정·힌트 버튼 무동작 → 계측 추가 대상). 서버는 `raw_payload`의 문항별 `mean_rt_ms`·`hints`로 재계산해 클라이언트 값을 덮어쓴다.
- 크기 제약: `raw_payload` **32KB·문항 50개 상한**(초과 400), 전역 요청 본문 **256KB**(초과 413). 터치 좌표·센서 원본 스트림 전송 금지 — 문항 단위 요약만.
- 응답의 `today`·`weekly_goal`로 S-7 보상 화면의 "오늘 목표 4/5" 진행 표시를 갱신한다(하드코딩 제거).
- 이미 성공 완료된 attempt에 **다른 키**로 재요청 시 409 → 큐에서 제거.

### POST `/seniors/{senior_id}/checkins` — 일일 체크인 (S-2, 하루 1회 upsert)

iOS 체크인은 기분(5단계)·수면 품질(3단계) 2스텝. 수면은 시간이 아닌 **품질 코드**로 수집한다.

```json
// 요청
{"mood": 4, "sleep_quality": 3, "note_text": null}
// 응답 200
{"checkin_id": "…", "checkin_date": "2026-07-21", "created": true, "earned_points": 10, "balance": 84550}
```
- `mood` 1~5 (😣힘들어요=1 … 😄아주 좋아요=5), `sleep_quality` 1~3 (잘 못 잤어요=1, 보통=2, 푹 잤어요=3), `note_text` ≤ 500자(선택 — 임의 개인정보 입력 가능성이 있어 **서버에서 암호화 저장**, LLM 입력 제외).
- 보상 **+10P는 최초 1회만**. 같은 날 재호출은 값 덮어쓰기(`created: false`, `earned_points: 0`).
- 성공 시 `daily_checkin` 미션 자동 완료.

### GET `/seniors/{senior_id}/activity-calendar?month=2026-07` — 활동 달력 (월간)

시니어(내 참여 기록)와 guardian(부모님 활동 한눈에 보기)이 같은 엔드포인트를 사용한다. **신규 화면** — 현재 iOS 목업에 없음.

```json
{
  "month": "2026-07",
  "days": [
    {"date": "2026-07-01", "status": "complete",
     "missions_completed": 5, "missions_total": 5,
     "checked_in": true, "earned_points": 190},
    {"date": "2026-07-02", "status": "partial",
     "missions_completed": 2, "missions_total": 5,
     "checked_in": true, "earned_points": 80},
    {"date": "2026-07-03", "status": "none",
     "missions_completed": 0, "missions_total": 5,
     "checked_in": false, "earned_points": 0}
  ],
  "summary": {"active_days": 18, "perfect_days": 6, "total_points": 1240, "longest_streak": 7}
}
```
- `status` 판정(달력 셀 스탬프 렌더링용): `complete` = 배정 5개 전부 완료 / `partial` = 미션 1개 이상 완료 또는 체크인 / `none` = 활동 없음.
- `days`는 해당 월 1일부터 **오늘까지만** 포함(미래 날짜 없음). 가입 이전 월 요청 시 `days: []`.
- `month`는 시니어 타임존(Asia/Seoul) 기준. 서버 캐시 5분(당일 셀은 쓰기 후 무효화).
- `summary.active_days` = `partial` 이상인 날 수, `perfect_days` = `complete`인 날 수, `longest_streak` = 해당 월 내 최장 연속 활동일.

### GET `/seniors/{senior_id}/activity-calendar/{date}` — 일별 활동 상세 (달력 셀 탭)

```json
{
  "date": "2026-07-21",
  "missions": [
    {"code": "sequence_recall", "emoji": "🧠", "title": "순서 기억하기",
     "status": "completed", "score": 92, "earned_points": 40, "completed_at": "…"}
  ],
  "checkin": {"done": true, "mood": 4, "sleep_quality": 3},
  "earned_points_total": 90
}
```
- 그날 배정된 미션 5개 전체가 `status`와 함께 내려온다(미완료는 `assigned`·`score: null`).

---

## 5. 교육 콘텐츠 (S-3 교육 탭 · S-15 상세)

### GET `/edu-contents` — 목록
```json
{"contents": [
  {"content_id": "…", "emoji": "🩺", "title": "혈압, 이렇게 관리해요", "format": "article",
   "estimated_seconds": 120, "reward_points": 15, "completed": false}
]}
```
- `format`: `article` | `video`. `completed`는 요청자(시니어) 기준.

### GET `/edu-contents/{content_id}` — 상세
```json
{"content_id": "…", "title": "…", "format": "article",
 "body_md": "## 혈압 관리\n\n…", "video_url": null, "reward_points": 15, "completed": false}
```
- 본문은 서버 제공(`body_md` 마크다운 또는 `video_url`). 현재 iOS의 하드코딩 본문 제거 대상.

### POST `/edu-contents/{content_id}/complete` — 완료 제출 (멱등)
```json
// 요청
{"idempotency_key": "…"}
// 응답 200
{"content_id": "…", "earned_points": 15, "balance": 84565, "replayed": false}
```
- 콘텐츠당 보상 1회. 재완료 시 `earned_points: 0`.

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
  "weekly_goal": {"week_start": "2026-07-20", "target_points": 500, "earned_points": 340}
}
```

### GET `/seniors/{senior_id}/points/ledger?cursor=&limit=20` — 적립·사용 내역
```json
{
  "entries": [
    {"id": "…", "delta": 40, "balance_after": 84500, "reason_type": "mission_complete",
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
  "target_points": 500,
  "earned_points": 340,
  "conditions": [
    {"type": "streak", "label": "3일 연속 참여", "achieved": true, "bonus_points": 100,
     "progress": {"current": 3, "target": 3}},
    {"type": "weekly_points", "label": "주간 목표 500P", "achieved": false,
     "progress": {"current": 340, "target": 500}}
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
- `GET /seniors/{senior_id}/withdrawals` — 신청 내역 목록.

### GET `/shop/items` — 상점 목록 (S-13)
```json
{"items": [
  {"item_id": "…", "emoji": "☕", "name": "카페 기프티콘", "description": "5,000원권",
   "price_points": 5000, "category": "gifticon", "purchased": false}
]}
```
- `price_points: 0` = 무료. `category`: `deco` | `theme` | `gifticon` | `widget` 등.

### POST `/shop/items/{item_id}/purchase` — 구매 (멱등)
```json
// 요청
{"idempotency_key": "…"}
// 응답 200
{"purchase_id": "…", "item_id": "…", "price_points": 5000, "balance": 29500, "replayed": false}
```
- 잔액 부족 400, 중복 구매 불가 아이템 재구매 409. 기프티콘 실발급은 MVP 범위 외(데모 처리, 부록 A-7).

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

### GET · PUT `/families/{family_id}/reward-rules?senior_id={sid}` — 보상 규칙 (F-4·F-10, guardian만 PUT)
```json
{
  "senior_id": "…",
  "weekly_goal_points": 500,
  "goal_bonus": {"enabled": true, "bonus_points": 100},
  "streak_bonus": {"enabled": true, "days": 3, "bonus_points": 100},
  "weekly_cap": {"enabled": true, "cap_points": 1000},
  "monthly_target_krw": 20000
}
```
- v1의 "용돈 플랜(monthly_amount·환산율·cycle)"을 대체한다. 시니어 S-17 주간 목표 화면과 같은 데이터 원천.
- 보너스 포인트는 충전 잔액에서 자동 지급. PUT 시 유효 범위: `weekly_goal_points` 100~2,000(100 단위).
- **환산율은 1P=1원으로 확정** — `point_to_krw_rate`는 항상 `1.0`(응답 호환용 상수). 자녀 보상 관리 화면(F-4)의 "포인트 단가 100P당 N원" 조절 UI는 제거 대상.

---

## 8. 가족 대시보드 (guardian 화면)

> ⚠️ guardian 앱은 이 섹션 응답을 **디스크에 영속화하지 않는다**(메모리 캐시만).

### GET `/families/{family_id}/dashboard?senior_id={sid}` — **대시보드 1콜 (F-1)**

시니어 홈과 동일하게 guardian 홈도 1콜로 구성한다. 서버 캐시 60초.

```json
{
  "senior": {"senior_id": "…", "display_name": "김순자", "relation_label": "어머니"},
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
- `comment`는 결과 제출 시 활동 데이터 구조화 단계(**Gemini 2.5 Flash**, §9 표)가 생성해 저장한 한 줄 요약 — 식별 정보 없이 활동 데이터만 전송(§1.9). v1에서 누락됐던 attempt 단위 이력 조회를 이 엔드포인트가 담당.

### GET `/families/{family_id}/care-alerts?senior_id={sid}&cursor=` — 케어 알림 (F-5)
```json
{"alerts": [
  {"alert_id": "…", "severity": "alert", "emoji": "⚠️", "title": "3일 연속 게임 정답률이 평소보다 낮아요", "occurred_at": "…"},
  {"alert_id": "…", "severity": "caution", "emoji": "📉", "title": "오후 활동량이 지난주보다 줄었어요", "occurred_at": "…"},
  {"alert_id": "…", "severity": "info", "emoji": "✅", "title": "오늘 미션 3개를 완료했어요", "occurred_at": "…"}
], "next_cursor": null}
```
- `severity` → 태그 매핑: `alert`=확인 필요 / `caution`=주의 / `info`=정상.

---

## 9. AI (리포트·추천)

태스크별 처리 방식(확정 스택):

| 태스크 | 주기 | 처리 |
|---|---|---|
| 활동 데이터 구조화 (attempt 한 줄 요약 등) | 결과 제출 시마다 | **Gemini 2.5 Flash** (Google AI Studio) |
| 변화 패턴 분석 (케어 상태·패턴 통계·이상 신호) | 일 배치 | **통계 로직(코드)** — AI 미사용 |
| 맞춤 활동 추천 | 하루 1회 배치 | **Qwen 3 32B** (Groq, OpenAI 호환 API) |
| 기피 게임 보너스 책정 | 하루 1회 (추천 배치에 포함) | **통계 로직**(기피 지표 산출) + **Qwen 3 32B**(보너스 대상·금액 결정, §1.8 상한 내) |
| 가족 주간 리포트 | 주 1회 배치 + 수동 생성 | **Qwen 3 32B** (Groq) — 비진단 톤 프롬프트 강제 |

- **개인정보 분리 전송**(§1.9): 외부 LLM에는 식별 정보 없이 활동 데이터만 전달한다. 리포트 문단은 `{name}` 플레이스홀더로 생성한 뒤 서버가 실명으로 치환해 응답한다.
- **사람 검수**: 배치 생성 리포트는 게시 전 사람이 검수한다(내부 운영 절차) — 목록·상세 API는 검수 완료본만 노출. 수동 생성(`:generate`)은 데모용으로 즉시 노출된다.

### GET `/families/{family_id}/reports?senior_id={sid}&limit=10` — 주간 리포트 목록
```json
{"reports": [
  {"report_id": "…", "senior_id": "…", "period_start": "2026-07-15", "period_end": "2026-07-21", "generated_at": "…"}
]}
```

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

### POST `/families/{family_id}/reports:generate` — 수동 생성 (guardian만)
- 응답은 리포트 본문과 동일. **가족당 3회/10분 레이트리밋(429)**. 최대 수십 초 소요 — 로딩 UI 필요.

### GET `/seniors/{senior_id}/recommendations?date=` — 오늘의 맞춤 추천
```json
{
  "senior_id": "…", "for_date": "2026-07-21", "source": "batch",
  "recommendations": [
    {"mission_id": "…", "code": "word_sort", "title": "단어·그림 분류하기", "emoji": "🧩",
     "mission_type": "game", "cognitive_domain": "language", "cognitive_label": "언어력",
     "reward_points": 30, "bonus_points": 15, "score": 0.87,
     "reason": "요즘 잘 안 하시는 활동이에요 — 오늘 완료하면 보너스 15P를 더 드려요"}
  ]
}
```
- 하루 1회 배치(Qwen 3 32B@Groq)가 사전 계산한 결과 조회(항상 빠름). 홈 `featured_mission`과 리포트 `recommendations`의 원천. **기피 게임 보너스(`bonus_points`) 책정도 이 배치에서 함께 수행**되어 당일 assignment에 반영된다(§1.8).

---

## 10. 알림 (인앱 알림함·설정)

### GET `/me/notifications?cursor=&limit=20` — 알림함 (S-18)
```json
{"notifications": [
  {"notification_id": "…", "type": "mission", "emoji": "🎯", "title": "오늘의 미션이 준비됐어요", "occurred_at": "…", "read": false},
  {"notification_id": "…", "type": "reward", "emoji": "🪙", "title": "어제 미션 보상 90P가 저금통에 담겼어요", "occurred_at": "…", "read": true},
  {"notification_id": "…", "type": "family", "emoji": "💌", "title": "김민준 자녀가 응원을 보냈어요", "occurred_at": "…", "read": true}
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
- 현재 iOS는 로컬 `@State` — 서버 저장으로 전환(푸시 발송 필터의 원천이 되어야 함).

---

## 11. 운영 (앱에서 직접 사용 안 함)

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

> **확정 완료**(본문 반영): ⓐ 1P=1원 고정(§6·§7) ⓑ 일일 미션 5개·홈 분모 5(§1.8) ⓒ 초대 코드 4자리+검증 레이트리밋(§3) ⓓ 휴대폰 OTP 미제공·로그인 Apple+카카오(§1.1) ⓔ 게임 시드=기획서 최종 6종·같은그림/소리퀴즈/길찾기 폐기(§1.8) ⓕ 걸음 기능 전면 제거 ⓖ 반응시간·힌트 계측 필수(§4) ⓗ SECURITY.md 통제 반영(§1.9).

21. **① 미구현 게임 5종 운영**: 완성 게임이 `sequence_recall` 1종뿐 — 신규 3종(digit_symbol·spot_difference·signal_tap)과 목업 2종(word_sort·daily_order) 구현 전까지 일일 5개 배정 방안(같은 게임 중복 배정 허용 vs 배정 수 임시 축소) 결정. 신규 게임의 성공 판정 기준·보상 포인트도 구현 시 확정(기획서 §10).
22. **② 실결제·실이체 범위**: IAP 영수증 검증, 출금 실이체, 기프티콘 실발급 — MVP는 전부 데모 상태 관리만.
23. **③ 시니어 계좌 등록/변경 API**: 출금 화면의 계좌 정보 원천 결정. 단, 보안 원칙상 **금융결제 정보 미수집**(§1.9) — MVP는 계좌 수집 없이 데모 표시만 유지하고, 실출금 도입 시 별도 보안 검토 필수.
24. **④ 자녀 응원 메시지**: 알림함에 "자녀가 응원을 보냈어요" 항목 존재 — 응원 보내기 API 신설 여부.
25. **⑤ OpenAI 용도**: 확정 스택에 OpenAI가 포함되나 §9 태스크 매핑에 없음 — 임베딩(pgvector 기반 추천 유사도)·폴백 등 용도 확정 필요.
26. **⑥ 무료 티어 약관 검토**: Google AI Studio·Groq 무료 티어는 입력 데이터가 모델 개선에 활용될 수 있음 — 가명화 전송 원칙(§1.9·§9) 준수 확인 + 정식 서비스 전 유료 티어 전환 검토.

---

## 부록 B. 화면 → API 매핑

| 화면 | API |
|---|---|
| O-4 로그인 (Apple·카카오) | Firebase Auth(Apple / `oidc.kakao`) → `POST /auth/session` |
| O-5b 약관 동의 | `POST /auth/session` (consents 포함) |
| O-3 역할 선택 | `POST /auth/session` (role 포함) |
| O-6 가족 연결(자녀) | `POST /families` → `POST /invites` → `GET /invites/{id}` 폴링 |
| O-7 가족 연결(시니어) | `POST /families/join` |
| S-1 시니어 홈 | `GET /seniors/{id}/home` |
| S-2 체크인 | `POST /seniors/{id}/checkins` |
| S-3 미션 목록 | `GET /missions/today`, `GET /missions?type=game`, `GET /edu-contents` |
| S-4~6 게임 인트로·플레이·결과 | `POST /attempts` → `POST /attempts/{id}/complete` |
| S-7 보상 획득 | `complete` 응답의 `earned_points`·`today` |
| S-8 용돈함 | `GET /wallet`, `GET /points/ledger` |
| S-9 걸음수 | **기능 제거 확정** — 화면·API 없음 |
| S-10 마이페이지 | `GET /me`, `GET /families/{id}/members` |
| S-12 출금 | `POST /withdrawals`, `GET /withdrawals` |
| S-13 상점 | `GET /shop/items`, `POST /shop/items/{id}/purchase` |
| S-15 교육 상세 | `GET /edu-contents/{id}`, `POST …/complete` |
| S-17 주간 목표 | `GET /seniors/{id}/weekly-goal` |
| S-18 알림함 | `GET /me/notifications`, `POST …:mark-read` |
| S-19 활동 달력 (**신규 화면**) | `GET /seniors/{id}/activity-calendar`, `GET …/{date}` |
| F-1 대시보드 | `GET /families/{id}/dashboard` |
| F-2 활동 추이 | `GET /seniors/{id}/metrics/daily` ×3 지표 |
| F-3 주간 리포트 | `GET /reports`, `GET /reports/{id}`, `POST /reports:generate` |
| F-4 용돈 관리 | `GET /reward-wallet`, `GET·PUT /reward-rules` |
| F-5 케어 알림 | `GET /families/{id}/care-alerts` |
| F-6 부모님 프로필 | `GET /families/{id}/members` |
| F-7 미션 히스토리 | `GET /seniors/{id}/missions/{mid}/history` |
| F-8 포인트 충전 | StoreKit + `POST /reward-wallet/charges` |
| F-9 참여 패턴 | `GET /seniors/{id}/pattern-summary` |
| F-10 보상 조건 | `GET·PUT /reward-rules` |
| F-11 알림 설정 | `GET·PUT /me/notification-settings` |
| F-12 부모님 활동 달력 (**신규 화면**) | S-19와 동일 엔드포인트 (guardian 읽기·감사 로그) |
| G-1 설정 | `GET /me`, `PATCH /me`, 로그아웃(Firebase) |
