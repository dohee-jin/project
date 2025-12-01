# Sprint 3 Planning (수정본 v1.1) – 백엔드 완료 + React 시작

**기간**: 2025.10.13 (월) ~ 2025.10.20 (월) - 7일  
**SM**: 진도희 (B팀)  
**목표**: 백엔드 3대 기능(세션 관리, AI/오디오 처리, WebRTC) 완성 + 프론트엔드 초기 구축

---

## 주요 변경 사항

| 항목 | 변경 전 | 변경 후 | 비고 |
|------|----------|----------|------|
| **WebRTC Signaling 서버** | 김경민 | 김경민 | 서버(WebSocket) 유지 |
| **WebRTC 클라이언트 (React)** | 김경민 | **진도희** | 프론트엔드 WebRTC 전담 |
| **GPT-4o Realtime API** | 진도희 | 진도희 | 동일 |
| **오디오 변환 파이프라인** | 진도희 | 진도희 | 동일 |

**변경 이유**  
GPT-4o Realtime 음성 송수신과 오디오 변환 파이프라인을 통합 관리하기 위함.  
음성 연결(WebRTC)과 AI 처리를 한 흐름으로 제어해 지연을 최소화하고, 연동 디버깅 효율을 높이기 위함.

---

## Sprint 3 목표

1. **세션 관리 API** (왕택준): `POST /api/dialogues/start`
2. **WebRTC Signaling 서버** (김경민): `WebSocket /ws/signaling/{sessionId}`, Janus 연동
3. **WebRTC 클라이언트** (진도희): 브라우저 음성 입출력, SDP/ICE 처리
4. **GPT-4o Realtime API 연동** (진도희): 실시간 음성 송수신
5. **오디오 변환 파이프라인** (진도희): Opus ↔ PCM 16kHz 변환
6. **실시간 분석** (김경민): 발화 속도, 추임새 감지
7. **피드백 생성** (왕택준): GPT-4 점수화, 개선안 생성
8. **프론트엔드 React UI 구축** (공동): 로그인, 회원가입, 시나리오 선택, 대화 화면

---

## 팀별 역할 (수정 반영)

### A팀: 김경민 (Backend Developer)

**담당 기능 (총 8일 예상)**  

#### 1. WebSocket Signaling 서버 (3.5일)
- 메시지 타입: `offer`, `answer`, `ice-candidate`, `error`
- SDP Offer/Answer 중계
- ICE Candidates 전달
- Janus REST API (Room 생성/조인)
- 에러 및 연결 종료 처리

**WebSocket 메시지 프로토콜**
```json
// 클라이언트 → 서버
{ "type": "offer", "data": { "sdp": "..." } }
{ "type": "ice-candidate", "data": { "candidate": "...", "sdpMLineIndex": 0 } }

// 서버 → 클라이언트
{ "type": "answer", "data": { "sdp": "..." } }
{ "type": "ice-candidate", "data": { "candidate": "...", "sdpMLineIndex": 0 } }
```

#### 2. 실시간 분석 (2.5일)
- 발화 속도 계산 (WPM)
- 추임새 감지 ("음", "어", "네" 등)
- 실시간 JSON 포맷 전송 `/ws/feedback/{sessionId}`

**실시간 피드백 포맷**
```json
{ "type": "speed", "value": 150, "unit": "wpm" }
{ "type": "filler", "word": "음", "count": 3 }
```

#### 3. AI 프롬프트 작성 (2일)
- "상사 보고"
- "면접 연습"
- "연인 갈등"
- "부모님 연락"

**각 프롬프트 포함:**
- 역할 정의 (상사/면접관/연인/부모)
- 톤 설정 및 예상 응답 예시
- 꼬리질문/심화 질문 시나리오
- 한국식 언어 문화 반영
- Voice 선택 (echo/alloy/fable/nova)

---

### B팀: 진도희 (Fullstack Developer, SM)

**담당 기능 (총 11일 예상)**  

#### 1. GPT-4o Realtime API 연동 (3.5일)

**OpenAI Realtime API 세션 생성**
- `POST https://api.openai.com/v1/realtime/sessions`
- 응답: `{ token, session_id, expires_at }`

**WebSocket 연결** (서버 → OpenAI)
- `wss://api.realtime.openai.com/v1/realtime?token={token}`
- 프롬프트 전송 (시나리오 프롬프트 + Voice 파라미터)
- 오디오 청크 송수신

**메시지 처리**
- Input Audio: Base64 인코딩된 PCM 데이터 (100ms 청크)
- Output Audio: 이벤트 스트림으로 수신
- 텍스트 트랜스크립션 (선택사항)

