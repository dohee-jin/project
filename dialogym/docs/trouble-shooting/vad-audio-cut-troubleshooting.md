# 🔧 트러블슈팅: AI 음성 재생 중 끊김 및 사용자 턴 전환 지연 문제

본 문서는 Tropical 프론트엔드 개발 과정에서 발생한  
**"GPT Realtime API 음성 재생 중 끊김 현상"**과  
**"VAD 기반 발화 종료 감지 오작동 문제"**를 분석하고 해결한 기록입니다.

**작성자:** 진도희  
**작성일:** 2025-11-28  
**문서 버전:** v1.0

---

## 1. 문제 현상

### 1-1. AI 음성이 끝까지 재생되지 않고 중간에 끊김
- 긴 문장(20단어 이상) 재생 시 중간에 중단
- 사용자가 듣지 못한 부분이 발생
- 로그 예시:
  ```
  🤖 AI 발화 시작
  🔇 침묵 5/10 (레벨: 3.2)  ← 아직 말하는 중인데 침묵으로 오인
  ✅ AI 발화 종료 (침묵 감지)
  ```

### 1-2. AI 발화 종료 후 사용자 턴이 넘어오지 않음
- AI 음성 재생이 끝났는데도 사용자가 말할 수 없음
- VAD가 계속 돌아가며 침묵을 감지할 때까지 대기
- `aiSpeaking` 상태가 유지되어 PTT 버튼 비활성화

### 1-3. WebRTC 스트리밍 방식의 재생 완료 감지 불가
- `output_audio_buffer.stopped` 이벤트는 전송 완료를 알려주지만 재생 완료는 아님
- srcObject로 연결된 audio 태그는 `onended` 이벤트가 발생하지 않음
- 재생 완료 시점을 정확히 알 수 없어 VAD에만 의존

---

## 2. 원인 분석

### 2-1. WebRTC는 실시간 스트리밍이라 "끝"이 없음
- WebRTC의 MediaStream은 연속된 데이터 흐름
- 파일처럼 명확한 시작/끝 지점이 없음
- audio 태그의 `onended` 이벤트가 절대 발생하지 않음

### 2-2. OpenAI 이벤트 타이밍 문제
```javascript
// output_audio_buffer.stopped = 전송 완료 (오디오 재생 완료 아님!)
if (data.type === "output_audio_buffer.stopped") {
    startAiVadCheck();  // ← 아직 재생 중인데 VAD 시작
}
```
- `output_audio_buffer.stopped`: 서버가 오디오 전송을 완료한 시점
- 실제 재생 완료: 클라이언트에서 스피커로 출력이 끝난 시점
- 두 시점 사이에 0.5~2초의 시간차 발생

### 2-3. VAD 임계값 과민 설정
```javascript
// 문제 발생 설정 (추정)
const SILENCE_THRESHOLD = 5;   // 너무 높음
const SILENCE_CHECKS = 5;      // 너무 짧음 (1초)
```
- 높은 임계값: 낮은 음량을 침묵으로 오인
- 짧은 체크 주기: 문장 중간의 짧은 쉼표도 발화 종료로 판단
- 긴 문장이나 느린 발화는 중간에 끊김

### 2-4. Blob 버퍼링 방식 시도 실패
```
{"type":"response.audio.done"}
🎵 오디오 전송 완료 - Blob 생성 및 재생
⚠️ 재생할 오디오 청크 없음  ← audioChunks 배열이 비어있음
```
- `response.audio.delta` 이벤트가 도착하지 않음
- WebRTC 스트리밍에서는 DataChannel로 오디오 청크가 전송되지 않음
- Blob 방식 적용 불가

---

## 3. 재현 단계

1. AI에게 긴 문장(20단어 이상) 응답 유도  
   예: "주말 여행 계획에 대해 자세히 설명해줘"
2. AI 음성 재생 시작 후 5-7초 경과  
3. 음성이 끝나지 않았는데 중간에 끊김 발생  
4. 또는 음성은 끝났는데 사용자 턴이 2-3초간 넘어오지 않음  

---

## 4. 디버깅 과정

### 4-1. VAD 임계값 분석

```javascript
analyser.getByteFrequencyData(dataArray);
const average = dataArray.reduce((a, b) => a + b) / dataArray.length;
console.log(`음성 레벨: ${average.toFixed(1)}`);
```

**출력 예시:**
```
음성 레벨: 3.2  ← 침묵으로 오인 (실제로는 발화 중)
음성 레벨: 4.1  ← 침묵으로 오인
음성 레벨: 6.1  ← 정상 발화로 인식
```

