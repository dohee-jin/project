# AI 생성 스몰토크 주제 중복 방지 로직 가이드 문서

본 문서는 스몰토크 AI 기능에서 생성된 주제의 **중복 방지 로직**을 정리하여, 팀 전체가 해당 로직을 이해하고 개발 및 유지보수에 활용할 수 있도록 작성되었습니다.

**작성자:** [진도희](https://github.com/dohee-jin)  
**문서 버전:** v1.2  
**최종 수정일:** 2025.09.22  
**대상 독자:** 팀원 전원 (개발자/QA), 팀장/PM

---

## 1. 목적
AI가 생성하는 스몰토크 주제 중 **기존 DB에 저장된 주제와 의미적으로 중복되는 항목을 최소화**하기 위해 설계된 로직을 명확히 정리합니다.  
이를 통해 사용자는 다양한 주제를 제공받고, 반복되는 추천을 피할 수 있습니다.

---

## 2. 기본 원리
1. 기존 저장 주제의 **내용(topicContent)과 예시 질문(exampleQuestion)**를 기반으로 비교  
2. 새로 생성된 주제와 DB에 있는 주제의 **임베딩 벡터**를 생성  
3. **코사인 유사도(cosine similarity)**를 이용하여 의미적 유사성 평가  
4. 유사도가 **임계값(threshold)** 이상이면 저장하지 않고 스킵  

---

## 3. 상세 구현 절차

### 3.1 기존 주제 수집
- 사용자의 `SmalltalkTopic` 리스트를 조회  
- `topicContent`와 `exampleQuestion`을 추출하여 리스트 생성  

### 3.2 AI 요청 후 새 주제 생성
- AI 모델에게 주제 생성을 요청  
- 응답 JSON을 파싱하여 `AISmallTalkResponse` 리스트 생성  

### 3.3 임베딩 벡터 생성
- 새 주제 및 예시 질문을 합쳐 텍스트 벡터화  
- `EmbeddingModel`을 사용해 벡터 생성  
- L2 정규화를 통해 벡터 길이를 1로 조정하여 코사인 유사도 계산 안정화  

#### 3.3.1 임베딩(Embedding) 개념
- 텍스트나 데이터를 **숫자 벡터**로 변환하는 과정  
- 벡터끼리 거리 또는 각도를 비교하여 의미적 유사도를 판단 가능  
- 예시:
  | 주제 | 벡터화 |
  |------|--------|
  | "오늘 날씨 어때?" | [0.12, -0.23, 0.56, …] |
  | "최근 본 영화는?" | [0.05, -0.10, 0.60, …] |

#### 3.3.2 L2 정규화(L2 Normalization)
- 벡터의 **크기(길이)를 1로 만드는 과정**  
- 수식:  
\[
\text{v_normalized} = \frac{v}{||v||_2}, \quad ||v||_2 = \sqrt{\sum_i v_i^2}
\]  
- **필요 이유**:  
  1. 벡터 크기 차이로 인한 유사도 편향 제거  
  2. 코사인 유사도 계산 시 **방향 정보만 사용**하여 의미적 비교 정확도 향상  

---

### 3.4 코사인 유사도(Cosine Similarity)
- 두 벡터 사이의 **각도 기반 유사도 계산**  
- 수식:  
\[
\text{cosine similarity} = \frac{A \cdot B}{||A|| \cdot ||B||}
\]  
- **필요 이유**:  
  1. 벡터 크기에 영향을 받지 않고 의미적 유사도 측정 가능  
  2. 주제 내용이 길거나 짧아도 상대적 의미 비교 가능  
  3. 중복 방지 로직에서 **새 주제와 기존 주제의 의미적 차이 판단**에 핵심  

---

### 3.5 유사도 검사
- DB에 저장된 주제 각각과 새 주제 벡터 간 **코사인 유사도 계산**
- **임계값(threshold)**:  
  - 저장된 주제 < 10개 → 0.85  
  - 저장된 주제 ≥ 10개 → 0.9  
- 유사도 ≥ threshold → **저장 스킵**  

### 3.6 DB 저장
- 유사도 검사를 통과한 주제만 `SmalltalkTopic` 엔티티로 변환 후 저장  
- 최대 저장 개수: 5개  

---

## 4. 기술적 상세

| 항목 | 설명 |
|------|------|
| 임베딩 생성 | AI Embedding 모델 사용, float[] → double[] 변환, L2 정규화 적용 |
| 코사인 유사도 | dot product / (normA * normB), 제로 벡터 처리 포함 |
| JSON 파싱 | `ObjectMapper`를 사용, 예외 발생 시 로그 기록 및 RuntimeException 발생 |
| 예외 처리 | 임베딩 직렬화 실패, JSON 파싱 실패 시 로그 출력 후 예외 발생 |

---

## 5. 참고 자료
- [코사인 유사도](https://en.wikipedia.org/wiki/Cosine_similarity)  
- [L2 정규화](https://en.wikipedia.org/wiki/Norm_(mathematics)#L2_norm)  
- Spring Boot `@Transactional` 문서: https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html  
- ObjectMapper JSON 처리: https://fasterxml.github.io/jackson-databind/javadoc/2.13/

---