**세션 관리**
- DialogueSession에 `ai_realtime_session_id` 저장
- 토큰 갱신 로직 (필요 시)
- 세션 종료 및 정리

#### 2. 오디오 변환 파이프라인 (2.5일)

**Opus → PCM 16kHz 변환**
- 라이브러리: `libopus.js` 또는 `ffmpeg.wasm` 또는 Web Audio API
- 16-bit PCM, 16kHz 샘플링레이트, 모노 채널
- 손실 최소화

**리샘플링 및 청킹**
- 100ms 단위 청크 분할 (1600 샘플)
- Base64 인코딩/디코딩
- 큐 관리 (입력/출력 오디오 동기화)

**오디오 렌더링**
- 응답 오디오 → PCM → 재생 (Web Audio API)
- 실시간 처리 (지연 최소화 <200ms)

**성능 최적화**
- CPU 사용량 모니터링
- 메모리 누수 방지 (버퍼 관리)

#### 3. WebRTC 클라이언트 (2일, **신규 이관**)

**React 컴포넌트 구현**
- `getUserMedia()`로 마이크 캡처  
- `RTCPeerConnection` 생성 및 관리  
- SDP Offer 생성 후 Signaling 서버로 전송  
- `ontrack` 이벤트로 AI 음성 수신 및 재생  
- ICE Candidates 처리
- 마이크 On/Off, 종료 처리  
- GPT-4o Realtime 세션과 연동  

**오디오 재생**
- 원격 오디오 스트림 재생
- 마이크 제어 (음소거/해제)

**담당 UI**: 마이크 버튼, 오디오 재생 제어

#### 4. AI 프롬프트 작성 (2일)
- "동료 협업"
- "교사-학부모"

#### 5. SM 역할 (전주간)
- 일일 스탠드업 (10시)
- 중간 점검 (18시)
- 리스크 관리 및 일정 조정

---

### C팀: 왕택준 (PO / Tech Lead)

**담당 기능**

#### 백엔드 (5일)

**1. 세션 관리 API (2일)**

**POST /api/dialogues/start**
- Request: `{ scenarioId, userId }`
- Response: `{ sessionId (UUID), scenario, profile, startedAt }`
- DialogueSession 엔티티에 저장
- Janus room 자동 생성 (`janus_room_id` 저장)

**에러 처리**
- 유효하지 않은 scenarioId/userId
- 타임아웃 설정

**2. 피드백 생성 API (2.5일)**

**POST /api/feedback/save**
- Request:
  ```json
  {
    "sessionId": "abc123",
    "audioUrl": "https://s3.../recording.mp3",
    "transcript": "안녕하세요. 보고드릴 내용이...",
    "duration": 180,
    "wpm": 145,
    "fillerCount": 3
  }
  ```

**GPT-4 점수화**
- 공손도 (politeness: 1~10)
- 명료성 (clarity: 1~10)
- 종합 점수 계산 (가중치 적용)
  ```
  종합 = wpm (30%) + filler (20%) + politeness (25%) + clarity (25%)
  ```

**개선안 생성** (3가지 버전)
- 간결하게 (concise)
- 정중하게 (polite)
- 따뜻하게 (warm)

**Feedback 엔티티 저장**
- User/Feedback/History 관계 처리
- audioUrl, transcript, scores (JSON), improvements (JSON) 저장

**3. 기타 (0.5일)**
- API 문서 (Swagger) 업데이트
- 에러 핸들링 및 로깅

#### 프론트엔드 (3.5일)

**1. React 프로젝트 기초 (1일)**
- Vite + React 초기화
- TailwindCSS 또는 module.scss 설정
- 디렉토리 구조 설계 (pages, components, hooks, utils)
- react-router-dom 라우팅 설정
- Context API 상태 관리 구조

**2. 기본 페이지 스켈레톤 (2.5일)**

**로그인/회원가입 페이지 (기본 구조)**
- 폼 HTML 구조만 (스타일링 최소화)
- 기본 입력 필드 (이메일, 비밀번호, 이름)
- 폼 검증 로직 기본
- API 연동 (POST /api/auth/login, /signup) 
- JWT 토큰 저장 로직
- 라우팅 (로그인 → 시나리오 선택)

**시나리오 선택 페이지 (기본 구조)**
- 시나리오 리스트 UI
- GET /api/scenarios API 연동
- 선택 기능
- 대화 화면으로 네비게이션

**대화 화면 페이지 (기본 구조)**
- 기본 레이아웃만 (마이크 버튼, 타이머, 종료 버튼)
- Signaling 연결 준비 (B팀의 WebRTC 클라이언트 통합)
- 실시간 피드백 수신 영역

