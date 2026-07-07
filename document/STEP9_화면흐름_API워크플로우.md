# STEP 9. 화면 흐름 · API 워크플로우

**프로젝트명:** Feelio
**작성일:** 2026-07-06
**기준 문서:** `STEP3_사용자흐름_및_화면흐름_정리.md`(화면 명세), `STEP8_API명세서.md`(API 설계 초안)
**보조 문서:** STEP 4 요구사항정의서, STEP 5 기능명세서, STEP 6 ERD, STEP 7 데이터베이스설계서

**작성 원칙**

- STEP 8에 실제 정의된 API만 매핑한다. 화면에 필요한데 STEP 8에 없으면 **"API 추가 필요"**, API는 있는데 화면에서 쓰이지 않으면 **"사용 화면 확인 필요"**로 표시한다.
- 현재 백엔드는 0건(전부 프론트 로컬 상태 + mock)이므로, 각 기능은 **[현재: 로컬/mock]** 과 **[연동 대상 API]** 를 구분해 적는다.
- 화면 이동과, 같은 화면 안에서 상태만 바뀌는 흐름(🔄 표시)을 구분한다.
- 기록·목표 변경 후 **재조회(캐시 무효화)가 필요한 지점**을 ♻️로 표시한다.
- 문서에 근거가 없는 내용은 "추정" 또는 "추가 정의 필요"로 표시한다.

---

## 0. 공통 워크플로우

### 0-1. 앱 부팅 시퀀스 (모든 화면 공통 선행)

```text
앱 접속
→ 저장된 accessToken 존재?
   ├─ 없음 → LoginPage
   └─ 있음 → GET /api/users/me
        ├─ 200 → onboardingDone 확인 → false: OnboardingPage / true: HomePage
        ├─ 401 → POST /api/auth/token/refresh
        │        ├─ 성공 → GET /api/users/me 재시도
        │        └─ 실패 → 토큰 폐기 → LoginPage
        └─ 네트워크 오류 → 재시도 안내 화면 (추가 정의 필요)
```

- **[현재: 로컬]** localStorage의 `isLoggedIn`/`onboardingDone`으로만 분기 (`App.jsx`). 토큰·재발급 흐름은 연동 시 신규 구현.
- `POST /api/auth/token/refresh`는 특정 화면이 아니라 **공통 HTTP 인터셉터**에서 사용 (401 수신 시 자동 재발급 → 원 요청 재시도).

### 0-2. 마스터 데이터 로딩

- 로그인 직후(또는 첫 화면 진입 시) `GET /api/meta` 1회 호출 → 감정 8종(색상·정렬)·카테고리·상황 목록을 전역 상태에 캐시.
- 사용 화면: 기록(선택지 렌더링), 거래내역(필터 옵션), 거래 상세 모달(수정 폼), 홈(감정 색상·범례).
- 자주 바뀌지 않으므로 세션 내 캐시 유지, 앱 재시작 시 재조회. **[현재: 로컬]** `src/data/emotions.js` 상수가 이 역할.

### 0-3. 공통 에러 처리 (STEP 5 보완 기준)

- API 호출 중: 해당 버튼 Disabled + 로딩 표시. 타임아웃 5초.
- 401: 토큰 재발급 시도 → 실패 시 LoginPage 리다이렉트.
- 4xx: 필드 검증 에러는 인라인 텍스트, 그 외 토스트.
- 5xx/네트워크: "네트워크 연결이 불안정합니다" 토스트 + 재시도 허용.

---

## 1. LoginPage (S1)

### 화면 목적
서비스 첫인상(감정 말랑이 히어로)을 전달하고, 소셜 계정으로 Feelio에 진입시키는 화면.

### 진입 경로
- 앱 최초 접속 (토큰 없음)
- 로그아웃 / 회원탈퇴 후
- 토큰 만료·재발급 실패 후 리다이렉트

### 주요 기능 및 API 매핑

| 기능 | 사용자 행동 | 프론트 처리 | 필요 API | 성공 시 도달 상태 | 실패 시 처리 |
|---|---|---|---|---|---|
| Google/Kakao/Naver 로그인 | 소셜 버튼 클릭 | OAuth 시작 → provider 토큰 획득 → 서버 교환. **[현재: mock — 클릭 즉시 로그인]** | POST /api/auth/login | onboardingDone=false → OnboardingPage / true → HomePage | OAuth 거부·취소: 안내 토스트. 401: "인증에 실패했어요" 토스트 |
| 히어로 슬라이드 | (자동) 3.2초 간격 전환 | 로컬 타이머 🔄 상태 변경만 | 없음 (API 불필요) | 같은 화면 | — |
| 다크모드 토글 | ☾/☀ 클릭 | 로컬 상태 즉시 반영 🔄 | (로그인 전이므로 서버 저장 불가 — 로컬만) | 같은 화면 | — |

