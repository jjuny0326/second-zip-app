# README 이미지

루트 README에서 사용하는 화면 캡처 8장입니다. 2026-09-28 실제 Vue 컴포넌트를 로컬 촬영 환경에서 렌더링했습니다. 개발 도구 표시 없이 서비스 화면만 촬영했습니다.

| 파일 | 화면 경로 | 데이터 |
| --- | --- | --- |
| `main-map.jpg` | `/` | 촬영용 시도별 가상 피해주택 건수. 실제 통계가 아님 |
| `report-create.jpg` | `/report/create` | 주소 선택 전 입력 폼, 예시 동·호수와 보증금 |
| `report-preview.jpg` | `/report/analysis/preview/a` | `frontend/src/constants/report/mock.js`에 정의된 샘플 리포트 |
| `report-details.jpg` | `/report/analysis/preview/a` | 같은 샘플의 첫 번째 사기 유형을 펼친 상태 |
| `ai-special-terms.jpg` | `/report/analysis/preview/a` | 같은 샘플에 저장된 특약 예시. 촬영 시 AI 호출 없음 |
| `checklist.jpg` | `/checklist/1` | 촬영용 항목 8개 중 3개를 완료한 상태, 녹음 시작 전 |
| `dictionary.jpg` | `/dictionary` | 전세사기 도감의 기본 안내 화면 |
| `document-guide.jpg` | `/dictionary/register` | 기존 서비스 자산을 사용하는 등기부등본 웹툰 안내 |

공개 화면을 재촬영하려면 `frontend/`에서 `npm run dev`를 실행하고 위 경로를 엽니다. 리포트 예시 경로는 로그인이나 실제 유료 문서 발급 없이 열 수 있습니다.

지도·체크리스트 촬영은 별도 로컬 진입점에서 실제 화면 컴포넌트에 샘플 API 응답을 연결했습니다. 인증이 필요한 운영 화면에 접속하거나 실제 녹음·분석 요청을 실행하지 않았습니다. 체크리스트 항목, 진행 상태, 지도 수치는 UI 설명용으로 구성했으며 기본 서비스 경로에서 동일한 데이터가 자동으로 제공되지는 않습니다.

상단 로고는 기존 서비스 자산인 `frontend/src/assets/images/main-logo.png`를 참조합니다. 화면 캡처와 로고는 저장소 상대 경로를 사용하므로 README와 함께 관리됩니다.
