# Dialogym Sprint 3 MVP (수정본 v1.1)

**원칙**: 한 명이 최소 3일 안에 끝낼 수 있고, "작동하는" 수준만 목표. UI 디자인은 전혀 고려 X

---

## 주요 변경 사항

| 항목 | 변경 전 | 변경 후 |
|------|----------|----------|
| **WebRTC 클라이언트** | 김경민 (A팀) | **진도희 (B팀)** |

**이유**: GPT-4o Realtime과 WebRTC 클라이언트를 한 흐름으로 통합 관리

---

## 백엔드 MVP (5일)

### 1. 세션 생성 API (1일) - 왕택준 (C팀)
```
POST /api/dialogues/start
Request: { scenarioId: 1, userId: 1 }
Response: { sessionId: "uuid", janusRoomId: 12345 }
```
- Janus와 OpenAI를 미리 시작해두고 UUID만 반환
- 실제 방 생성은 프론트에서 WebRTC 연결할 때

### 2. WebRTC Signaling 최소 서버 (2일) - 김경민 (A팀)
```
WebSocket ws://localhost:8080/ws/signaling/{sessionId}

클라이언트 → 서버:
{ "type": "offer", "data": { "sdp": "..." } }
{ "type": "ice", "candidate": "..." }

서버 → 클라이언트:
{ "type": "answer", "data": { "sdp": "..." } }
{ "type": "ice", "candidate": "..." }
```
- 커스텀 WebSocket Handler 100줄 정도면 충분
- 메모리에만 저장 (프로덕션은 나중)
- 에러 처리는 기본만

### 3. 피드백 저장 API (1일) - 왕택준 (C팀)
```
POST /api/feedback/save
Request: { 
  sessionId, 
  transcript: "사용자가 말한 텍스트",
  duration: 120,  // 초
  wpm: 150,
  fillerCount: 3
}
Response: { feedbackId, scores: { score: 7.5 } }
```
- **점수는 간단한 수식**: `(150 - wpm/10) * 0.5 + (10 - fillerCount/2) * 0.5` → 정규화
- GPT-4 호출 X (시간 낭비)
- DB에 그냥 저장만

### 4. 실시간 분석 (1일) - 김경민 (A팀)
- 발화 속도 계산 (WPM)
- 추임새 감지 ("음", "어", "네" 등)
- WebSocket `/ws/feedback/{sessionId}` 전송

**실시간 피드백 포맷**
```json
{ "type": "speed", "value": 150, "unit": "wpm" }
{ "type": "filler", "word": "음", "count": 3 }
```

---

## 프론트엔드 MVP (4일)

### 1. 기본 페이지 구조 (1일) - 왕택준 (C팀)
```
/login → /scenarios → /dialogue/{sessionId} → /feedback
```
- HTML 폼 + 기본 라우팅만
- CSS 프레임워크 쓰지 말고 기본 CSS만

### 2. 로그인/회원가입 (0.5일) - 왕택준 (C팀)
```jsx
// 예시만
const handleLogin = async (email, password) => {
  const res = await fetch('/api/auth/login', {
    method: 'POST',
    body: JSON.stringify({ email, password })
  });
  localStorage.setItem('token', res.token);
  navigate('/scenarios');
};
```
- 검증은 `email.includes('@')` 정도만
- 에러 표시는 `alert()` 써도 됨

### 3. 시나리오 선택 (0.5일) - 왕택준 (C팀)
```jsx
const [scenarios, setScenarios] = useState([]);

useEffect(() => {
  fetch('/api/scenarios')
    .then(r => r.json())
    .then(data => setScenarios(data));
}, []);

return (
  
    {scenarios.map(s => (
      <button key={s.id} onClick={() => startDialogue(s.id)}>
        {s.name}
      
    ))}
  
);
```
- 리스트만 (카드 디자인 X)
- 클릭하면 `/dialogue/{sessionId}`로 이동

### 4. 대화 화면 (2일) - **진도희 (B팀)** ⭐
```jsx
const [isRecording, setIsRecording] = useState(false);
const [transcript, setTranscript] = useState('');
const [feedback, setFeedback] = useState(null);

const startRecording = async () => {
  const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
  const pc = new RTCPeerConnection();
  pc.addTrack(stream.getAudioTracks()[0]);
  
  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);
  
  // Signaling 서버로 offer 전송
  ws.send(JSON.stringify({
    type: 'offer',
    data: { sdp: offer.sdp }
  }));
  
  setIsRecording(true);
};

const stopRecording = async () => {
  // 녹음 중지, 텍스트만 전송 (분석은 서버에서)
  const response = await fetch('/api/feedback/save', {
    method: 'POST',
    body: JSON.stringify({
      sessionId,
      transcript,  // Web Speech API로 생성된 텍스트
      duration: recordingDuration
    })
  });
  
  const feedback = await response.json();
  // /feedback 페이지로 이동
  navigate(`/feedback`, { state: { feedback } });
  setIsRecording(false);
};
```

**화면 구성:**
```
[마이크 버튼] [타이머] [종료 버튼]
─────────────────────────────────
```