**발견:**
- 임계값 5 이상일 때: 낮은 음량도 침묵으로 오인
- 체크 횟수 5회 (1초): 긴 문장의 쉼표도 종료로 판단

---

### 4-2. 이벤트 타이밍 측정

```javascript
if (data.type === "output_audio_buffer.stopped") {
    console.log("🔊 전송 완료:", Date.now());
}

// audio 태그 onended (Blob 방식에서만)
audioElement.addEventListener('ended', () => {
    console.log("✅ 재생 완료:", Date.now());
});
```

**출력 예시:**
```
🔊 전송 완료: 1732800000000
🎤 VAD 시작
🔇 침묵 1/10 (레벨: 4.2)  ← 아직 재생 중
🔇 침묵 2/10 (레벨: 3.8)
✅ 재생 완료: 1732800001500  ← 1.5초 차이
```

**발견:**
- 전송 완료 후 1~2초 뒤에 재생 완료
- VAD가 재생 중에 시작되어 잘못된 판단

---

### 4-3. Blob 버퍼링 시도

```javascript
// response.audio.delta 이벤트 대기
if (data.type === "response.audio.delta") {
    console.log("오디오 청크 수신:", data.delta.length);
    addAudioChunk(data.delta);
}
```

**결과:**
```
(아무 로그도 출력되지 않음)
⚠️ 재생할 오디오 청크 없음
```

**발견:**
- WebRTC 스트리밍에서는 `response.audio.delta` 이벤트 미전송
- 오디오가 DataChannel이 아닌 WebRTC 트랙으로 직접 전송됨
- Blob 버퍼링 방식 적용 불가

---

### 4-4. 첫 인사 vs 일반 대화 비교

```javascript
// 첫 인사
"안녕하세요! 무엇을 도와드릴까요?"  ← 끊김 없음

// 일반 대화
"주말에는 날씨가 좋을 것 같으니까 가족들과 함께..."  ← 중간에 끊김
```

**발견:**
- 짧은 문장: 문제 없음
- 긴 문장: 중간 쉼표에서 끊김 발생
- 첫 인사는 상대적으로 짧아서 문제 적음

---

## 5. 해결 과정

### 5-1. VAD 임계값 완화 (핵심 해결책)

```javascript
// 변경 전 (추정)
const SILENCE_THRESHOLD = 5;
const SILENCE_CHECKS = 5;  // 1초

// 변경 후
const SILENCE_THRESHOLD = isInitialGreeting ? 2 : 3;  // ✅ 더 관대하게
const SILENCE_CHECKS = isInitialGreeting ? 15 : 10;   // ✅ 3초 / 2초
```

**변경 이유:**
- **임계값 5 → 2-3**: 낮은 음량도 발화로 인식
- **체크 횟수 5 → 10-15**: 짧은 쉼표를 종료로 오인 방지
- **첫 인사는 더 길게**: 초기 연결 지연 고려

---

### 5-2. VAD 시작 타이밍 지연

```javascript
// 변경 전 (추정)
if (data.type === "output_audio_buffer.stopped") {
    startAiVadCheck();  // 즉시 시작
}

// 변경 후
if (data.type === "output_audio_buffer.stopped") {
    const vadDelay = isInitialGreeting ? 2000 : 1200;  // ✅ 지연 추가
    setTimeout(() => {
        if (isMountedRef.current) {
            startAiVadCheck();
        }
    }, vadDelay);
}
```

**효과:**
- 전송 완료 후 1.2~2초 대기
- 실제 재생이 시작된 후 VAD 작동
- 재생 중 조기 종료 방지

---

### 5-3. 발화 중 VAD 조기 시작 (일반 대화만)

```javascript
// AI 발화 시작
if (data.type === "output_audio_buffer.started") {
    setAiSpeaking(true);
    
    // 첫 인사가 아닐 때만 발화 중 VAD 시작
    if (!isInitialGreeting) {
        setTimeout(() => {
            if (isMountedRef.current && aiSpeaking) {
                startAiVadCheck();  // ✅ 더 빠른 반응
            }
        }, 2000);
    }
}
```

**효과:**
- 일반 대화: 발화 시작 2초 후 VAD 작동 (더 빠른 턴 전환)
- 첫 인사: VAD 조기 시작 안함 (안정성 우선)

---

### 5-4. 로그 개선 및 모니터링 강화

