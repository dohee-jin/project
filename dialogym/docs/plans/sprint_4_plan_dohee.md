# Sprint 3 결과 요약 및 Sprint 4 계획 제안서 (진도희)
**작성자:** 진도희 (B팀, Fullstack)  
**작성일:** 2025.10.19  
**Sprint 기간:** 10.13 ~ 10.19  

---

## Sprint 3 진행 결과 요약

### 완료된 항목
| 항목 | 상태 | 주요 내용 |
|------|------|-----------|
| **WebRTC 클라이언트 (React)** | 완료 | `getUserMedia()`, `RTCPeerConnection`, `offer/answer`, `ICE candidate` 교환 성공. <br>→ Signaling 서버와 통신 정상 작동 |
| **GPT-4o Realtime 세션 생성** | 완료 | OpenAI Realtime API 세션 토큰 발급 및 관리 로직 구현. <br>백엔드 세션 ID(`ai_realtime_session_id`) 저장 구조 확립 |
| **AI 프롬프트 (더미)** | 완료 | “동료 협업”, “교사-학부모” 프롬프트 더미 데이터 추가 |
| **E2E 초기 통합 테스트** | 부분 성공 | 클라이언트 ↔ Signaling ↔ Backend까지 정상 작동. <br>단, Backend ↔ GPT-4o 연결 미완성으로 대화 미완 |

---

## 미완료 및 이슈 요약

### 주요 미해결 과제: **WebRTC ↔ GPT-4o 간 실시간 오디오 송수신**
- 클라이언트와 Signaling 서버 간 offer/answer 교환은 성공했으나,  
  **Backend ↔ GPT-4o** 간 WebRTC 세션 연결 시 **동시성(Concurrency) 문제** 발생.
- GPT 세션이 `audioHandler`와 `gptSessionManager`에 의해 **이중 접근**되며,  
  오디오 스트림 전송 타이밍이 꼬여 handshake 불안정.

**원인 분석**
1. `SessionCoordinator`가 사용자 오디오 업로드 및 GPT 응답 스트림을 동시에 제어함 → race condition 발생  
2. 백엔드 내부에서 RTCPeerConnection 객체를 공유함 → 세션 상태 충돌  
3. GPT-4o API가 handshake 완료 전 PCM chunk를 수신함 → “no active transport” 오류

---

## Sprint 4 제안 과제 (개선 방향)

### 1. **세션 구조 개선 및 동기화 제어**
- `SessionCoordinator` 단에 **Lock/Queue 기반 세션 직렬화 로직** 추가  
- `backend ↔ GPT-4o` 전용 **RTCPeerConnection 분리** (user–backend와 완전 분리)
- 오디오 송신/수신을 event-driven 비동기 처리로 재구성

### 2. *WebRTC ↔ GPT 오디오 파이프라인 안정화**
- PCM chunk 단위 전송 재시도 로직 및 상태 핸들링 개선  
- 음성 입력→응답 간 round-trip latency 측정 및 로깅 (목표: <200ms)

### 3. **프론트엔드 UI 고도화**
- 기존 WebRTC 대화 화면 확장:
  - 탭으로 말하기(Tap-to-Talk)
  - 실시간 텍스트 표시(Streaming transcript)
  - AI 응답 시 음성/텍스트 동시 출력
- 실시간 상태 표시(“연결 중”, “응답 중”, “대화 종료됨”) 추가

### 4️. **테스트 및 검증**
- **E2E 1회 완전 성공** (사용자 발화 ↔ GPT 응답) 목표  
- **Cross-browser test (Chrome/Safari)**  
- 버그 로깅 자동화 (예: latency, 연결 해제, 오류 카운트)

---

## Sprint 4 제안 일정 (10.21 ~ 10.27)

| 날짜 | 주요 작업 | 비고 |
|------|-------------|------|
| **10.21 (화)** | 백엔드 `SessionCoordinator` 개선, Lock 기반 구조 설계 | concurrency 해결 시작 |
| **10.22 (수)** | Backend ↔ GPT-4o 전용 PeerConnection 분리 | 실시간 세션 안정화 |
| **10.23 (목)** | 오디오 송수신 파이프라인 개선 및 latency 로그 수집 | PCM chunk 전송 검증 |
| **10.24 (금)** | 프론트엔드 UI 고도화 (탭 발화, 실시간 텍스트 표시) | UX 개선 |
| **10.25 (토)** | E2E 통합 테스트 (대화 1회 성공 목표) |  |
| **10.26 (일)** | 버그 수정 / 프로덕션 테스트 |  |
| **10.27 (월)** | Sprint Review & Retrospective |  |

---

## Sprint 4 목표 (DoD)

- Backend ↔ GPT-4o WebRTC 세션 안정화  
- 오디오 송수신 round-trip latency < 200 ms  
- 대화 1회 이상 성공 (실시간 음성 + 텍스트 표시)  
- UI 개선 (탭 발화, 텍스트 실시간 표시 기능)  
- 브라우저 2종 이상 테스트 통과  

---

## 전달 요약

> WebRTC 클라이언트 및 GPT 세션 생성까지 성공했으나,  
> 백엔드와 GPT 간 동시성 문제로 실시간 음성 대화 불안정.  
> Sprint 4에서는 **세션 구조 분리 및 오디오 파이프라인 안정화**를 핵심 목표로 함.  
> 프론트엔드 고도화(탭 발화, 실시간 텍스트) 병행 추진 예정.

