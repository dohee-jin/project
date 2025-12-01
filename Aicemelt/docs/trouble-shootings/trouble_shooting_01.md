# 🔧 트러블슈팅: AI 스몰토크 주제 중복 생성 및 임베딩 오류 문제

본 문서는 Tropical 백엔드 개발 과정에서 발생한  
**“AI 스몰토크 주제 추천 중복 문제”**와  
**“OpenAI Embedding 적용 시 타입 오류 및 벡터 불일치 문제”**를 분석하고 해결한 기록입니다.

**작성자:** 진도희  
**작성일:** 2025-11-27  
**문서 버전:** v1.0

---

## 1. 문제 현상

### 1-1. AI가 유사한 주제를 계속 생성함
- threshold(0.9) 기준이 너무 높아 **모든 주제가 ‘유사함’으로 판정**
- DB 저장 개수 0개 발생
- 로그 예시:
  ```
  유사한 주제 발견 - 저장 안함: "주말에 뭐 했나요?" (유사도: 0.91)
  ```

### 1-2. Spring AI Embedding 타입 오류
- 문서에는 `List<Double>` 로 되어 있으나 실제 타입은 `float[]`
- 컴파일 오류 발생:
  ```
  Incompatible types. Found: 'float[]', required: 'List<Double>'
  ```

### 1-3. 임베딩–주제 매핑 인덱스 mismatch
- topicContent 중 빈 문자열이 존재
- 임베딩 개수와 주제 개수 불일치
- 예외:
  ```
  IndexOutOfBoundsException
  ```

---

## 2. 원인 분석

### 2-1. LLM이 생성한 embedding은 “진짜 임베딩 아님”
- AI가 응답 JSON에서 제공하는 `"embedding": [0.3, ...]` 값은  
  **의미 기반 벡터가 아니라 단순 숫자 배열**
- 실제 OpenAI Embedding 모델로 재계산 필요

### 2-2. Spring AI 1.0.0-M5 Embedding 타입 변경
- 최신 버전: `Embedding.getOutput()` → `float[]`
- 문서와 실제가 달라 변환 에러 발생

### 2-3. threshold 문제
- 짧은 문장은 의미와 관계없이 코사인 값이 0.9 이상으로 나옴
- 모든 주제가 “유사” 처리되는 부작용

### 2-4. topicContent 전처리 부족
- 공백/빈 문자열이 필터링되지 않음
- 임베딩 개수 mismatch 발생

---

## 3. 재현 단계

1. DB에 기존 스몰토크 10개 이상 저장  
2. 새 주제 15개 요청  
3. threshold = 0.9 유지  
4. AI가 생성한 주제 15개 중 15개 모두 "유사함" 처리  
5. 임베딩 타입 오류 추가 발생  

---

## 4. 디버깅 과정

### 4-1. Embedding 타입 확인

```java
System.out.println(embedding.getOutput().getClass());
```

출력:
```
class [F   // float[]
```

### 4-2. 모든 주제가 유사 처리되는 원인 파악
- L2 normalization 미적용
- threshold 값이 과도함

### 4-3. 인덱스 mismatch 확인
```
responses: 15
topicTexts after filtering: 12
```

---

## 5. 해결 과정

### 5-1. 임베딩: AI 응답의 임베딩 사용 중단 → **OpenAI Embedding 재계산**

```java
List<double[]> embeddings = batchEmbed(topicTexts);
```

---

### 5-2. batchEmbed 구현

```java
private List<double[]> batchEmbed(List<String> texts) {
    if (texts == null || texts.isEmpty()) return List.of();

    List<double[]> out = new ArrayList<>(texts.size());
    int batchSize = 64;

    for (int i = 0; i < texts.size(); i += batchSize) {
        int end = Math.min(i + batchSize, texts.size());
        List<String> slice = texts.subList(i, end);

        EmbeddingResponse res = embeddingModel.embedForResponse(slice);
        List<Embedding> results = res.getResults();

        for (Embedding e : results) {
            double[] v = toDoubleArray(e);
            out.add(l2normalize(v));
        }
    }
    return out;
}
```

---

### 5-3. float[] → double[] 변환

```java
private double[] toDoubleArray(Embedding e) {
    float[] floats = e.getOutput();
    double[] arr = new double[floats.length];
    for (int i = 0; i < floats.length; i++) {
        arr[i] = floats[i];
    }
    return arr;
}
```

---

### 5-4. L2 normalization 적용

```java
private double[] l2normalize(double[] v) {
    double sum = 0;
    for (double x : v) sum += x * x;
    double norm = Math.sqrt(sum);
    if (norm == 0) return v;

    double[] out = new double[v.length];
    for (int i = 0; i < v.length; i++) out[i] = v[i] / norm;
    return out;
}
```

---

## 6. 결론 및 배운 점

### 해결된 핵심 문제
- LLM-생성 임베딩 제거  
- 실제 OpenAI 임베딩 기반 중복처리 완성  
- float[] ↔ double[] 타입 오류 해결  
- 인덱스 mismatch 해결  
- 유사도 필터의 안정성과 정확도 확보  

### 배운 점
- “LLM이 만든 임베딩 ≠ 진짜 임베딩”  
- Spring AI Embedding 타입 확인 필수  
- 코사인 유사도는 L2 normalization이 필수  
- 임베딩은 단순 기술이 아니라 ‘데이터 품질’의 핵심이다  