**공통 컴포넌트 (기본)**
- Form 컴포넌트 (기본 검증)
- LoadingSpinner
- ErrorBoundary

---

## 기술 흐름 (수정 후)

```
[User Mic]
   ↓ getUserMedia() (진도희)
   ↓ RTCPeerConnection (진도희)
   ↔ WebSocket /ws/signaling (김경민)
   ↔ Janus Media Server (김경민)
   ↔ GPT-4o Realtime API (진도희)
   ↔ AudioConverter (진도희)
   ↓
[User Speaker]
```

---

## 변경 효과

| 항목 | 효과 |
|------|------|
| **음성 흐름 통합** | WebRTC ↔ GPT-4o ↔ 오디오 변환이 한 흐름에서 관리됨 |
| **지연 최소화** | 오디오 처리–송수신 병목 감소 (<200ms 목표) |
| **의존성 감소** | 클라이언트-서버 간 디버깅 구간 명확화 |
| **업무 집중도 향상** | 김경민: 서버 집중 / 진도희: 음성 파이프라인 집중 |

---

## Sprint 3 주간 일정 (10.13 ~ 10.20)

### 10.13 (월) - 백엔드 API 및 프론트엔드 초기화

**왕택준 (C팀)**
- POST /api/dialogues/start 구현 및 테스트
- Vite + React 프로젝트 초기화
- TailwindCSS 설정
- 로그인/회원가입 페이지 HTML 구조

**김경민 (A팀)**
- WebSocket /ws/signaling/{sessionId} 구현 시작
- Janus REST API 호출 로직 프로토타입
- AI 프롬프트 4개 초안 작성
- 발화 속도 계산 로직 프로토타입

**진도희 (B팀, SM)**
- OpenAI Realtime API 키 발급 및 계정 설정
- GPT-4o Realtime 문서 상세 숙지
- 오디오 변환 라이브러리 선택 및 테스트
- WebRTC 클라이언트 구현 준비
- 일일 스탠드업 준비

---

### 10.14 (화) - 핵심 로직 구현

**왕택준 (C팀)**
- 회원가입 화면 UI 작성
- POST /api/feedback/save 스켈레톤 설계
- GPT-4 점수화 프롬프트 작성

**김경민 (A팀)**
- WebSocket Signaling 본격 구현
  - SDP Offer/Answer 처리
  - ICE Candidates 중계
- Janus REST API 연동
- AI 프롬프트 4개 정교화
- 추임새 감지 알고리즘 프로토타입

**진도희 (B팀, SM)**
- GPT-4o Realtime 세션 생성 로직 구현
- OpenAI WebSocket 클라이언트 연결 테스트
- 오디오 Base64 인코딩/디코딩 구현
- WebRTC 클라이언트 기본 구조 작성

---

### 10.15 (수) - 프론트엔드 UI & WebRTC 통합

**왕택준 (C팀)**
- 시나리오 선택 화면 UI 작성
- 대화 화면 UI 기본 틀 작성
- 폼 검증 로직

**김경민 (A팀)**
- Signaling 서버 완성
- 발화 속도/추임새 WebSocket 메시지 구현
- 실시간 피드백 JSON 포맷 최종화
- 프롬프트 4개 완성

**진도희 (B팀, SM)**
- **WebRTC 클라이언트 구현 완성**
  - getUserMedia()
  - RTCPeerConnection 생성
  - Signaling 연동
  - ontrack 이벤트 처리
- 오디오 변환 파이프라인 완성
- GPT-4o와 WebRTC 통합 테스트
- AI 프롬프트 2개 작성

---

### 10.16 (목) - 전체 통합 테스트

**왕택준 (C팀)**
- 모든 프론트엔드 페이지 기본 구조 완성
- 기본 API 연동 테스트
- 라우팅 전체 흐름 테스트

**김경민 (A팀)**
- Signaling 서버 전체 테스트
- 실시간 분석 테스트
- 에러 처리 및 재연결 로직

**진도희 (B팀, SM)**
- **GPT-4o + WebRTC 통합 테스트** (1회 이상 성공)
- 오디오 레이턴시 측정 (<200ms 목표)
- 브라우저 호환성 테스트

**공통**
- E2E 통합 테스트
- 버그 리스트 작성

---

### 10.17 (금) - 버그 수정 & 배포 준비

**전체 팀**
- 버그 수정
- 최종 테스트
- 배포 준비

---

### 10.18 (토) - 프로덕션 배포

