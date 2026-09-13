<div align="center">

# 🏠 이번집 (Second-Zip)

**전세사기 유형별 위험 분석·예방 서비스**

![Java](https://img.shields.io/badge/Java-17-orange)
![Spring](https://img.shields.io/badge/Spring%20MVC-5.3-6db33f)
![Spring Security](https://img.shields.io/badge/Spring%20Security-5.8-6db33f)
![MyBatis](https://img.shields.io/badge/MyBatis-black)
![MySQL](https://img.shields.io/badge/MySQL-8-4479A1?logo=mysql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?logo=redis&logoColor=white)
![Vue](https://img.shields.io/badge/Vue-3-42b883?logo=vue.js&logoColor=white)

KB IT's Your Life 7기 · 24반 3팀 (5인)

**본 저장소 작성자는 백엔드 리포트(RE) 도메인을 담당했습니다**

</div>

---

## 서비스 개요

전세 매물의 주소와 보증금을 입력하면, 여러 공공·민간 데이터를 결합해 **위험을 판정**하고 계약 체크리스트와 녹취 기반 AI 확인까지 이어주는 서비스입니다.

판정 범위는 **필수 점검 5개 · 전세사기 유형 3개 · 유형별 상세 판정 3개, 총 9개**입니다. 주요 기준은 전세가율 70/80%, 선순위채권 54%, HUG 기준가 90%, 수도권 7억·비수도권 5억 한도 등입니다.

> ### 설계 원칙 — "확답이 아닌 도움"
>
> 이 서비스는 "이 집은 안전합니다"라고 말하지 않습니다. 전세사기 판정은 최종적으로 법률적 판단이고, 데이터만으로 단정하면 사용자가 잘못된 확신을 갖게 됩니다.
> **판정 보조 도구**라는 위치를 지키는 것이 모든 기술적 결정의 기준이었습니다. 아래 설계들은 전부 이 원칙에서 나왔습니다.

---

## 팀 구성과 담당

**5인 팀**(프론트엔드 2 · 백엔드 3)이며, 저는 **백엔드 리포트(RE) 도메인**을 맡았습니다.

| 영역 | 내용 |
|---|---|
| 분석 워크플로 | CODEF 양방향 인증 상태 머신, 분산 락, 멱등 처리 |
| 외부 API 계층 | 등기·건축물대장·실거래가·주소 클라이언트와 파서, 캐시·타임아웃 정책 |
| 위험 판정 엔진 | 위험도와 데이터 확보 상태를 분리한 판정 모델 |
| 리포트 | 저장·조회·공유·즐겨찾기·체크리스트 자동 체크 |
| 품질 | report 도메인 테스트 보강 |

인증/마이페이지/약관 도메인과 배포 파이프라인은 다른 백엔드 팀원이 담당했습니다.

---

## 백엔드에서 실제로 푼 문제

### 1. 한 번의 요청으로 끝나지 않는 외부 인증

CODEF 건축물대장 발급은 **동기 요청 한 번으로 끝나지 않습니다.** 간편인증, 주소·동·호 선택, CAPTCHA 같은 추가 입력이 요청 사이에 끼어듭니다. 서버 재시작, 중복 클릭, HTTP 재시도까지 고려하면 컨트롤러 메서드 하나나 세션 메모리로는 감당할 수 없습니다.

분석 요청 자체를 **상태 머신**으로 모델링했습니다.

```
AUTH_REQUIRED → AUTH_PENDING / SELECTION_REQUIRED → PROCESSING → COMPLETED / FAILED
```

- 주소 검색에서 얻은 법정동코드·본번·부번을 UUID `addressId`로 Redis에 보관해, 이후 단계가 주소를 다시 조회하지 않게 했습니다
- CODEF의 `jobIndex`·`threadIndex`·`jti`·`twoWayTimestamp`와 사용자 선택을 `AnalysisWorkflowState`에 저장 (TTL 15분)
- 조회할 때 `accountId` 소유권을 함께 확인하고, **만료·미존재·타인 소유를 전부 같은 404로** 응답해 요청 존재 여부가 노출되지 않게 했습니다

### 2. 유료 API 중복 과금 막기

등기부등본 조회는 **건당 과금**됩니다. 더블클릭, 네트워크 재시도, DB 저장 직후 응답 단절 — 모두 중복 과금으로 이어집니다.

| 층 | 방법 | 막는 것 |
|---|---|---|
| 진입 | Redis `SET NX` + TTL 요청 락 | 동시에 들어온 중복 요청 |
| 해제 | UUID 토큰 + Lua compare-and-delete | 만료된 뒤 남의 락을 잘못 지우는 것 |
| 순서 | 무료 데이터 선검증 후 유료 호출 | 애초에 분석 불가한 건의 과금 |
| 저장 | `analysis_reports.request_id` UNIQUE | 락을 빠져나간 중복의 최종 저지 |

락 해제에 Lua를 쓴 이유는, 소유권 확인과 삭제 사이가 원자적이지 않으면 **오래 걸린 실행이 남의 새 락을 지우는 일**이 생기기 때문입니다.

> **다만 이건 완전한 단일 실행 보장이 아닙니다.**
> 락 TTL이 300초인데 CODEF read timeout도 300초이고 락 갱신 기능이 없습니다. 처리가 TTL을 넘기면 실행이 겹칠 수 있습니다. 현재 구현은 **정상 처리 구간의 중복을 줄이는 장치**이며, 운영 수준에서는 lock renewal/watchdog과 UNIQUE 충돌 후 기존 결과 재조회가 필요합니다.

### 3. "확인 불가"가 "안전"으로 읽히지 않게

가장 위험한 실패는 **데이터를 못 가져왔는데 위험 없음으로 표시되는 것**입니다. 사용자는 그걸 "안전하다"로 읽습니다. 외부 API 실패나 금액 파싱 실패가 `0`이나 `false`로 대체되면 위험 매물이 SAFE가 됩니다.

그래서 판정 결과 `Judgement`를 두 축으로 분리했습니다.

| DataStatus | 의미 | 집계 |
|---|---|---|
| `VERIFIED` | 데이터를 확보해 실제 판정 | 위험도에 반영 |
| `UNVERIFIED` | 데이터 부족·파싱 실패로 확인 불가 | **CAUTION으로 집계에 참여** |
| `NOT_APPLICABLE` | 해당 주택에 적용되지 않는 규칙 | 집계에서 제외 |

점검 근거는 JSON `evidence`로 함께 저장해 판정 결과를 추적할 수 있게 했습니다.

### 4. 외부 데이터를 그대로 믿지 않기

외부 응답을 위험 엔진에 바로 넘기지 않고 내부 도메인으로 정규화합니다. 실제로 판정을 뒤집는 것은 대부분 이 단계의 디테일이었습니다.

**등기부등본**

- 말소된 권리 표시를 제외
- 근저당 표현은 있는데 **금액 파싱에 실패하면 0원이 아니라 `null`로 유지** — 0원으로 두면 안전으로 판정됩니다
- 신탁 이후 발생한 권리침해와 신탁 이전 권리를 구분
- 소유자가 신탁회사인 경우 신탁 문구 누락을 보완

**공시가격**

과거 최대 금액이 아니라 **기준일이 가장 최근인 가격**을 선택합니다. 날짜가 전혀 없을 때만 최대값으로 폴백합니다. 가격 하락기에 과거 최고가를 쓰면 전세가율이 낮게 계산되어 위험을 과소평가하기 때문입니다. 최신 4.2억·과거 5억일 때 4.2억으로 판정하는 것을 단위 테스트로 고정했습니다.

**주소**

카카오 주소 검색 결과가 없으면 장소 검색으로 폴백하고, 프론트에는 Redis에 저장한 **후보 ID만 노출**해 분석 대상 식별값 변조를 줄였습니다.

### 5. LLM 응답을 업무 데이터에 반영하기 전 3단계 방어

LLM 출력은 비결정적이라 DB에 바로 저장하지 않습니다.

1. **strict JSON Schema** — 응답 구조를 모델 단에서 제한
2. **애플리케이션 불변식** — 특약은 3~5개·제목 50자·본문 200자·중복 거부, 체크리스트는 요청한 item ID가 정확히 한 번씩 존재하는지와 0~1 신뢰도 검증
3. **트랜잭션 반영** — 특약 교체는 리포트 행을 `SELECT ... FOR UPDATE`로 잠근 뒤 delete/insert

실시간 녹취는 **provisional / final을 분리**합니다. 중간 분석이 CHECKED를 반환해도 실제 체크리스트를 바꾸지 않고 JVM 메모리에 후보로만 두었다가, 녹음 종료 후 전체 녹취를 다시 분석한 최종 결과만 반영합니다.

### 6. 외부 API 특성별 타임아웃 분리

| 대상 | connect / read |
|---|---|
| 일반 API | 3초 / 10초 |
| 특약 GPT | 3초 / 30초 |
| CODEF 장시간 작업 | 5초 / 300초 |

모든 호출에 같은 타임아웃을 쓰면 느린 문서 발급이 정상인데도 실패하고, 반대로 일반 API 장애가 요청 스레드를 오래 점유합니다. CODEF 토큰은 만료 60초 전 선제 갱신하고, **401일 때만** 캐시를 폐기하고 한 번 재시도합니다. 무조건 재시도하지 않는 이유는 유료·비멱등 호출이 중복 실행될 수 있기 때문입니다.

### 7. 유료 API 없이도 개발할 수 있게

`RegistryDataProvider` · `PriceDataProvider` · `BuildingRegisterGateway` 인터페이스와 Spring `Condition`(`RealApiCondition` / `MockApiCondition`)으로 실제 API와 Mock을 전환합니다.

건축물대장 Mock은 **실제 CODEF 필드 구조를 그대로 사용**해 파서까지 검증하고, 등기·가격 Mock은 정규화된 내부 도메인을 반환해 오케스트레이션을 검증합니다. 유료 호출 없이 전체 흐름을 돌릴 수 있어, 개발 중 과금과 팀원 간 키 공유 문제를 함께 해결했습니다.

---

## 스키마가 남긴 기록

Flyway 마이그레이션 9개 중 두 개가 **문제를 발견하고 뒤늦게 고친 흔적**입니다.

| 버전 | 내용 | 왜 나중에 추가됐나 |
|---|---|---|
| `V2__add_analysis_report_request_id.sql` | `request_id` UNIQUE | 애플리케이션 락만으로는 DB 저장 직후 응답 단절을 못 막는다는 걸 뒤늦게 인식 |
| `V3__add_data_status_to_report_results.sql` | `data_status` 컬럼 | 처음엔 위험도만 저장했고, "확인 불가"와 "안전"이 구분되지 않는 문제를 나중에 발견 |

처음부터 완벽하게 설계한 게 아니라, **돌려보고 문제를 발견해 스키마를 고쳤습니다.**

---

## 테스트

실패가 **사용자 피해나 외부 API 비용으로 직결되는 경로**를 우선 검증했습니다. (2026-08-17 측정, report 도메인 기준)

| 지표 | 보강 전 | 보강 후 |
|---|---:|---:|
| 테스트 수 | 93 | **252** (+159) |
| 실패 / 오류 | 0 / 0 | 0 / 0 |
| Instruction | 61.31% | **82.12%** |
| Line | 58.53% | **80.84%** |
| Branch | 49.35% | **66.31%** |

branch coverage를 함께 본 이유는, 위험 판정 로직이 조건 분기 덩어리라 line coverage만으로는 검증됐다고 말할 수 없기 때문입니다.

주요 검증 항목:

- `@ParameterizedTest`로 전세가율·선순위채권 **경계값의 바로 아래·같음·바로 위** — 위험도만이 아니라 `DataStatus`도 함께 단언해 "확인 불가를 안전으로 처리"하는 회귀를 차단
- `SET NX` 락 획득 실패 시 실행 중단, Lua 해제, 예외 여부와 무관한 `finally` 해제
- 완료된 `requestId` 재요청 시 유료 호출 없이 기존 리포트 반환
- `MockRestServiceServer`로 URL·쿼리·헤더·form body 계약 검증, 필수값 누락·오류 응답·통신 예외·깨진 JSON
- RSA 암호화는 임시 키 쌍으로 실제 복호화까지 확인하고, **잘못된 공개키가 민감정보를 평문으로 넘기지 않는 실패 경로**도 검증

> AI 특약 전용 테스트 4개는 팀원 담당 범위와 중복되어 제거했으며 위 수치에 포함되지 않습니다. 동일 제외 기준의 전체 백엔드 커버리지는 line 49.74%, branch 48.75%입니다.

---

## 기술 스택

| 구분 | 기술 | 사용 이유 |
|---|---|---|
| Runtime | Java 17 · Tomcat 9 · WAR | `javax` 기반 Spring MVC 구성과 배포 환경 |
| Web | Spring MVC 5.3 · Spring Security 5.8 | REST API, 필터 기반 JWT 처리 |
| Persistence | MyBatis · MySQL 8 · Flyway | 복잡한 조회/UPSERT/행 잠금 SQL 제어와 스키마 버전 관리 |
| Ephemeral state | Redis · Spring Data Redis | 인증 워크플로, TTL 상태, 요청 락, 캐시, 토큰 |
| Async / streaming | `@Async` · WebSocket · gRPC | 녹음 수신과 STT/AI 처리 분리 |
| Storage | Ncloud Object Storage | 업로드 녹음 원본 |
| Frontend | Vue 3 · Pinia · Axios | 모바일 퍼스트 반응형 |
| API 문서 | Swagger (springfox) | 프론트와의 계약 공유 |
| CI/CD | GitHub Actions · Docker Compose | backend/frontend 각각 CI와 배포 워크플로 |
| Test | JUnit 5 · Mockito · AssertJ · JaCoCo | 판정·오케스트레이션 분기와 커버리지 측정 |

**외부 데이터** — CODEF(등기·건축물대장), 국토교통부 실거래가, 한국부동산원, 건축HUB, 카카오 주소, CLOVA STT, OpenAI

### 인증·공통

- BCrypt 비밀번호 해시, HMAC 서명 JWT (Access / Refresh 타입 분리), stateless session
- 로그아웃 시 Access Token을 **남은 만료 시간만큼 Redis blacklist**에 등록
- `anyRequest().authenticated()` 기본값에 공개 엔드포인트만 화이트리스트
- `BusinessException` + `ErrorCode` 공통 예외 모델, `status / code / message / path / timestamp` 통일 응답

---

## 프로젝트 구조

```text
second-zip-app/
├── backend/     Spring MVC WAR 애플리케이션
├── frontend/    Vue 3 애플리케이션
├── docker/      로컬 개발 환경 (MySQL · Redis healthcheck 후 백엔드 기동)
├── docs/        ERD · 시퀀스 다이어그램 · 설계 문서
└── .github/     CI · 배포 워크플로
```

### 백엔드 패키지

| 패키지 | 역할 |
|---|---|
| **`report`** | **분석 워크플로 · 외부 API · 위험 판정 · 리포트** ← 담당 |
| `checklist` | 계약 체크리스트 |
| `record` | 녹취 (WebSocket · gRPC · STT · AI 분석) |
| `map` | 전세가격 지수 · 사기 피해 지도 |
| `account` / `terms` | 계정 · 약관 |
| `security` | JWT 발급·검증·blacklist |
| `common` | 공통 예외 · 응답 모델 |

---

## 알려진 한계

잘된 부분만 적으면 검증 질문에서 무너집니다. 실제 상태는 이렇습니다.

**보안**

- WebSocket이 `setAllowedOrigins("*")`이고, URL의 세션 ID 소유권 검증이 없습니다
- 다른 기기의 기존 Access Token을 일괄 무효화하는 기능이 없습니다
- 전세가격 수동 동기화 API에 별도 관리자 인가가 없습니다

**동시성·정합성**

- 등기 캐시 TTL 기본값이 300초로 워크플로 TTL(15분)보다 짧아, 유료 조회 후 DB 저장 전 실패하고 5분이 지나면 재과금될 수 있습니다. 워크플로보다 긴 값으로 정렬해야 합니다
- `RegistryClient`의 유료 호출이 `synchronized`라 단일 JVM에서는 직렬화되지만, 처리량 병목이며 다중 인스턴스에는 적용되지 않습니다
- 실시간 녹취의 Context와 gRPC 세션이 JVM 메모리라 scale-out 시 sticky session이나 외부 상태 저장소가 필요합니다
- 일부 서비스가 트랜잭션 안에서 외부 API를 호출해 DB 자원을 오래 점유합니다

**관측성**

- Actuator, metrics, tracing, request correlation ID, 구조화 로그가 없습니다
- 외부 API 429/5xx backoff와 circuit breaker가 없습니다

**성과 지표**

유료 호출 감소율·캐시 적중률·P95 처리시간은 **측정하지 않았습니다.** 코드 구조상 기대되는 효과는 있으나 수치로 검증한 것이 아니므로 적지 않았습니다.

---

<div align="center">
<sub>KB IT's Your Life 7기 · 24반 3팀</sub>
</div>
