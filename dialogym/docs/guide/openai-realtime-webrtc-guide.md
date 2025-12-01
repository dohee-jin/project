# OpenAI Realtime API WebRTC 연결 가이드

**담당자 (Author)**: [진도희](https://github.com/dohee-jin)

**검토자 (Reviewer / PO·SM)**: [왕택준](https://github.com/TJK98)

**작성일 (Created)**: 2025.12.01

**문서 버전 (Version)**: v0.1

**문서 상태 (Status)**: Draft

---

## 대상 독자 (Intended Audience)

* **풀스택 개발자**: OpenAI Realtime API와 WebRTC 기반 음성 통신을 구현해야 하는 담당자
* **백엔드 개발자**: Ephemeral Key 발급 및 세션 관리 API를 구현하는 담당자
* **프론트엔드 개발자**: WebRTC P2P 연결 및 실시간 오디오 스트림을 처리하는 담당자
* **시스템 아키텍트**: 실시간 음성 AI 서비스의 전체 아키텍처를 설계하는 책임자

---

## 핵심 요약 (Executive Summary)

본 문서는 OpenAI Realtime API를 활용한 WebRTC 기반 음성 통신 시스템의 구현 방법을 설명합니다.
백엔드는 OpenAI로부터 Ephemeral Key를 발급받아 프론트엔드에 전달하고, 프론트엔드는 해당 키를 사용하여 WebRTC P2P 연결을 수립합니다.
연결 후 DataChannel을 통해 양방향 음성 및 텍스트 데이터를 실시간으로 주고받으며, WebSocket을 통해 대화 내역을 백엔드에 저장합니다.
사용자는 PTT(Push-To-Talk) 방식으로 발화하고, AI는 VAD(Voice Activity Detection) 기반으로 자동 응답합니다.

---

## 목차 (Table of Contents)

1. [문서 개요](#문서-개요-overview)
2. [시스템 아키텍처](#시스템-아키텍처)
3. [연결 플로우](#연결-플로우)
4. [백엔드 구현](#백엔드-구현)
5. [프론트엔드 구현](#프론트엔드-구현)
6. [주요 이벤트 타입](#주요-이벤트-타입)
7. [세션 관리 및 복구](#세션-관리-및-복구)
8. [에러 처리 및 재연결](#에러-처리-및-재연결)
9. [트러블슈팅](#트러블슈팅)
10. [변경 이력](#변경-이력-change-log)

---

## 문서 개요 (Overview)

OpenAI Realtime API는 GPT-4 모델과 실시간 음성 대화를 가능하게 하는 API입니다.
기존 텍스트 기반 API와 달리, WebRTC를 통한 P2P(Peer-to-Peer) 방식으로 낮은 지연시간의 음성 통신을 제공합니다.

본 문서는 다음과 같은 배경에서 작성되었습니다.

**작성 배경:**
- 실시간 음성 AI 대화 시스템 구현 필요
- 낮은 지연시간과 자연스러운 대화 흐름 요구
- 세션 관리 및 대화 내역 저장 기능 필요
- 네트워크 불안정 상황에서의 재연결 처리 필요

**적용 범위:**
- OpenAI Realtime API를 사용하는 모든 프로젝트
- WebRTC 기반 음성 통신 시스템
- Spring Boot(백엔드) + React(프론트엔드) 환경

---

## 시스템 아키텍처

### 전체 구성도

```
[사용자] <--WebRTC P2P--> [OpenAI Realtime API]
   |                              ^
   |                              |
   +--HTTP--> [백엔드] --HTTP---+
   |            |
   +--WebSocket-+ (대화 내역 저장)
```

### 구성 요소

| 구성 요소 | 역할 | 기술 스택 |
|----------|------|----------|
| 프론트엔드 | WebRTC 연결 및 음성 스트림 처리 | React, WebRTC API |
| 백엔드 | Ephemeral Key 발급, 세션 관리 | Spring Boot, RestTemplate |
| OpenAI Realtime API | 음성 인식 및 생성, 대화 처리 | GPT-4o Realtime |
| WebSocket | 실시간 대화 내역 전송 및 저장 | WebSocket |

---

## 연결 플로우

### 초기 연결 시퀀스

```
1. 프론트엔드 → 백엔드
   POST /api/v1/realtime/session
   { sessionId, model, voice, sttModel, language }

2. 백엔드 → OpenAI
   POST https://api.openai.com/v1/realtime/sessions
   { model, voice, instructions, turn_detection, input_audio_transcription }

3. OpenAI → 백엔드
   { client_secret: { value: "ephemeral_key" }, id: "session_id" }

4. 백엔드 → 프론트엔드
   { success: true, data: { client_secret, id } }

5. 프론트엔드 → OpenAI
   WebRTC SDP Offer/Answer 교환
   - createOffer()
   - POST https://api.openai.com/v1/realtime?model=...
   - setRemoteDescription(answer)

6. 프론트엔드 ↔ OpenAI
   DataChannel 'oai-events' 개방
   - 양방향 음성/텍스트 데이터 전송 시작
```

### 데이터 흐름

**음성 입력 (사용자 → AI):**
```
마이크 → getUserMedia() → RTCPeerConnection → OpenAI
                                              ↓
                                    input_audio_buffer.commit
                                              ↓
                                    STT 처리 완료
                                              ↓
                            conversation.item.input_audio_transcription.completed
```

**음성 출력 (AI → 사용자):**
```
OpenAI → response.audio.delta (청크) → 오디오 버퍼 수집 → Blob 생성 → <audio> 재생
         ↓
    response.audio_transcript.done (텍스트)
```

---

## 백엔드 구현

### Ephemeral Key 발급 API

**엔드포인트:**
```
POST /api/v1/realtime/session
```

**요청 파라미터:**
```java
public record RealtimeSessionRequest(
    Long sessionId,     // 대화 세션 ID
    String model,       // gpt-4o-realtime-preview-2024-10-01
    String voice,       // alloy, echo, fable, onyx, nova, shimmer
    String sttModel,    // whisper-1
    String language     // ko, en 등
) {}
```

**주요 처리 로직:**

```java
@PostMapping("/session")
public ResponseEntity<?> createEphemeralSession(@RequestBody RealtimeSessionRequest req) {
    // 1. 세션 검증
    DialogueSession session = dialogueSessionService.getSessionWithUserAndScenario(req.sessionId());
    
    if(session.getStatus() != SessionStatus.ONGOING) {
        throw new TrainException(SESSION_ALREADY_COMPLETED);
    }
    
    // 2. 프롬프트 생성
    Scenario scenario = session.getScenario();
    User user = session.getUser();
    String instructions = scenarioService.createPrompt(scenario, user);
    
    // 3. OpenAI API 호출
    String url = "https://api.openai.com/v1/realtime/sessions";
    
    HttpHeaders headers = new HttpHeaders();
    headers.setBearerAuth(openAiApiKey);
    headers.setContentType(MediaType.APPLICATION_JSON);
    
    Map<String, Object> request = new HashMap<>();
    request.put("model", req.model());
    request.put("voice", req.voice());
    request.put("instructions", instructions);
    request.put("turn_detection", Map.of(
        "type", "server_vad",
        "create_response", false  // 수동 응답 제어
    ));
    request.put("input_audio_transcription", Map.of(
        "model", req.sttModel(),
        "language", req.language()
    ));
    
    // 4. 임시키 발급 및 응답
    ResponseEntity<Map> response = restTemplate.postForEntity(url, 
        new HttpEntity<>(request, headers), Map.class);
    
    return ResponseEntity.ok()
        .body(ApiResponse.success("webRtc 연결을 위한 키 발급에 성공했습니다.", response.getBody()));
}
```

**핵심 설정 항목:**

| 항목 | 값 | 설명 |
|-----|---|------|
| model | gpt-4o-realtime-preview-2024-10-01 | Realtime API 전용 모델 |
| voice | alloy, echo, fable, onyx, nova, shimmer | AI 음성 종류 |
| turn_detection.type | server_vad | 서버 측 음성 감지 활성화 |
| turn_detection.create_response | false | 자동 응답 비활성화 (수동 제어) |
| input_audio_transcription.model | whisper-1 | STT 모델 |
| input_audio_transcription.language | ko | STT 언어 코드 |

⚠️ **중요:** `create_response: false` 설정으로 AI가 자동으로 응답하지 않고, 클라이언트에서 `response.create` 이벤트를 통해 명시적으로 응답을 요청해야 합니다.

---

## 프론트엔드 구현

### WebRTC 연결 초기화

**주요 단계:**

```javascript
const initWebRtc = async (ephemeralKey, sessionId) => {
    // 1. RTCPeerConnection 생성
    pcRef.current = new RTCPeerConnection({
        iceServers: [
            { urls: "stun:stun.l.google.com:19302" },
            {
                urls: "turn:openrelay.metered.ca:80",
                username: "openrelayproject",
                credential: "openrelayproject"
            }
        ]
    });
    
    // 2. DataChannel 생성
    dataChannelRef.current = pcRef.current.createDataChannel('oai-events');
    
    dataChannelRef.current.onopen = () => {
        console.log("✅ Data Channel 열림");
        setConnected(true);
    };
    
    // 3. 마이크 스트림 획득
    const localStream = await navigator.mediaDevices.getUserMedia({
        audio: {
            echoCancellation: true,
            noiseSuppression: false,
            autoGainControl: false,
            sampleRate: 16000,
            channelCount: 1
        }
    });
    
    localStreamRef.current = localStream;
    
    localStream.getTracks().forEach(track => {
        track.enabled = false;  // 초기 상태: 비활성화
        pcRef.current.addTrack(track, localStream);
    });
    
    // 4. AI 오디오 스트림 수신
    pcRef.current.ontrack = (event) => {
        if (audioTagRef.current) {
            audioTagRef.current.srcObject = event.streams[0];
            audioTagRef.current.autoplay = true;
            audioTagRef.current.play();
        }
    };
    
    // 5. SDP Offer/Answer 교환
    const offer = await pcRef.current.createOffer();
    await pcRef.current.setLocalDescription(offer);
    
    const sdpResponse = await fetch(
        `https://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-10-01`,
        {
            method: "POST",
            headers: {
                "Authorization": `Bearer ${ephemeralKey}`,
                "Content-Type": "application/sdp"
            },
            body: offer.sdp
        }
    );
    
    const answerSdp = await sdpResponse.text();
    await pcRef.current.setRemoteDescription({ type: "answer", sdp: answerSdp });
    
    console.log("✅ WebRTC 연결 완료");
};
```

### DataChannel 메시지 처리

**수신 이벤트 핸들러:**

```javascript
dataChannelRef.current.onmessage = (event) => {
    const data = JSON.parse(event.data);
    
    switch(data.type) {
        // AI 음성 청크 수신
        case "response.audio.delta":
            addAudioChunk(data.delta);  // Base64 디코딩 후 버퍼에 추가
            break;
            
        // AI 음성 전송 완료
        case "response.audio.done":
            createAndPlayBlob();  // Blob 생성 및 재생
            break;
            
        // AI 발화 시작
        case "output_audio_buffer.started":
            setAiSpeaking(true);
            break;
            
        // AI 텍스트 응답 완료
        case "response.audio_transcript.done":
            setTranscripts(prev => [...prev, {
                speaker: 'ai',
                text: data.transcript,
                timestamp: new Date().toISOString()
            }]);
            sendTranscript('ai', data.transcript);  // WebSocket으로 저장
            break;
            
        // 사용자 STT 완료
        case "conversation.item.input_audio_transcription.completed":
            setTranscripts(prev => [...prev, {
                speaker: 'user',
                text: data.transcript,
                timestamp: new Date().toISOString()
            }]);
            sendTranscript('user', data.transcript);  // WebSocket으로 저장
            
            // AI 응답 요청
            dataChannelRef.current.send(JSON.stringify({
                type: 'response.create',
                response: { modalities: ["audio", "text"] }
            }));
            break;
    }
};
```

### PTT(Push-To-Talk) 구현

**발화 시작:**

```javascript
const startPushToTalk = () => {
    if (!connected || aiSpeaking) return;
    
    // 마이크 활성화
    localStreamRef.current.getAudioTracks().forEach(track => {
        track.enabled = true;
    });
    
    setIsPttActive(true);
    setUserSpeaking(true);
    
    // 30초 타임아웃 설정
    pttTimerRef.current = setTimeout(() => {
        stopPushToTalk('timeout');
    }, 30000);
};
```

**발화 종료:**

```javascript
const stopPushToTalk = () => {
    if (!isPttActive) return;
    
    // 마이크 비활성화
    localStreamRef.current.getAudioTracks().forEach(track => {
        track.enabled = false;
    });
    
    setIsPttActive(false);
    setUserSpeaking(false);
    
    // 오디오 커밋 (STT 처리 요청)
    dataChannelRef.current.send(JSON.stringify({
        type: 'input_audio_buffer.commit'
    }));
};
```

---

## 주요 이벤트 타입

### 클라이언트 → OpenAI

| 이벤트 타입 | 설명 | 사용 시점 |
|-----------|------|----------|
| `input_audio_buffer.commit` | 사용자 음성 처리 요청 | PTT 종료 시 |
| `response.create` | AI 응답 생성 요청 | STT 완료 후 또는 첫 인사 |
| `conversation.item.create` | 컨텍스트 메시지 추가 | 세션 복구 또는 대화 이어가기 |

**예시:**

```javascript
// 응답 요청
dataChannelRef.current.send(JSON.stringify({
    type: 'response.create',
    response: { 
        modalities: ["audio", "text"] 
    }
}));

// 컨텍스트 추가
dataChannelRef.current.send(JSON.stringify({
    type: "conversation.item.create",
    item: {
        type: "message",
        role: "user",
        content: [{
            type: "input_text",
            text: "안녕하세요"
        }]
    }
}));
```

### OpenAI → 클라이언트

| 이벤트 타입 | 설명 | 페이로드 |
|-----------|------|---------|
| `response.audio.delta` | AI 음성 청크 | `{ delta: "base64_audio_data" }` |
| `response.audio.done` | AI 음성 전송 완료 | - |
| `output_audio_buffer.started` | AI 발화 시작 | - |
| `output_audio_buffer.stopped` | AI 발화 종료 | - |
| `response.audio_transcript.done` | AI 텍스트 응답 | `{ transcript: "..." }` |
| `conversation.item.input_audio_transcription.completed` | 사용자 STT 완료 | `{ transcript: "..." }` |

---

## 세션 관리 및 복구

### 세션 상태

| 상태 | 설명 |
|-----|------|
| `ongoing` | 진행 중 |
| `completed` | 정상 완료 |
| `abandoned` | 사용자 중단 |
| `failed` | 연결 실패 |

### 세션 복구 메커니즘

**WebSocket을 통한 대화 내역 복구:**

```javascript
// 1. 재연결 시 복구 요청
wsRef.current.send(JSON.stringify({
    type: "SESSION_RECONNECT",
    sessionId: sessionId,
    timestamp: new Date().toISOString()
}));

// 2. 백엔드로부터 대화 내역 수신
const handleSessionRecovery = (message) => {
    if (message.transcripts && message.transcripts.length > 0) {
        const recovered = message.transcripts.map(item => ({
            speaker: item.speaker.toLowerCase(),
            text: item.content,
            timestamp: item.timestamp
        }));
        
        setTranscripts(recovered);
        
        // 3. AI에게 컨텍스트 전송
        recovered.forEach((transcript, index) => {
            dataChannelRef.current.send(JSON.stringify({
                type: "conversation.item.create",
                item: {
                    id: `recovery_${Date.now()}_${index}`,
                    type: "message",
                    role: transcript.speaker === 'user' ? 'user' : 'assistant',
                    content: [{
                        type: "input_text",
                        text: transcript.text
                    }]
                }
            }));
        });
        
        // 4. 세션 복구 지시
        dataChannelRef.current.send(JSON.stringify({
            type: "conversation.item.create",
            item: {
                type: "message",
                role: "system",
                content: [{
                    type: "input_text",
                    text: "위 대화는 연결 끊김 전의 실제 대화입니다. 마지막 대화에서 자연스럽게 이어가세요."
                }]
            }
        }));
    }
};
```

---

## 에러 처리 및 재연결

### WebSocket 재연결

```javascript
const MAX_RECONNECT_ATTEMPTS = 5;
const RECONNECT_DELAY = 2000;

const attemptReconnect = (sessionId) => {
    reconnectAttemptsRef.current++;
    
    if (reconnectAttemptsRef.current > MAX_RECONNECT_ATTEMPTS) {
        console.error('재연결 실패: 최대 시도 횟수 초과');
        failSession(scenarioId, userId, 'WebSocket 재연결 실패');
        return;
    }
    
    const delay = RECONNECT_DELAY * Math.pow(2, reconnectAttemptsRef.current - 1);
    
    setTimeout(() => {
        initWebSocket(sessionId, true);
    }, delay);
};
```

### WebRTC 연결 상태 모니터링

```javascript
pcRef.current.onconnectionstatechange = () => {
    console.log("연결 상태:", pcRef.current.connectionState);
    
    switch(pcRef.current.connectionState) {
        case 'disconnected':
            console.log('연결이 끊어졌습니다. 재연결을 시도합니다...');
            attemptFullReconnect('WebRTC 연결 끊김');
            break;
            
        case 'failed':
            console.log('연결에 실패했습니다.');
            failSession(scenarioId, userId, 'WebRTC 연결 실패');
            break;
            
        case 'connected':
            console.log('연결 성공');
            break;
    }
};
```

### 마이크 권한 에러 처리

```javascript
try {
    const localStream = await navigator.mediaDevices.getUserMedia({
        audio: { /* ... */ }
    });
} catch (err) {
    if (err.name === 'NotAllowedError' || err.name === 'PermissionDeniedError') {
        throw new Error('마이크 권한이 필요합니다. 브라우저에서 마이크 권한을 허용해주세요.');
    }
    throw err;
}
```

---

## 트러블슈팅

### 일반적인 문제

| 문제 | 원인 | 해결 방법 |
|-----|------|----------|
| DataChannel이 열리지 않음 | WebRTC 연결 실패 | TURN 서버 설정 확인, 방화벽 점검 |
| 마이크 입력이 전송되지 않음 | track.enabled = false | PTT 시작 시 track.enabled = true 설정 |
| AI 음성이 재생되지 않음 | audio 태그 설정 오류 | autoplay, srcObject 확인 |
| STT 결과가 오지 않음 | input_audio_buffer.commit 누락 | PTT 종료 시 커밋 이벤트 전송 확인 |
| AI가 응답하지 않음 | response.create 미전송 | STT 완료 후 response.create 이벤트 전송 |

### 디버깅 팁

**DataChannel 상태 확인:**
```javascript
console.log('DataChannel 상태:', dataChannelRef.current?.readyState);
// 'open'이어야 정상
```

**RTCPeerConnection 상태 확인:**
```javascript
console.log('연결 상태:', pcRef.current?.connectionState);
console.log('ICE 상태:', pcRef.current?.iceConnectionState);
```

**마이크 트랙 상태 확인:**
```javascript
localStreamRef.current?.getAudioTracks().forEach(track => {
    console.log('트랙 상태:', track.enabled, track.readyState);
});
```

### 성능 최적화

**오디오 버퍼링:**
- AI 음성 청크를 수집하여 Blob으로 변환 후 재생
- 네트워크 지연 시에도 부드러운 재생 가능

```javascript
// 청크 수집
const addAudioChunk = (base64Data) => {
    const binaryString = atob(base64Data);
    const bytes = new Uint8Array(binaryString.length);
    for (let i = 0; i < binaryString.length; i++) {
        bytes[i] = binaryString.charCodeAt(i);
    }
    audioChunksRef.current.push(bytes.buffer);
};

// Blob 생성 및 재생
const createAndPlayBlob = () => {
    const blob = new Blob(audioChunksRef.current, { type: 'audio/pcm' });
    const blobUrl = URL.createObjectURL(blob);
    audioTagRef.current.src = blobUrl;
    audioTagRef.current.play();
};
```

---

## 변경 이력 (Change Log)

| 버전 | 변경 일자 | 작성자 | 주요 변경 내용 |
|------|-----------|--------|----------------|
| v0.1 | 2025.12.01 | 작성자명 | 최초 작성 |