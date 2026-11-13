# 웹서비스설계 9주차 실습 2 · 검색·정렬·페이지네이션 게시판 개선

202219423 이동현

## 실행 방법

```bash
npm install
node server.js
# http://localhost:3000 접속
```

## 수정 내용

### server.js
- **정렬 키 화이트리스트**: `Object.hasOwn`으로 화이트리스트에 직접 정의된 키만 허용하고, 나머지는 `latest`로 처리 (`__proto__`, `constructor` 등 상속 속성 차단, 배열 파라미터 차단)
- **검색 파라미터 방어**: `?search=a&search=b`처럼 문자열이 아닌 값이 들어오면 빈 검색어로 처리 (기존에는 500 에러)
- **커서 API 검색 추가**: 검색 조건 → `id < cursor` → `id DESC` 정렬 → `limit + 1` 조회 후 `hasNext` 판단
- **응답 meta 확장**: 실제로 적용된 `sort`, `search`를 응답에 포함

### index.html
- **커서 모드**: 검색어 전송, 정렬 선택 비활성화(id 내림차순 고정 안내), 더 보기 중복 요청 방지, id 기준 중복 제거
- **URL 상태**: URL의 page·sort 값 검사, 서버 응답 meta로 상태·URL 보정, 타이핑 중에는 `replaceState`, 커서 모드에서 뒤로 가기 시 오프셋 모드로 복귀
- **결과 없음**: 페이지 버튼 숨김
- **비동기**: 요청 번호(`latestRequestId`)로 오래된 응답 무시(AbortController와 병행), 조회 버튼·Enter 입력 시 남은 디바운스 타이머 정리
- **화면 상태**: 로딩·결과 상태 줄(`role="status"`), 오류 안내와 다시 시도 버튼(`role="alert"`)

검증 결과와 개념 설명(오프셋·커서, 디바운스·요청 취소, limit + 1, offset·totalPages 계산식)은 제출한 실습 보고서에 정리했습니다.

---

# 09주차 실습 패키지 · 대용량 검색·정렬·페이징 심화 (3학년 전공)

본 실습 패키지는 웹서비스설계 9주차 (3시간 마스터) 강의에서 다루는 **1,000건 대용량 데이터셋 기반 검색, 정렬 보안, 오프셋 vs 커서 페이지네이션, 프론트엔드 비동기 최적화**의 실제 구동 코드입니다.

---

## 🚀 빠른 실행 방법

```bash
# 1. 의존성 설치
npm install express

# 2. 백엔드 서버 구동
node server.js

# 3. 브라우저 접속
# http://localhost:3000
```

---

## 📂 파일 구성 및 엔지니어링 특징

### 1. `server.js` (Express 백엔드 API)
- **대량 데이터셋**: 1,000건의 인메모리 게시글 데이터 자동 시드.
- **방어적 파라미터 파싱**: `parseInt(..., 10)` 기수 명시, 음수 및 0 페이지 차단, `Math.min(..., 100)` 대량 조회 공격 방어.
- **ORDER BY 화이트리스트 보안**: `SORT_WHITELIST` 객체 매핑으로 SQL Injection 원천 봉쇄 및 `id DESC` 동점자 방지 2차 정렬 결합.
- **2대 페이징 API 제공**:
  - `GET /api/posts`: 오프셋 기반 페이징 (`LIMIT` / `OFFSET`, 표준 `meta` Envelope 응답)
  - `GET /api/posts/cursor`: 고성능 커서 기반 페이징 (`WHERE id < ? ORDER BY id DESC LIMIT 10`, `limit + 1` 패턴으로 `COUNT(*)` 제거)

### 2. `index.html` (SPA 프론트엔드 & 실시간 DevTools 검사기)
- **탭 모드 전환**: [오프셋 번호 페이징] ↔ [커서 기반 무한 스크롤] 즉시 비교.
- **실시간 디바운스 (300ms)**: 연속 타이핑 시 불필요한 API 요청 폭격 방지.
- **AbortController 비동기 방어**: 새 검색어 입력 시 이전 진행 중인 fetch 요청을 즉각 취소하여 Race Condition 버그 예방.
- **브라우저 히스토리 동기화**: `URLSearchParams`와 `history.pushState` 및 `window.onpopstate` 이벤트로 뒤로 가기 / 새로고침 상태 복원 지원.

---

## 🎯 3학년 전공 실습 과제 체크리스트

1. **정렬 보안**: `server.js`에서 허용되지 않은 정렬 키(예: `sort=views; DROP TABLE--`) 입력 시 안전하게 기본값(`latest`)으로 격리되는지 검증하세요.
2. **커서 페이징**: 100페이지 이상 스킵 시 오프셋 방식의 한계와 커서 방식의 속도 차이를 DevTools의 `X-Response-Time` 헤더로 측정해 보세요.
3. **Race Condition 실험**: `index.html`에서 빠른 타이핑 시 콘솔에 `[AbortController] 이전 비동기 요청이 정상 취소되었습니다` 로그가 정상 출력되는지 확인하세요.