녹음 종료 후 → `/feedback` 페이지에서 점수/개선안 표시

---

## AI & 오디오 처리 MVP (2일)

### 1. GPT-4o Realtime 최소 통합 (1일) - 진도희 (B팀)
```python
# 서버에서 한 번만 세션 생성해두기
async def create_gpt_session():
    response = await openai.realtime.sessions.create(
        model="gpt-4o-realtime-preview",
        voice="alloy"
    )
    # DB에 저장해두고 재사용
    return response.token

# 프론트에서는 WebSocket만 받기
```
- 프롬프트는 매우 간단하게: `"You are a Korean business mentor. User will speak. Respond naturally in Korean."`
- 음성 송수신은 나중 (지금은 텍스트만)

### 2. 오디오 처리 (1일) - 진도희 (B팀)
```javascript
// 브라우저에서 마이크 음성 → Base64 PCM
const audioContext = new (window.AudioContext || window.webkitAudioContext)();
const mediaStream = await navigator.mediaDevices.getUserMedia({ audio: true });
const source = audioContext.createMediaStreamSource(mediaStream);
const processor = audioContext.createScriptProcessor(4096, 1, 1);

processor.onaudioprocess = (event) => {
  const samples = event.inputBuffer.getChannelData(0);
  const pcm16 = new Int16Array(samples.length);
  
  for (let i = 0; i < samples.length; i++) {
    pcm16[i] = Math.max(-1, Math.min(1, samples[i])) < 0 
      ? samples[i] * 0x8000 
      : samples[i] * 0x7FFF;
  }
  
  const base64 = btoa(String.fromCharCode(...pcm16));
  // Signaling 서버로 전송
};

source.connect(processor);
processor.connect(audioContext.destination);
```
- 라이브러리 없이 Web Audio API만 사용
- 16bit PCM, 16kHz는 나중 (지금은 브라우저 기본값)

---

## 프롬프트 MVP (1일)

**6개 프롬프트** (김경민 4개, 진도희 2개)

### 김경민 (A팀) - 4개 프롬프트
```
[상사 보고]
You are a Korean business superior. 
The user will report to you about a project.
Ask follow-up questions naturally in Korean.
Respond in formal Korean with 존댓말.

[면접 연습]
You are a Korean job interviewer.
Ask about the user's experience and skills.
Follow up with challenging questions.
Respond in Korean.

[연인 갈등]
You are in a relationship with the user.
There's a conflict to resolve.
Express emotions naturally in Korean.
Respond in casual Korean with 반말.

[부모님 연락]
You are the user's parent.
Have a natural conversation about daily life.
Show care and concern in Korean.
Respond in Korean with parental tone.
```

### 진도희 (B팀) - 2개 프롬프트
```
[동료 협업]
You are a Korean coworker.
Discuss a project with the user.
Be professional but friendly.
Respond in Korean with 존댓말.

[교사-학부모]
You are a Korean teacher.
Discuss the student's progress with the parent.
Be professional and supportive.
Respond in Korean with 존댓말.
```

- 한 시나리오당 3~4줄만
- 예시나 톤 설정 없음
- 자세한 가이드는 Sprint 4에서

---

## 데이터베이스 MVP

기존 ERD 12개 엔티티 중 필요한 것만:

```sql
-- 이미 있음
User
Scenario

-- 추가 필요
CREATE TABLE DialogueSession (
  id VARCHAR(36) PRIMARY KEY,
  user_id BIGINT,
  scenario_id BIGINT,
  janus_room_id VARCHAR(255),
  ai_realtime_session_id VARCHAR(255),
  created_at TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES User(id),
  FOREIGN KEY (scenario_id) REFERENCES Scenario(id)
);

CREATE TABLE Feedback (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  session_id VARCHAR(36),
  transcript TEXT,
  audio_url VARCHAR(500),
  duration INT,
  wpm INT,
  filler_count INT,
  scores JSON,
  improvements JSON,
  created_at TIMESTAMP,
  FOREIGN KEY (session_id) REFERENCES DialogueSession(id)
);
```

---

## 테스트 MVP

**E2E 플로우 1번만 성공하면 됨:**

```
1. 로그인
2. 시나리오 선택
3. 마이크 버튼 클릭 → Signaling 연결 (에러 없음)
4. 5초 녹음 (더미 오디오)
5. 피드백 저장 & 화면 표시
```

**테스트 체크리스트:**
- [ ] POST /api/dialogues/start 200 응답
- [ ] WebSocket /ws/signaling 연결 성공
- [ ] WebRTC offer/answer 교환 성공
- [ ] 마이크 캡처 성공
- [ ] POST /api/feedback/save 200 응답
- [ ] 피드백 화면 표시

---

## 배포 MVP

```dockerfile
# docker-compose.yml
version: '3'
services:
  backend:
    image: dialogym-backend:latest
    ports: ["8080:8080"]
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
  
  frontend:
    image: dialogym-frontend:latest
    ports: ["80:80"]
```

- 프로덕션 배포는 git push만 (GitHub Actions)
- HTTPS는 이미 설정되어 있다고 가정

---