### 화면 흐름
1. LoginPage 진입 → 히어로 슬라이드 자동 재생.
2. 소셜 버튼 클릭 → OAuth 인증 (방식: 프론트 SDK vs 서버 콜백 — STEP 8 확인 항목 #2, 추가 정의 필요).
3. `POST /api/auth/login` → accessToken·refreshToken 저장, user(닉네임·프로필 이미지·onboardingDone) 수신. //메디스 db에 넣어서 
4. 신규 사용자면 서버가 users + social_accounts + notification_settings 기본값 + terms_agreements 생성 (STEP 8 4-1).
5. onboardingDone 분기 → OnboardingPage 또는 HomePage.

### 관련 API
POST /api/auth/login · (공통) POST /api/auth/token/refresh, GET /api/users/me

### 관련 DB 테이블
users, social_accounts, refresh_tokens, notification_settings, terms_agreements

### 예외 흐름
- 소셜 인증 거부/취소 → 로그인 화면 유지 + 토스트
- 네트워크 오류/타임아웃 → 토스트 + 재시도
- 탈퇴 계정(users.status=WITHDRAWN) 재로그인 → 정책 미정 (재가입 처리 vs 차단 — **추가 정의 필요**)
- 약관 동의 화면 부재 — 실서비스 필수이나 화면 없음 (**화면·API 흐름 추가 정의 필요**, terms_agreements는 로그인 시 일괄 생성으로 가정 중)

---

## 2. OnboardingPage (S2)

### 화면 목적
첫 로그인 사용자가 5단계로 첫 목표(대표 목표)를 설정하고 서비스를 시작하는 화면.

### 진입 경로
- 로그인 성공 + onboardingDone=false (유일한 진입 경로)

### 주요 기능 및 API 매핑

| 기능 | 사용자 행동 | 프론트 처리 | 필요 API | 성공 시 도달 상태 | 실패 시 처리 |
|---|---|---|---|---|---|
| 단계 이동 (1~4) | 목표·금액·기간·현재 금액 선택 후 "다음" | 로컬 폼 상태 누적 🔄 (필수값 없으면 버튼 비활성 — STEP 5 보완) | 없음 | 같은 화면, 다음 단계 | — |
| 목표 확정 | 5단계 요약에서 "시작하기" | 버튼 로딩 상태 → 목표 생성 + 온보딩 완료 처리. **[현재: 로컬 `completeOnboarding()`]** | POST /api/goals (isMain=true) → PATCH /api/users/me/onboarding | HomePage | 400(금액 최소값 미달): 인라인 안내. 실패 시 온보딩 화면 유지·재시도 |

### 화면 흐름
1. 1단계 목표명 → 2단계 목표 금액 → 3단계 기간 → 4단계 현재 금액 → 5단계 요약.
2. "시작하기" → `POST /api/goals` (name, targetAmount, currentAmount, startDate=오늘, dueDate=오늘+기간, isMain=true).
3. 성공 → `PATCH /api/users/me/onboarding` → onboardingDone=true.
4. 두 호출 모두 성공 시 HomePage 이동. (순서 보장 필요 — 목표 생성 성공 후 온보딩 완료 처리. 중간 실패 시 재시도 규칙 **추가 정의 필요**)

### 관련 API
POST /api/goals · PATCH /api/users/me/onboarding

### 관련 DB 테이블
goals, users(onboarding_done)

### 예외 흐름
- 기간 선택값 미저장 (현재 코드 결함) → 연동 시 dueDate 계산에 필수 반영 (STEP 7 보완 #3)
- 목표 금액 0원 → 최소값 검증으로 차단 (STEP 5 보완)
- 온보딩 중 이탈(새로고침) → onboardingDone=false 유지 → 재진입 시 처음부터 (단계 저장은 **추가 정의 필요**)

---

## 3. HomePage (S3)

### 화면 목적
감정 소비 회고의 중심 화면. 선택한 날의 대표 감정 말랑이, 감정 캘린더, 감정 능선, 목표 진행률, AI 감정 신호를 보여준다.

### 진입 경로
- 온보딩 완료 직후 / 로그인 직후 (기본 탭)
- 사이드바·하단 탭 "홈"

### 주요 기능 및 API 매핑

| 기능 | 사용자 행동 | 프론트 처리 | 필요 API | 성공 시 도달 상태 | 실패 시 처리 |
|---|---|---|---|---|---|
| 월간 감정 캘린더 렌더 | 화면 진입 | 표시 월 기준 조회. **[현재: 로컬 집계]** | GET /api/summary/calendar?year&month | 날짜별 감정 색 표시 🔄 | 실패 시 빈 캘린더 + 재시도 버튼 (추가 정의 필요) |
| 월 이동 | ‹ › 클릭 | visibleMonth 변경 → 해당 월 재조회 ♻️ | GET /api/summary/calendar (새 연·월) | 같은 화면, 캘린더·능선 갱신 🔄 | 이전 데이터 유지 + 토스트 |
| 날짜 선택 | 날짜 1회 클릭 | selectedDate 변경 🔄 → 그 날 대표 감정 말랑이 갱신 (calendar summary의 dominantEmotion 재사용 — 별도 호출 불필요) | (기존 응답 재사용) | 같은 화면, 말랑이·문구 갱신 | — |
| 날짜 재클릭 (600ms 내 또는 선택된 날짜) | 같은 날짜 다시 클릭 | 화면 이동 | — | **TransactionsPage** (해당 날짜 컨텍스트 전달 — 전달 방식 추가 정의 필요) | — |
| 감정 능선 / AI 감정 신호 | 화면 진입 | 이번 달 감정 분포 + 전월 비교. **[현재: 능선은 로컬 집계, 신호는 하드코딩]** | GET /api/summary/emotions?year&month | 능선(8종 축)·증감 칩 표시 🔄. 감정 기록 5건 미만이면 빈 상태 | 빈 상태 문구 표시 |
| 목표 카드 | 화면 진입 | 대표 목표 조회. **[현재: 로컬 goals[0]]** | GET /api/goals (isMain 우선 표시) | 진행률 카드 표시 | 목표 없음 → 목표 만들기 유도 (화면 추가 정의 필요) |
| 목표 카드 클릭 | 카드 클릭 | 화면 이동 | — | **UniversePage** | — |
| "분석 자세히 보기" | 링크 클릭 | 화면 이동 | — | **AnalysisPage** | — |

### 화면 흐름 (진입 시 병렬 조회)
```text
HomePage 진입
→ (병렬) GET /api/summary/calendar + GET /api/summary/emotions + GET /api/goals
→ 캘린더 색·말랑이·능선·신호·목표 카드 렌더
→ 날짜 클릭 🔄 / 월 이동 ♻️ / 재클릭 → TransactionsPage / 목표 카드 → UniversePage
```

### 관련 API
GET /api/summary/calendar · GET /api/summary/emotions · GET /api/goals

### 관련 DB 테이블
transactions, transaction_situations(집계 시), emotions, goals, monthly_summaries(캐시)

### 예외 흐름
- 기록 0건: 물음표 말랑이 + 빈 능선 + "첫 기록 유도" (구현됨)
- '오늘' 고정값(2026-07-01) → 연동 시 실제 오늘 기준 (교체 대상)
- 대표 감정 동률 → "최근 기록 우선" 서버 규칙 (팀 확정 필요)

---

## 4. RecordPage (S4 — 지출·수입 기록)

### 화면 목적
하루의 소비·수입을 감정과 함께 남기는 입력 화면. Feelio 핵심 루프의 시작.

### 진입 경로
- 사이드바·하단 탭 "지출·수입"

### 주요 기능 및 API 매핑

| 기능 | 사용자 행동 | 프론트 처리 | 필요 API | 성공 시 도달 상태 | 실패 시 처리 |
|---|---|---|---|---|---|
| 입력 폼 렌더 | 화면 진입 | 감정·카테고리·상황 선택지 렌더. **[현재: 하드코딩 상수]** | GET /api/meta (캐시 재사용) | 폼 표시 | meta 실패 시 입력 불가 안내 + 재시도 |
| 지출/수입 토글·금액·감정·카테고리·상황·메모·일시 입력 | 각 요소 선택/입력 | 로컬 폼 상태 🔄. 금액 정규식(0 초과 정수), 메모 500자 제한·살균(STEP 5 보완). 필수 3종 충족 시 저장 버튼 활성 | 없음 | 같은 화면 | 인라인 검증 안내 |
| 기록 저장 | "감정 기록 저장하기" 클릭 | 버튼 Disabled+로딩 → 저장 → 성공 시 토스트·폼 초기화·홈 선택 날짜 갱신(로컬 상태). **[현재: 로컬 `addTransaction()` — 상황 첫 값만 저장(교체 대상)]** | POST /api/transactions (emotionId, categoryId, situationIds[], …) | 같은 화면 유지 (🔄 폼 리셋). ♻️ 무효화: transactions 목록, summary/calendar, summary/emotions, analysis | 400: 인라인 안내. 네트워크: "네트워크 연결이 불안정합니다" 토스트, 입력값 보존 |

### 화면 흐름
```text
RecordPage 진입 → (meta 캐시로) 폼 렌더
→ 입력 (금액 → 감정 말랑이 → 카테고리 → 상황·메모·일시) 🔄
→ 저장 → POST /api/transactions (단일 트랜잭션: transactions + transaction_situations)
→ 201 → 토스트 "기록 저장됨" → 폼 초기화 🔄
→ ♻️ 캐시 무효화: 거래내역 목록 / 홈 캘린더·능선·신호 / 분석 (다음 진입 시 재조회)
→ 홈 selectedDate = 기록 날짜 (로컬 상태 전달)
```
- 화면 이동은 없음 (기록 화면 유지). 낙관적 업데이트 적용 여부는 STEP 5 보완 기준 "낙관적 또는 응답 후 갱신" — 최종 택1 **추가 정의 필요**.

### 관련 API
GET /api/meta · POST /api/transactions

### 관련 DB 테이블
transactions, transaction_situations, emotions, categories, situations, monthly_summaries(갱신)

### 예외 흐름
- 필수(금액·감정·카테고리) 누락 → 저장 비활성 + 안내 문구 (구현됨)
- 존재하지 않는 emotionId 등 → 400 (서버 검증)
- 일시 기본값 고정(2026-07-01T21:30) → 현재 시각으로 교체 대상

---

## 5. TransactionsPage (S5) + 거래 상세 모달 (S6)

### 화면 목적
쌓인 기록을 일별/월별/감정별로 되돌아보고, 개별 기록을 확인·수정·삭제하는 화면.

### 진입 경로
- 사이드바·하단 탭 "거래내역"
- 홈 캘린더 날짜 재클릭 (해당 날짜 컨텍스트)

### 주요 기능 및 API 매핑

| 기능 | 사용자 행동 | 프론트 처리 | 필요 API | 성공 시 도달 상태 | 실패 시 처리 |
|---|---|---|---|---|---|
| 목록 조회 | 화면 진입 | 현재 연·월 기준 조회. **[현재: 로컬 필터링 + 기록 0건 시 목데이터 표시(제거 대상)]** | GET /api/transactions?year&month | 그룹 목록 + 합계 표시 | 실패: 빈 목록 + 재시도 |
| 월 이동 | ‹ › 클릭 | 연·월 변경 → 재조회 ♻️ | GET /api/transactions (새 기간) | 같은 화면 갱신 🔄 | 이전 데이터 유지 + 토스트 |
| 뷰 전환 (일별/월별/감정별) | 탭 클릭 | **프론트 그룹핑만 변경 🔄 (재조회 불필요 — STEP 8 설계: 서버는 평면 목록 반환)** | 없음 | 같은 화면, 그룹 재배열 | — |
| 검색 | 검색어 입력 | 디바운스 후 재조회 ♻️ (또는 로컬 필터 — 방식 추가 정의 필요) | GET /api/transactions?query= | 같은 화면, 결과 갱신 🔄 | 결과 0건: 빈 목록 (안내 문구 개선 검토) |
| 필터·정렬 | 필터 패널에서 연도/월-일/정렬/카테고리/감정 선택 | 조건 변경 시 재조회 ♻️. 필터 옵션은 meta 사용 | GET /api/transactions?emotionId=&categoryId=&sort= | 같은 화면, 결과 갱신 🔄 | — |
| 기록 상세 | 행 클릭 | 모달 오픈 — 목록 응답 객체 재사용 | (선택) GET /api/transactions/{id} — **사용 화면 확인 필요** (목록 데이터로 충분, 딥링크·최신화 필요 시만 사용) | 상세 모달 표시 | 404: "삭제된 기록" 토스트 + 모달 닫기 |
| 기록 수정 | 모달 "수정" → 폼 수정 → 저장 | 수정 폼(meta 선택지 기반 — 현재 자유 텍스트는 교체 대상) → 저장 | PUT /api/transactions/{id} | 모달 상세 뷰 복귀 🔄. ♻️ 목록·홈 요약·분석 무효화 | 400/403/404 처리, 실패 시 폼 유지 |
| 기록 삭제 | 모달 "삭제" → 확인 다이얼로그(정책 #5) | 확인 후 삭제 | DELETE /api/transactions/{id} | 모달 닫힘, 목록에서 제거 🔄. ♻️ 홈 요약·분석 무효화 | 404: 이미 삭제됨 안내 |

### 화면 흐름 — 기록 수정/삭제 후 데이터 갱신
```text
행 클릭 → 상세 모달 → 수정 저장(PUT) 또는 삭제(DELETE)
→ 성공 토스트 → 목록 즉시 반영 🔄
→ ♻️ 무효화: summary/calendar, summary/emotions (홈), analysis/monthly, universe/simulation
   (해당 화면 재진입 시 재조회 — monthly_summaries 서버 캐시도 갱신, STEP 7 보완 #5)
```

### 관련 API
GET /api/transactions · GET /api/transactions/{id}(선택) · PUT /api/transactions/{id} · DELETE /api/transactions/{id} · GET /api/meta

### 관련 DB 테이블
transactions, transaction_situations, emotions, categories, situations, monthly_summaries

### 예외 흐름
- 기록 0건 → 빈 상태 안내 (목데이터 대체 표시는 제거 확정 — 정책 #3)
- 타인 기록 접근(403) → 발생 불가 설계이나 방어적 처리
- 필터 조합 결과 0건 → 빈 목록 + 필터 초기화 버튼 (추가 정의 필요)

---

## 6. AnalysisPage (S7 — AI 분석)

### 화면 목적
감정에 따른 소비 패턴 분석 인사이트(감정·카테고리·시간대 집계, 위험 신호, 챌린지)를 제공하는 화면.

### 진입 경로
- 사이드바·하단 탭 "AI 분석"
- 홈 AI 감정 신호 카드 "분석 자세히 보기"

### 주요 기능 및 API 매핑

| 기능 | 사용자 행동 | 프론트 처리 | 필요 API | 성공 시 도달 상태 | 실패 시 처리 |
|---|---|---|---|---|---|
| 월간 분석 조회 | 화면 진입 | 기준 월 분석 로드. **[현재: 전부 하드코딩 상수]** | GET /api/analysis/monthly?year&month (3순위, 응답 스키마 확정 필요) | KPI 카드·예산 현황·차트·인사이트 표시 | 데이터 부족: "기록이 더 쌓이면 분석해 드려요" 빈 상태 (추가 정의 필요) |
| 소비 코어 차트 탭 전환 | [가장 많이 쓴 곳/주로 쓴 시간/주된 감정] 클릭 | 응답 내 데이터 전환만 🔄 (재조회 불필요) | 없음 | 같은 화면, 도넛 차트 전환 | — |
| 인사이트 카드 뒤집기 | 카드 클릭 | 로컬 상태 🔄 | 없음 | 같은 화면 | — |
| AI 맞춤 챌린지 | (조회만) | **[현재: 하드코딩]** | **API 추가 필요** (챌린지 생성·진행 — STEP 8에서 4순위 제외) | — | — |
| 누수율 KPI 배지 | — | **제거 대상 잔여 UI** (누수율 제거 확정) | 없음 (API 만들지 않음) | — | — |

### 관련 API
GET /api/analysis/monthly (3순위)

### 관련 DB 테이블
transactions, transaction_situations, emotions, categories, monthly_summaries, ai_insights

### 예외 흐름
- 기록 부족 시 빈 상태 처리 없음 (현재) → 실데이터 전환 시 필수 정의
- 월 이동 UI가 분석 화면에 없음 — 기준 월 선택 기능 여부 **추가 정의 필요**

---

## 7. UniversePage (S8 — 평행우주)

### 화면 목적
"지금처럼 소비한 나" vs "감정소비를 줄인 나" 두 미래를 비교 체험시키고 목표와 연결하는 화면.

### 진입 경로
- 사이드바·하단 탭 "평행우주"
- 홈 목표 카드 클릭

### 주요 기능 및 API 매핑

| 기능 | 사용자 행동 | 프론트 처리 | 필요 API | 성공 시 도달 상태 | 실패 시 처리 |
|---|---|---|---|---|---|
| 시뮬레이션 데이터 로드 | 화면 진입 | 대표 목표 기준 두 시나리오 수치·내레이션 로드. **[현재: 고정 U_DATA]** | GET /api/universe/simulation?goalId (3순위, 스키마 확정 필요) | 두 행성·콘솔 표시 | 실패: 연출만 표시 + 수치 "계산 중" 처리 (추가 정의 필요) |
| 우주(행성) 선택 | 행성 클릭 | phase: idle→flying(1.2s)→result 🔄 — 로컬 연출 | 없음 (로드된 데이터 재사용) | 같은 화면, 결과 패널 | — |
| 내레이션 전환 | 말랑이 클릭 | narrativeIndex 순환 🔄 | 없음 | 같은 화면 | — |
| 콘솔 레버 조작 (시뮬레이션 조건 변경) | 레버 드래그 | leverA/B 상태 변경 🔄. **[현재: 연출용 — 수치 계산 미연결]** | **API 추가 필요 또는 파라미터 확장** — GET /api/universe/simulation?reductionRate= 형태로 조건 반영 (추정, **추가 정의 필요**. `planetScoring.mjs` 프론트 계산 활용안도 검토) | 결과 수치 갱신 🔄 ♻️ | — |
| 이스터에그 | 콘솔 점화 | 로컬 연출 🔄 | 없음 | 같은 화면 | — |

### 관련 API
GET /api/universe/simulation (3순위) · GET /api/goals (대표 목표 컨텍스트)

### 관련 DB 테이블
transactions, emotions, goals, monthly_summaries

### 예외 흐름
- 대표 목표 없음 → 목표 설정 유도 (추가 정의 필요)
- 기록 부족으로 시뮬레이션 불가 → 최소 기록 수 기준 정의 필요 (추정)

---

## 8. Profile/Settings Modal (S9)

### 화면 목적
계정(프로필·탈퇴), 목표, 태그, 알림, 화면 설정, 데이터 관리를 담당하는 공통 모달.

### 진입 경로
- 데스크톱: 사이드바 하단 프로필 영역
- 모바일: **진입 경로 없음 — 화면 추가 필요** (STEP 3 9절)

### 주요 기능 및 API 매핑

| 기능 | 사용자 행동 | 프론트 처리 | 필요 API | 성공 시 도달 상태 | 실패 시 처리 |
|---|---|---|---|---|---|
| 프로필 표시 | 모달 열기 | 사용자 정보 표시 (아바타=소셜 프로필 이미지 확정, 현재 첫 글자) | GET /api/users/me (캐시 재사용) | 모달 메인 | — |
| 닉네임 수정 | 프로필 수정 → 저장 | **[현재: 로컬 `updateUser()`]** | PATCH /api/users/me | 모달 메인 복귀 🔄, 사이드바 반영 ♻️(users/me 캐시 갱신) | 400: 인라인 안내 |
| 목표 목록 | 목표 관리 진입 | **[현재: 로컬 goals 표시]** | GET /api/goals | 목표 목록 표시 | — |
| 목표 추가/수정/삭제 | +추가 / 수정 / 삭제 | **[현재: UI만 — 버튼이 화면 복귀만 수행, 스토어 액션도 없음]** 폼 화면 자체도 미구현 — **화면 추가 정의 필요** | POST /api/goals · PUT /api/goals/{id} · DELETE /api/goals/{id} | 목록 갱신 🔄. ♻️ 홈 목표 카드·평행우주 무효화. isMain 변경 시 서버가 기존 대표 해제 | 400/404 처리 |
| 태그 관리 | 태그 관리 진입 | **[현재: UI만 — 잠금 감정 8종 + 하드코딩 상황 태그]** 정책 #6: 이번 범위 메뉴 숨김 권장 | 커스텀 태그 CRUD — **API 추가 필요 (범위 제외 상태, 화면 유지 여부 팀 확정 필요)** | — | — |
| 알림 설정 | 토글·시간 변경 | **[현재: UI만 — 로컬 토글]** | 알림 설정 조회/저장 — **API 추가 필요** (테이블 notification_settings는 STEP 7에 존재, API는 STEP 8 4순위 제외) | 설정 저장 🔄 | — |
| 화면·오로라 설정 | 다크모드 스위치·테마 선택 | 즉시 반영 🔄 **[현재: 로컬]** | PATCH /api/users/me/settings | 같은 화면, 전역 테마 갱신 | 실패 시 로컬 유지 + 토스트 (오프라인 허용 — 추정) |
| 데이터 백업 | 백업 버튼 | **[현재: UI만]** | **API 추가 필요 (4순위)** | — | — |
| 데이터 초기화 | 초기화 버튼 → 확인 다이얼로그(정책 #5) | **[현재: 로컬 `resetData()` — 즉시 실행]** | DELETE /api/transactions | 토스트, 홈·거래내역 빈 상태 ♻️ (전 화면 요약·분석 무효화) | 실패 토스트 |
| 로그아웃 | 로그아웃 버튼 | 토큰 폐기 | POST /api/auth/logout | LoginPage (onboardingDone은 서버 보존 — 정책 #7) | 실패해도 로컬 토큰 폐기 후 이동 |
| 회원탈퇴 | 탈퇴 → 사유 설문(STEP 5 보완) → 확인 | **[현재: 로그아웃과 동일]** | DELETE /api/users/me — 탈퇴 사유 필드는 STEP 8에 없음 → **API 보완 필요 (reason 파라미터)** | LoginPage, 계정 데이터 CASCADE 삭제 | 재확인 다이얼로그 필수 |

### 관련 API
GET·PATCH·DELETE /api/users/me · PATCH /api/users/me/settings · goals CRUD 4종 · DELETE /api/transactions · POST /api/auth/logout

### 관련 DB 테이블
users, social_accounts, goals, notification_settings, terms_agreements, refresh_tokens, transactions(초기화·탈퇴 시)

### 예외 흐름
- 모바일 진입 불가 (하단 탭에 프로필 버튼 없음) — 추가 구현 필요
- 탈퇴 후 재로그인 정책 미정 (1절 참조)

---

## 9. 필수 화면 흐름 11종 요약

| # | 흐름 | 경로 · API | 구분 |
|---|---|---|---|
| 1 | 로그인 후 온보딩 분기 | LoginPage → POST /api/auth/login → onboardingDone ? HomePage : OnboardingPage | 화면 이동 |
| 2 | 온보딩 완료 후 홈 진입 | OnboardingPage → POST /api/goals + PATCH /api/users/me/onboarding → HomePage | 화면 이동 |
| 3 | 홈 → 기록 화면 이동 | HomePage → 탭 "지출·수입" → RecordPage (+ GET /api/meta 캐시) | 화면 이동 |
| 4 | 기록 등록 후 데이터 갱신 | RecordPage → POST /api/transactions → 🔄 폼 리셋 → ♻️ transactions·summary/calendar·summary/emotions·analysis 무효화 → 홈/거래내역 재진입 시 재조회 | 상태 변경 + 무효화 |
| 5 | 홈 캘린더 월 이동/일자 선택 | 월 이동: GET /api/summary/calendar 재조회 ♻️ / 일자 선택: 응답 재사용 🔄 (말랑이 갱신) / 재클릭: TransactionsPage 이동 | 혼합 |
| 6 | 거래내역 일별/월별/감정별 조회 | GET /api/transactions (평면 목록) → 뷰 전환은 프론트 그룹핑만 🔄 | 상태 변경 |
| 7 | 거래내역 검색/필터/정렬 | GET /api/transactions?query&emotionId&categoryId&sort ♻️ (조건 변경 시 재조회) | 상태 변경 + 재조회 |
| 8 | AI 분석 조회 | AnalysisPage → GET /api/analysis/monthly → 차트 탭 전환은 로컬 🔄 | 화면 이동 + 상태 변경 |
| 9 | 평행우주 조건 변경 → 결과 갱신 | GET /api/universe/simulation 로드 → 행성 선택·내레이션은 로컬 🔄 / 레버 조건 반영은 파라미터 확장 필요 (**추가 정의 필요**) | 상태 변경 |
| 10 | 프로필/설정 변경 | 목표: goals CRUD (♻️ 홈·평행우주) / 태그: API 추가 필요 / 알림: API 추가 필요 / 화면: PATCH /api/users/me/settings 🔄 | 상태 변경 + 무효화 |
| 11 | 기록 수정/삭제 후 갱신 | 상세 모달 → PUT·DELETE /api/transactions/{id} → 목록 즉시 반영 🔄 → ♻️ 홈 요약·분석·평행우주 무효화 | 상태 변경 + 무효화 |

**캐시 무효화 매트릭스 (♻️ 지점)**

| 데이터 변경 | 무효화 대상 |
|---|---|
| 기록 생성/수정/삭제/전체 초기화 | GET /api/transactions, /api/summary/calendar, /api/summary/emotions, /api/analysis/monthly, /api/universe/simulation (+ 서버 monthly_summaries 갱신) |
| 목표 생성/수정/삭제/대표 변경 | GET /api/goals, /api/universe/simulation |
| 프로필/설정 변경 | GET /api/users/me |

---

## 10. API 누락/보완 필요 목록

| 화면 | 기능 | 필요한 API | 현재 STEP8 존재 여부 | 비고 |
|---|---|---|---|---|
| Profile Modal | 알림 설정 조회/저장 | GET·PUT /api/users/me/notifications (가칭) | ❌ 없음 (4순위 제외 명시) | 테이블(notification_settings)은 STEP 7에 존재 — API만 추가 필요 |
| Profile Modal | 데이터 백업 | POST /api/backup (가칭) | ❌ 없음 (4순위) | 기능 자체 후순위 |
| Profile Modal | 회원탈퇴 사유 수집 | DELETE /api/users/me에 reason 필드 | △ API는 있으나 사유 필드 없음 | STEP 5 보완(탈퇴 설문)과 정합 필요 |
| Profile Modal | 커스텀 태그 CRUD | /api/tags (가칭) | ❌ 없음 (범위 제외) | 태그 관리 화면 유지 여부 팀 확정 필요 (정책 #6은 숨김 권장) |
| AnalysisPage | AI 맞춤 챌린지 | /api/challenges (가칭) | ❌ 없음 (4순위) | 테이블도 확장 제안 상태 |
| UniversePage | 레버(조건) 반영 시뮬레이션 | GET /api/universe/simulation 파라미터 확장 (reductionRate 등) | △ API는 있으나 조건 파라미터 없음 | 프론트 계산(planetScoring) 대안 검토 — 추가 정의 필요 |
| LoginPage | 약관 동의 수집 화면·API | POST /api/terms/agreements (가칭) 또는 login 파라미터 | △ 로그인 시 일괄 생성으로 가정 중 | 실서비스 법적 필수 — 흐름 확정 필요 |
| 공통 | 재시도/오류 안내 화면 | — (화면 정의) | — | 부팅 실패·데이터 로드 실패 시 UX 추가 정의 필요 |

**사용 화면 확인 필요 API**

| API | 상태 |
|---|---|
| GET /api/transactions/{transactionId} | 상세 모달이 목록 응답 객체를 재사용하면 불필요. 딥링크·항상 최신 조회가 필요할 때만 사용 — 확인 필요 |

## 11. 화면별 API 매핑 요약

| 화면 | 사용 API | 목적 |
|---|---|---|
| 공통(부팅·인터셉터) | GET /api/users/me, POST /api/auth/token/refresh, GET /api/meta | 세션 복원·토큰 재발급·마스터 캐시 |
| LoginPage | POST /api/auth/login | 소셜 로그인·계정 생성 |
| OnboardingPage | POST /api/goals, PATCH /api/users/me/onboarding | 첫 목표 생성·온보딩 완료 |
| HomePage | GET /api/summary/calendar, GET /api/summary/emotions, GET /api/goals | 캘린더·말랑이·능선·신호·목표 카드 |
| RecordPage | GET /api/meta, POST /api/transactions | 입력 폼·기록 등록 |
| TransactionsPage(+상세 모달) | GET /api/transactions, (GET /{id} 확인 필요), PUT /{id}, DELETE /{id}, GET /api/meta | 조회·검색·필터·수정·삭제 |
| AnalysisPage | GET /api/analysis/monthly | 월간 분석 인사이트 |
| UniversePage | GET /api/universe/simulation, GET /api/goals | 미래 비교 시뮬레이션 |
| Profile/Settings Modal | GET·PATCH·DELETE /api/users/me, PATCH /api/users/me/settings, goals CRUD, DELETE /api/transactions, POST /api/auth/logout | 계정·목표·설정·데이터 관리 |

## 12. 전체 사용자 흐름 요약

```text
[앱 접속]
   │  (토큰 없음)
   ▼
LoginPage ──소셜 로그인(POST /auth/login)──► 온보딩 완료?
   ▲                                          │
   │ 로그아웃/탈퇴/토큰 만료                    ├─ 아니오 ─► OnboardingPage
   │                                          │              │ 목표 설정(POST /goals)
   │                                          │              │ 완료(PATCH /users/me/onboarding)
   │                                          │              ▼
   │                                          └─ 예 ────► HomePage ◄──────────────┐
   │                                                        │                      │
   │                     ┌──────────────┬──────────────┬────┴───────┬──────────┐   │
   │                     ▼              ▼              ▼            ▼          ▼   │
   │               RecordPage    TransactionsPage  AnalysisPage  UniversePage  │   │
   │               기록 저장        조회/검색/필터     월간 분석      미래 비교    │   │
   │               (POST /tx)     (GET /tx)        (GET/analysis) (GET/universe)   │
   │                     │              │                                          │
   │                     │        상세 모달(PUT·DELETE /tx/{id})                    │
   │                     └──────♻️ 저장·수정·삭제 시 홈/분석/평행우주 데이터 무효화──┘
   │
   └── Profile/Settings Modal (사이드바 프로필)
        닉네임(PATCH /users/me) · 목표(goals CRUD) · 설정(PATCH /settings)
        데이터 초기화(DELETE /tx) · 로그아웃(POST /logout) · 탈퇴(DELETE /users/me)
```