**전체 팀**
- 프로덕션 배포 (https://dialogym.com)
- 프로덕션 테스트

---

### 10.19 (일) - Sprint Review & Retrospective

**오후 3시: Sprint Review (45분)**

데모 순서:
1. **C팀**: 회원가입 → 로그인 → 시나리오 선택 화면
2. **A팀**: WebRTC Signaling 연결 + 실시간 분석
3. **B팀**: WebRTC 클라이언트 + GPT-4o 음성 송수신
4. **통합**: 전체 플로우 데모

**저녁 5시: Sprint Retrospective (30분)**

---

## Sprint 3 완료 기준 (DoD)

### 필수 완료

**백엔드**
- ✅ POST /api/dialogues/start 구현 완료
- ✅ WebSocket /ws/signaling/{sessionId} 구현 완료
- ✅ WebSocket /ws/feedback/{sessionId} 구현 완료
- ✅ Janus REST API 연동 완료
- ✅ GPT-4o Realtime API 연동 완료
- ✅ 오디오 변환 파이프라인 완성
- ✅ POST /api/feedback/save 구현 완료
- ✅ 6개 시나리오 프롬프트 작성 완료

**프론트엔드**
- ✅ 로그인/회원가입 페이지 (기본 구조)
- ✅ 시나리오 선택 페이지 (리스트 UI)
- ✅ 대화 화면 페이지 (WebRTC 클라이언트 포함)
- ✅ React 라우팅 전체 흐름
- ✅ 프로덕션 배포 완료

**테스트**
- ✅ E2E 테스트 성공
- ✅ GPT-4o + WebRTC 음성 송수신 1회 이상 성공
- ✅ 오디오 레이턴시 <200ms 달성

---

## 리스크 및 완화 방안

| 리스크 | 영향도 | 가능성 | 완화 방안 |
|--------|--------|--------|----------|
| **WebRTC 클라이언트 복잡도** | 높음 | 중간 | B팀(진도희)에게 충분한 시간, A팀과 긴밀한 소통 |
| **오디오 레이턴시 >200ms** | 높음 | 중간 | 오디오 변환 최적화, CPU 모니터링 |
| **GPT-4o API 비용 증가** | 중간 | 중간 | 일일 한도 설정, 사용량 모니터링 |
| **팀 간 의존성 블로커** | 높음 | 중간 | SM(진도희)의 적극적 조율 |

---

## 주요 기술 스택

| 항목 | 기술 | 담당 |
|------|------|------|
| **WebRTC 시그널링** | 커스텀 WebSocket | 김경민 |
| **WebRTC 클라이언트** | React + RTCPeerConnection | **진도희** |
| **미디어 서버** | Janus | 김경민 |
| **AI 음성** | GPT-4o Realtime API | 진도희 |
| **오디오 변환** | libopus.js / Web Audio API | 진도희 |
| **점수화** | GPT-4 API | 왕택준 |
| **프론트엔드** | React + Vite | 왕택준 + 진도희 |

---

## 중간 점검 일정

| 날짜 | 시간 | 주제 | 진행자 |
|------|------|------|--------|
| 10.13 (월) | 14:00 | Sprint 3 킥오프 + API 명세 확인 | 진도희(SM) |
| 10.14 (화) | 18:00 | 핵심 로직 진행 상황 | 진도희(SM) |
| 10.15 (수) | 18:00 | 프론트엔드 UI & WebRTC 클라이언트 | 진도희(SM) |
| 10.16 (목) | 18:00 | 전체 통합 테스트 상황 | 진도희(SM) |
| 10.17 (금) | 18:00 | 배포 준비 & 버그 수정 현황 | 진도희(SM) |
| 10.19 (일) | 15:00 | Sprint Review (데모) | 진도희(SM) |
| 10.19 (일) | 17:00 | Sprint Retrospective | 진도희(SM) |

---

## Success Metrics

### 기술 지표
- ✅ Signaling 연결 성공률 > 95%
- ✅ 오디오 레이턴시 평균 < 150ms
- ✅ API 응답 시간 평균 < 300ms
- ✅ 브라우저 호환성 3가지 이상 (Chrome, Safari, Firefox)

### 기능 지표
- ✅ E2E 플로우 성공률 100% (로그인 → 대화 → 피드백)
- ✅ GPT-4o 음성 송수신 성공률 > 90%
- ✅ 피드백 점수 신뢰도 > 80%

### 일정 지표
- ✅ Sprint 일정 준수율 > 95%
- ✅ 버그 해결 시간 < 4시간
- ✅ 배포 준비 완료도 100%

---

**최종 결론:**  
진도희가 WebRTC 클라이언트를 직접 담당하며 GPT-4o 음성 처리와 통합함.  
김경민은 서버 측 Signaling과 실시간 분석에 집중함.  
전체 시스템 음성 플로우의 품질과 안정성이 향상될 것으로 예상됨.