## 버린 것들 (Sprint 4로 미루기)

❌ **GPT-4 피드백 점수화** (간단한 수식만 사용)
❌ **오디오 레이턴시 최적화** (<200ms 목표 무시)
❌ **브라우저 호환성 테스트** (Chrome만)
❌ **프롬프트 엔지니어링** (기본 프롬프트만)
❌ **UI 디자인** (버튼과 텍스트만)
❌ **에러 처리** (기본 try-catch만)
❌ **로깅 & 모니터링** (나중)
❌ **API 문서** (Swagger는 코드 주석으로 대체)
❌ **동시 세션 테스트** (1명만)
❌ **실시간 피드백 WebSocket** (녹음 종료 후 한 번에 분석)
❌ **GPT-4o 음성 송수신** (텍스트 기반으로만 진행)
❌ **오디오 변환 파이프라인** (Web Speech API로 텍스트 변환)

---

## 예상 소요 시간 (수정본)

| 항목 | 시간 | 담당 |
|------|------|------|
| **Signaling 서버** | 2일 | 김경민 (A팀) |
| **실시간 분석** | 1일 | 김경민 (A팀) |
| **프롬프트 4개** | 0.5일 | 김경민 (A팀) |
| **WebRTC 클라이언트** | **2일** | **진도희 (B팀)** ⭐ |
| **GPT-4o 통합** | 1일 | 진도희 (B팀) |
| **오디오 처리** | 1일 | 진도희 (B팀) |
| **프롬프트 2개** | 0.5일 | 진도희 (B팀) |
| **프론트엔드 페이지** | 2일 | 왕택준 (C팀) |
| **세션/피드백 API** | 2일 | 왕택준 (C팀) |
| **배포 & 테스트** | 1일 | 공동 |
| **합계** | **9일** | |

---

## 팀별 작업 요약

### A팀: 김경민 (3.5일)
1. WebSocket Signaling 서버 (2일)
2. 실시간 분석 (WPM, 추임새) (1일)
3. 프롬프트 4개 작성 (0.5일)

### B팀: 진도희 (4.5일 + SM)
1. **WebRTC 클라이언트 (React)** (2일) ⭐
2. GPT-4o Realtime 최소 통합 (1일)
3. 오디오 처리 (Web Audio API) (1일)
4. 프롬프트 2개 작성 (0.5일)
5. SM 역할 (전주간)

### C팀: 왕택준 (4일)
1. 세션 생성 API (1일)
2. 피드백 저장 API (1일)
3. 프론트엔드 페이지 구조 (2일)

---

## 성공 기준 (MVP)

### 필수
- ✅ E2E 플로우 한 번 이상 성공 (에러 없음)
- ✅ Signaling 연결 성공
- ✅ **WebRTC 클라이언트 ↔ Signaling 서버 통신 성공**
- ✅ GPT-4o 세션 생성 성공
- ✅ 피드백 저장 & 표시
- ✅ 프로덕션에 배포됨

### 선택 (시간 있으면)
- 오디오 시각화
- 마이크 테스트 기능
- 에러 모니터링

---

## 주요 변경점 요약

| 변경 사항 | 이유 |
|----------|------|
| **WebRTC 클라이언트를 진도희에게 이관** | GPT-4o + 오디오 처리와 통합 관리 |
| **실시간 피드백 제거** | 녹음 종료 후 한 번에 처리 (단순화) |
| **GPT-4 점수화 제거** | 간단한 수식으로 대체 (시간 절약) |
| **오디오 최적화 미룸** | 일단 작동만 하면 됨 |

---

## 일정 (간소화)

### 10.13 (월)
- **왕택준**: 세션 API, React 프로젝트 초기화
- **김경민**: Signaling 서버 시작, 프롬프트 초안
- **진도희**: GPT-4o 문서 숙지, WebRTC 클라이언트 준비

### 10.14 (화)
- **왕택준**: 로그인/회원가입 페이지
- **김경민**: Signaling 서버 완성
- **진도희**: WebRTC 클라이언트 구현 시작

### 10.15 (수)
- **왕택준**: 시나리오 선택, 대화 화면 틀
- **김경민**: 실시간 분석, 프롬프트 완성
- **진도희**: WebRTC 클라이언트 완성, GPT-4o 통합

### 10.16 (목)
- **전체**: 통합 테스트

### 10.17 (금)
- **전체**: 버그 수정

### 10.18 (토)
- **전체**: 배포

### 10.19 (일)
- **Sprint Review & Retrospective**

---

## 기술 흐름 (수정본)

```
[User Mic]
   ↓ getUserMedia() (진도희)
   ↓ RTCPeerConnection (진도희)
   ↔ WebSocket /ws/signaling (김경민)
   ↔ Janus Media Server (김경민)
   ↔ GPT-4o Realtime API (진도희)
   ↔ Web Audio API (진도희)
   ↓
[User Speaker]
```

---

**핵심 원칙:**
1. **작동하는 것이 최우선**
2. **UI는 나중**
3. **복잡한 기능은 Sprint 4로**
4. **E2E 한 번만 성공하면 OK**

**화이팅! 🚀**