# WebSocket 재활용 가이드

## 📋 목차
1. [개요](#개요)
2. [아키텍처 변경](#아키텍처-변경)
3. [백엔드 구현](#백엔드-구현)
4. [프론트엔드 구현](#프론트엔드-구현)
5. [재시도 로직](#재시도-로직)
6. [테스트 시나리오](#테스트-시나리오)
7. [트러블슈팅](#트러블슈팅)

---

## 개요

### 목적
기존에 구현된 WebSocket을 활용하여 대화 내용(Transcript)을 실시간으로 백엔드에 전송하고 저장합니다.

### 주요 장점
- ✅ **데이터 안전성**: 연결 끊김 시에도 이미 저장된 대화 보존
- ✅ **재연결 지원**: 저장된 대화를 불러와 이어서 진행 가능
- ✅ **실시간 모니터링**: 관리자 대시보드에서 실시간 대화 확인
- ✅ **기존 코드 활용**: 팀원이 만든 WebSocket 인프라 재사용

### 변경 전/후 비교

#### 변경 전
```
대화 진행 → 로컬 state에만 저장 → 세션 종료 시 일괄 전송
문제: 연결 끊김 시 모든 대화 손실 💥
```

#### 변경 후
```
대화 진행 → 즉시 WebSocket으로 전송 → DB 즉시 저장
결과: 연결 끊김 시에도 이미 저장된 대화 안전 ✅
```

---

## 아키텍처 변경

### 전체 흐름

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│  프론트엔드  │         │   백엔드      │         │  OpenAI GPT │
└─────────────┘         └──────────────┘         └─────────────┘
       │                       │                        │
       │ 1. POST /sessions     │                        │
       │ ──────────────────>   │                        │
       │ ← sessionId           │                        │
       │                       │                        │
       │ 2. WS Connect         │                        │
       │    /ws/transcript/id  │                        │
       │ ═══════════════════>  │                        │
       │    (WebSocket 연결)   │                        │
       │                       │                        │
       │ 3. WebRTC P2P         │                        │
       │ ═══════════════════════════════════════════>   │
       │    (오디오 직접 전송)  │                       │
       │                       │                        │
       │ 4. 대화 진행          │                        │
       │ - 사용자 발화         │                        │
       │ - AI 응답             │                        │
       │                       │                        │
       │ 5. Transcript 실시간 전송                      │
       │ ──────────────────>   │                        │
       │    via WebSocket      │ → DB 즉시 저장 💾      │
       │                       │                        │
       │ 6. 저장 확인          │                        │
       │ <──────────────────   │                        │
       │    { type: "saved" }  │                        │
```

### 데이터 흐름

```
GPT → Data Channel → 프론트엔드 → WebSocket → 백엔드 → DB
              ↓
         Local State
         (UI 표시용)
```

---

## 백엔드 구현

### 1. TranscriptWebSocketHandler 생성

**파일 위치**: `backend/websocket/handler/TranscriptWebSocketHandler.java`

```java
package com.aid.train.backend.websocket.handler;

import com.aid.train.backend.domain.session.service.TranscriptService;
import com.aid.train.backend.websocket.dto.TranscriptMessage;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Transcript 실시간 전송 WebSocket Handler
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class TranscriptWebSocketHandler extends TextWebSocketHandler {
    
    private final TranscriptService transcriptService;
    private final ObjectMapper objectMapper;
    
    // sessionId → WebSocketSession 매핑
    private final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {
        String sessionId = extractSessionId(session);
        sessions.put(sessionId, session);
        
        log.info("✅ Transcript WebSocket 연결 - sessionId: {}", sessionId);
        
        // 연결 확인 메시지
        TranscriptMessage confirmMessage = TranscriptMessage.builder()
                .type("connected")
                .message("Transcript WebSocket 연결 완료")
                .build();
        
        session.sendMessage(new TextMessage(
            objectMapper.writeValueAsString(confirmMessage)
        ));
    }
    
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) 
            throws Exception {
        String sessionId = extractSessionId(session);
        
        try {
            TranscriptMessage transcriptMsg = objectMapper.readValue(
                message.getPayload(), 
                TranscriptMessage.class
            );
            
            log.info("📨 Transcript 수신 - sessionId: {}, speaker: {}", 
                    sessionId, transcriptMsg.getSpeaker());
            
            // DB 저장
            if ("user".equalsIgnoreCase(transcriptMsg.getSpeaker())) {
                transcriptService.saveUserTranscript(
                    sessionId, 
                    transcriptMsg.getText()
                );
            } else if ("ai".equalsIgnoreCase(transcriptMsg.getSpeaker())) {
                transcriptService.saveAiTranscript(
                    sessionId, 
                    transcriptMsg.getText()
                );
            }
            
            // 저장 완료 확인
            TranscriptMessage ackMessage = TranscriptMessage.builder()
                    .type("saved")
                    .message("Transcript 저장 완료")
                    .speaker(transcriptMsg.getSpeaker())
                    .build();
            
            session.sendMessage(new TextMessage(
                objectMapper.writeValueAsString(ackMessage)
            ));
            
        } catch (Exception e) {
            log.error("❌ Transcript 처리 실패 - sessionId: {}", sessionId, e);
            
            TranscriptMessage errorMessage = TranscriptMessage.builder()
                    .type("error")
                    .message("저장 실패: " + e.getMessage())
                    .build();
            
            session.sendMessage(new TextMessage(
                objectMapper.writeValueAsString(errorMessage)
            ));
        }
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        String sessionId = extractSessionId(session);
        sessions.remove(sessionId);
        log.info("⚠️ Transcript WebSocket 닫힘 - sessionId: {}", sessionId);
    }
    
    private String extractSessionId(WebSocketSession session) {
        String path = session.getUri().getPath();
        String[] parts = path.split("/");
        return parts[parts.length - 1];
    }
}
```

### 2. TranscriptMessage DTO 생성

**파일 위치**: `backend/websocket/dto/TranscriptMessage.java`

```java
package com.aid.train.backend.websocket.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class TranscriptMessage {
    private String type;        // "transcript", "saved", "error", "connected"
    private String speaker;     // "user" or "ai"
    private String text;        // 발화 내용
    private String timestamp;   // ISO 8601
    private String message;     // 상태 메시지
}
```

### 3. WebSocketConfig 수정

**파일 위치**: `backend/websocket/config/WebSocketConfig.java`

```java
@Configuration
@EnableWebSocket
@RequiredArgsConstructor
public class WebSocketConfig implements WebSocketConfigurer {

    private final FeedbackHandler feedbackHandler;
    private final TranscriptWebSocketHandler transcriptWebSocketHandler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        // ⭐ Transcript 실시간 전송용
        registry.addHandler(transcriptWebSocketHandler, "/ws/transcript/{sessionId}")
                .setAllowedOriginPatterns("*");
        
        // 기존 Feedback WebSocket
        registry.addHandler(feedbackHandler, "/ws/feedback/{sessionId}")
                .setAllowedOriginPatterns("*");
    }

    @Bean
    public ServletServerContainerFactoryBean createWebSocketContainer() {
        ServletServerContainerFactoryBean container = 
            new ServletServerContainerFactoryBean();
        container.setMaxTextMessageBufferSize(1024 * 1024);
        container.setMaxBinaryMessageBufferSize(1024 * 1024);
        container.setMaxSessionIdleTimeout(15 * 60000L);
        return container;
    }
}
```

### 4. 세션 완료 API 추가

**파일 위치**: `backend/domain/session/controller/DialogueSessionController.java`

```java
/**
 * 세션 완료 처리
 * (transcript는 이미 WebSocket으로 저장됨)
 */
@PostMapping("/complete")
public ResponseEntity<?> completeSession(
        @RequestBody SessionCompleteRequest request) {
    
    log.info("세션 완료 요청 - sessionId: {}", request.getSessionId());

    dialogueSessionService.completeSession(request.getSessionId());

    return ResponseEntity.ok().body(
        ApiResponse.success("세션 완료", null)
    );
}

@Data
class SessionCompleteRequest {
    private String sessionId;
}
```

---

## 프론트엔드 구현

### 1. WebSocket 연결 및 상태 관리

```javascript
// AiTest.jsx
const transcriptWsRef = useRef(null);
const [wsConnected, setWsConnected] = useState(false);

/**
 * Transcript WebSocket 연결
 */
const connectTranscriptWebSocket = (sessionId) => {
    console.log("🔌 Transcript WebSocket 연결 중...");
    
    const ws = new WebSocket(`ws://localhost:8080/ws/transcript/${sessionId}`);
    
    ws.onopen = () => {
        console.log("✅ Transcript WebSocket 연결됨");
        setWsConnected(true);
    };
    
    ws.onmessage = (event) => {
        try {
            const message = JSON.parse(event.data);
            console.log("📨 WebSocket 메시지:", message);
            
            if (message.type === 'connected') {
                console.log("✅ 연결 확인:", message.message);
            }
            
            if (message.type === 'saved') {
                console.log("💾 저장 완료:", message.speaker);
            }
            
            if (message.type === 'error') {
                console.error("❌ 저장 에러:", message.message);
            }
            
        } catch (err) {
            console.error("❌ 메시지 파싱 실패:", err);
        }
    };
    
    ws.onerror = (error) => {
        console.error("❌ WebSocket 에러:", error);
        setWsConnected(false);
    };
    
    ws.onclose = () => {
        console.log("⚠️ WebSocket 닫힘");
        setWsConnected(false);
    };
    
    transcriptWsRef.current = ws;
};
```

### 2. Transcript 실시간 전송

```javascript
/**
 * Transcript를 WebSocket으로 즉시 전송
 */
const sendTranscriptToBackend = (speaker, text) => {
    if (!transcriptWsRef.current || 
        transcriptWsRef.current.readyState !== WebSocket.OPEN) {
        console.warn("⚠️ WebSocket 연결 안됨 - 로컬에만 저장");
        return;
    }
    
    const message = {
        type: "transcript",
        speaker: speaker,  // "user" or "ai"
        text: text,
        timestamp: new Date().toISOString()
    };
    
    transcriptWsRef.current.send(JSON.stringify(message));
    console.log("📤 Transcript 전송:", speaker, text);
};
```

### 3. Data Channel 메시지 처리 수정

```javascript
dataChannelRef.current.onmessage = (event) => {
    try {
        const data = JSON.parse(event.data);
        console.log("📨 GPT 이벤트:", data.type);

        // 사용자 발화 완료
        if (data.type === 'conversation.item.input_audio_transcription.completed') {
            console.log("👤 사용자:", data.transcript);
            
            // 로컬 state 업데이트
            setTranscripts(prev => [...prev, {
                speaker: 'user',
                text: data.transcript,
                timestamp: new Date().toISOString()
            }]);
            
            // ⭐ 즉시 백엔드로 전송
            sendTranscriptToBackend('user', data.transcript);
        }

        // AI 응답 완료
        if (data.type === 'response.audio_transcript.done') {
            console.log("🤖 AI:", data.transcript);
            
            // 로컬 state 업데이트
            setTranscripts(prev => [...prev, {
                speaker: 'ai',
                text: data.transcript,
                timestamp: new Date().toISOString()
            }]);
            
            // ⭐ 즉시 백엔드로 전송
            sendTranscriptToBackend('ai', data.transcript);
        }

    } catch (err) {
        console.error("❌ 이벤트 파싱 실패:", err);
    }
};
```

### 4. 초기화 시 WebSocket 연결

```javascript
const initRealtimeConnection = async () => {
    setLoading(true);
    
    try {
        // 1. DialogueSession 생성
        const sessionResponse = await apiClient.post('/dialogue/sessions', {
            scenarioId: scenarioId
        });
        const newSessionId = sessionResponse.data.data.sessionId;
        setSessionId(newSessionId);
        
        // 2. ⭐ Transcript WebSocket 연결
        connectTranscriptWebSocket(newSessionId);
        
        // 3. Ephemeral Key 발급
        // 4. WebRTC 연결
        // ...
        
    } catch (err) {
        console.error("❌ 연결 실패:", err);
        cleanupConnection();
    } finally {
        setLoading(false);
    }
};
```

### 5. 세션 종료 수정

```javascript
const handleEndSession = async () => {
    if (loading) return;
    setLoading(true);

    try {
        console.log("🛑 세션 종료 시작...");

        if (!sessionId) {
            console.warn("⚠️ sessionId 없음");
            cleanupConnection();
            return;
        }

        // ⭐ 이미 WebSocket으로 실시간 저장했으므로
        // 세션 상태만 완료 처리
        await apiClient.post('/dialogue/sessions/complete', {
            sessionId: sessionId
        });

        console.log("✅ 세션 완료");
        cleanupConnection();
        alert("대화가 저장되었습니다!");

    } catch (err) {
        console.error("❌ 세션 종료 실패:", err);
        alert(`세션 종료 중 오류: ${err.message}`);
        cleanupConnection();
    } finally {
        setLoading(false);
    }
};
```

### 6. 정리 시 WebSocket 닫기

```javascript
const cleanupConnection = () => {
    console.log("🧹 연결 정리 중...");

    // PeerConnection 정리
    if (pcRef.current) {
        pcRef.current.close();
        pcRef.current = null;
    }

    // ... 기타 정리 코드 ...
    
    // ⭐ Transcript WebSocket 닫기
    if (transcriptWsRef.current) {
        transcriptWsRef.current.close();
        transcriptWsRef.current = null;
    }
    
    setWsConnected(false);
    
    console.log("✅ 정리 완료");
};
```

---

## 재시도 로직

### 자동 재연결

```javascript
const sendTranscriptToBackend = (speaker, text, retryCount = 0) => {
    if (!transcriptWsRef.current || 
        transcriptWsRef.current.readyState !== WebSocket.OPEN) {
        
        console.warn("⚠️ WebSocket 연결 안됨");
        
        // 재연결 시도 (최대 3회)
        if (retryCount < 3) {
            console.log(`🔄 재연결 시도 ${retryCount + 1}/3`);
            
            setTimeout(() => {
                connectTranscriptWebSocket(sessionId);
                sendTranscriptToBackend(speaker, text, retryCount + 1);
            }, 1000 * (retryCount + 1));  // 1초, 2초, 3초 대기
        } else {
            console.error("❌ 재연결 실패 - 로컬에만 저장");
            // TODO: 실패한 transcript를 별도 큐에 저장 후 나중에 전송
        }
        
        return;
    }
    
    const message = {
        type: "transcript",
        speaker: speaker,
        text: text,
        timestamp: new Date().toISOString()
    };
    
    transcriptWsRef.current.send(JSON.stringify(message));
    console.log("📤 Transcript 전송:", speaker, text);
};
```

### 실패한 Transcript 큐 관리 (선택사항)

```javascript
const failedTranscriptsRef = useRef([]);

const sendTranscriptToBackend = (speaker, text, retryCount = 0) => {
    // ... 재연결 로직 ...
    
    if (retryCount >= 3) {
        // 실패 큐에 저장
        failedTranscriptsRef.current.push({
            speaker: speaker,
            text: text,
            timestamp: new Date().toISOString()
        });
        console.log("💾 실패 큐에 저장:", failedTranscriptsRef.current.length);
    }
};

// WebSocket 재연결 성공 시 실패 큐 재전송
ws.onopen = () => {
    console.log("✅ WebSocket 연결됨");
    setWsConnected(true);
    
    // 실패 큐 재전송
    if (failedTranscriptsRef.current.length > 0) {
        console.log("🔄 실패 큐 재전송:", failedTranscriptsRef.current.length);
        
        failedTranscriptsRef.current.forEach(transcript => {
            sendTranscriptToBackend(transcript.speaker, transcript.text);
        });
        
        failedTranscriptsRef.current = [];
    }
};
```

---

## 테스트 시나리오

### 1. 정상 흐름 테스트

```
✅ 연결 시작
✅ WebSocket 연결 확인
✅ GPT 첫 인사
✅ 사용자 발화 → 즉시 DB 저장 확인
✅ AI 응답 → 즉시 DB 저장 확인
✅ 대화 3회 반복
✅ 세션 종료
✅ DB에서 모든 대화 확인
```

### 2. 연결 끊김 테스트

```
✅ 연결 시작
✅ 대화 5회 진행 (모두 저장됨)
❌ 네트워크 끊김 (개발자도구에서 오프라인 모드)
✅ DB에서 5개 대화 확인 (손실 없음)
✅ 네트워크 복구
✅ 재연결
✅ 이어서 대화 진행
```

### 3. WebSocket 재연결 테스트

```
✅ 연결 시작
✅ 대화 3회 진행
❌ WebSocket 강제 종료 (ws.close())
✅ 사용자 발화 시 자동 재연결 시도
✅ 재연결 성공 후 정상 전송
```

---

## 트러블슈팅

### 문제 1: WebSocket 연결 안됨

**증상**
```
❌ WebSocket connection failed
```

**해결**
1. CORS 설정 확인
```java
.setAllowedOriginPatterns("*")
```

2. 백엔드 실행 확인
```bash
# 로그 확인
✅ Transcript WebSocket 연결 - sessionId: xxx
```

3. URL 확인
```javascript
// 올바른 형식
ws://localhost:8080/ws/transcript/{sessionId}

// 잘못된 형식
ws://localhost:8080/ws/transcript  // sessionId 누락
```

### 문제 2: Transcript 저장 안됨

**증상**
```
📤 Transcript 전송: user 안녕하세요
(저장 완료 메시지 없음)
```

**해결**
1. 백엔드 로그 확인
```
❌ Transcript 처리 실패 - sessionId: xxx
```

2. TranscriptService 메서드 확인
```java
transcriptService.saveUserTranscript(sessionId, text);
```

3. DB 연결 확인

### 문제 3: 재연결 무한 루프

**증상**
```
🔄 재연결 시도 1/3
🔄 재연결 시도 2/3
🔄 재연결 시도 3/3
🔄 재연결 시도 1/3  // 다시 시작
```

**해결**
```javascript
// 재연결 중 플래그 추가
const isReconnectingRef = useRef(false);

const sendTranscriptToBackend = (speaker, text, retryCount = 0) => {
    if (isReconnectingRef.current) {
        console.log("⏳ 재연결 중... 대기");
        return;
    }
    
    if (retryCount < 3) {
        isReconnectingRef.current = true;
        
        setTimeout(() => {
            connectTranscriptWebSocket(sessionId);
            isReconnectingRef.current = false;
            sendTranscriptToBackend(speaker, text, retryCount + 1);
        }, 1000 * (retryCount + 1));
    }
};
```

### 문제 4: 중복 저장

**증상**
```
💾 저장 완료: user
💾 저장 완료: user  // 중복
```

**해결**
1. 중복 전송 방지
```javascript
const sentTranscriptsRef = useRef(new Set());

const sendTranscriptToBackend = (speaker, text) => {
    const key = `${speaker}-${text}-${Date.now()}`;
    
    if (sentTranscriptsRef.current.has(key)) {
        console.log("⚠️ 중복 전송 방지");
        return;
    }
    
    sentTranscriptsRef.current.add(key);
    
    // 전송 로직
    // ...
    
    // 1분 후 키 제거 (메모리 관리)
    setTimeout(() => {
        sentTranscriptsRef.current.delete(key);
    }, 60000);
};
```

---

## 체크리스트

### 백엔드
- [ ] TranscriptWebSocketHandler 생성
- [ ] TranscriptMessage DTO 생성
- [ ] WebSocketConfig 수정
- [ ] SessionCompleteRequest DTO 추가
- [ ] /complete API 추가
- [ ] CORS 설정 확인

### 프론트엔드
- [ ] transcriptWsRef 추가
- [ ] wsConnected 상태 추가
- [ ] connectTranscriptWebSocket 구현
- [ ] sendTranscriptToBackend 구현
- [ ] Data Channel 메시지 처리 수정
- [ ] initRealtimeConnection 수정
- [ ] handleEndSession 수정
- [ ] cleanupConnection 수정
- [ ] 재시도 로직 추가 (선택)

### 테스트
- [ ] 정상 흐름 테스트
- [ ] 연결 끊김 테스트
- [ ] WebSocket 재연결 테스트
- [ ] DB 저장 확인
- [ ] 중복 저장 방지 확인

---

## 참고 자료

### WebSocket 연결 상태

| readyState | 값 | 설명 |
|------------|-----|------|
| CONNECTING | 0 | 연결 중 |
| OPEN | 1 | 연결됨 |
| CLOSING | 2 | 닫는 중 |
| CLOSED | 3 | 닫힘 |

### 메시지 타입

| type | 방향 | 설명 |
|------|------|------|
| connected | 백→프 | 연결 확인 |
| transcript | 프→백 | 대화 전송 |
| saved | 백→프 | 저장 완료 |
| error | 백→프 | 에러 발생 |

### 유용한 로그

```javascript
// WebSocket 디버깅
console.log('WS State:', ws.readyState);
console.log('WS URL:', ws.url);
console.log('WS Protocol:', ws.protocol);
console.log('WS Extensions:', ws.extensions);
```

---

## 문의사항

구현 중 문제가 발생하면:
1. 콘솔 로그 확인
2. 백엔드 로그 확인
3. 네트워크 탭 확인
4. 이 가이드의 트러블슈팅 참고

---

**작성일**: 2025-01-15  
**버전**: 1.0  
**작성자**: AI Assistant