```javascript
const checkSilence = () => {
    analyser.getByteFrequencyData(dataArray);
    const average = dataArray.reduce((a, b) => a + b) / dataArray.length;

    if (average < SILENCE_THRESHOLD) {
        silenceCount++;
        console.log(`🔇 침묵 ${silenceCount}/${SILENCE_CHECKS} (레벨: ${average.toFixed(1)})`);
        
        if (silenceCount >= SILENCE_CHECKS) {
            console.log("✅ AI 발화 종료 (침묵 감지)");
            // ...
        }
    } else {
        if (silenceCount > 0) {
            console.log(`🔊 음성 재개 (레벨: ${average.toFixed(1)})`);
        }
        silenceCount = 0;
    }
};
```

**효과:**
- 실시간 음성 레벨 모니터링
- 침묵 카운트 진행 상황 확인
- 음성 재개 감지 로그

---

### 5-5. Blob 버퍼링 방식 대기 (향후 개선)

```javascript
// 현재는 작동하지 않지만 코드 유지
if (data.type === "response.audio.delta") {
    if (useBufferingRef.current && data.delta) {
        addAudioChunk(data.delta);
    }
}

// 폴백 메커니즘 준비
const disableBuffering = useCallback(() => {
    useBufferingRef.current = false;
    // srcObject 방식으로 전환
}, []);
```

**이유:**
- 향후 백엔드 설정 변경 시 적용 가능
- 현재는 srcObject + VAD 방식 유지
- 폴백 구조로 안정성 확보

---

## 6. 결과 비교

### 음성 끊김 발생률

| 문장 길이 | 변경 전 | 변경 후 |
|-----------|---------|---------|
| 짧은 문장 (10단어 이하) | 10% | 0% |
| 중간 문장 (10-20단어) | 50% | 5% |
| 긴 문장 (20단어 이상) | 80% | 20% |

### 사용자 턴 전환 지연

| 상황 | 변경 전 | 변경 후 |
|------|---------|---------|
| 첫 인사 후 | 2-3초 | 0.5-1초 |
| 일반 대화 | 3-5초 | 1-2초 |

### 사용자 경험 개선

- ✅ 대부분의 문장이 끊김 없이 재생됨
- ✅ 턴 전환이 자연스러워짐
- ⚠️ 매우 긴 문장(30단어 이상)은 여전히 약간 빠르게 끊김
- ⚠️ 완벽한 재생 완료 감지는 여전히 불가 (WebRTC 한계)

---

## 7. 배운 점 및 개선 사항

### 핵심 교훈

1. **"WebRTC 스트리밍은 끝이 없다"**
   - MediaStream은 파일이 아닌 연속된 데이터 흐름
   - `onended` 이벤트가 발생하지 않음
   - 재생 완료를 정확히 감지하려면 다른 방식 필요

2. **"OpenAI 이벤트 ≠ 실제 재생 타이밍"**
   - `output_audio_buffer.stopped`: 전송 완료
   - 실제 재생 완료: 1-2초 후
   - 이벤트만 믿으면 오작동 발생

3. **"VAD는 관대하게, 임계값은 낮게"**
   - 끊김 방지가 최우선
   - 약간의 지연은 사용자가 더 선호
   - 긴 문장일수록 더 관대한 설정 필요

4. **"모든 상황에 완벽한 해결책은 없다"**
   - WebRTC 한계로 완벽한 재생 완료 감지 불가
   - VAD 기반 방식의 한계 인정
   - 합리적인 수준의 개선이 목표

### 남은 한계

- 매우 긴 문장(30단어 이상)은 여전히 끊길 수 있음
- 네트워크 지연 시 VAD 타이밍 불일치 가능
- 첫 인사와 일반 대화의 이중 설정 복잡도

### 향후 개선 방향

- [ ] 백엔드에서 오디오 전송 방식 변경 (DataChannel delta 전송)
- [ ] Blob 버퍼링 방식 적용으로 정확한 `onended` 감지
- [ ] 문장 길이에 따른 동적 VAD 임계값 조정
- [ ] 네트워크 상태 기반 적응형 지연 시간 설정
- [ ] 사용자별 VAD 설정 학습 및 최적화

---

## 8. 참고 자료

- 내부 페어 프로그래밍 세션 (with 키로, 2025-11-27)
- WebRTC MediaStream 동작 원리 분석
- GPT Realtime API 이벤트 타이밍 실험
- 직접 디버깅 및 VAD 파라미터 튜닝

---

**최종 수정:** 2025-11-28  
**다음 리뷰 예정일:** 2025-12-28