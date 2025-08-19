# 🗓️ 마이페이지 API 기능 명세

## ✨ 진도희 (마이페이지, 좋아요 기능 담당)

**역할 요약:**
사용자가 모임을 탐색하고 신청하는 기능을 구현합니다.

**주요 구현 기능:**

* 모임 목록 조회 (키워드, 정렬 등 검색 포함) (FR-C-002)
* 모임 상세 조회 (참여자 목록, 게시판 등 포함) (FR-C-003)
* 내가 참여한 모임 목록 (마이페이지) (FR-U-003)

**관련 테이블:**

* `meeting`
* `meeting_participant`
* `meeting_review`
* `user`

---

## 1. 모임 목록 조회 API 명세서

### **1. 개요**

모든 모임의 목록을 조회합니다.
데이터베이스에 저장된 전체 모임 정보를 반환합니다.

---

### **2. 요청(Request)**

* **HTTP Method:** `GET`
* **URL:** `/api/meetings`
* **Query Parameters:** 없음

---

### **3. 응답(Response) 예시**

#### **성공 (200 OK)**


```json
{
  "status": true,
  "message": "마이페이지 정보 조회를 성공했습니다.",
  "timestamp": "2025-08-07T12:12:12.2987169",
  "data": {
    "profile": {
      "username": "책벌레123",
      "email": "user@example.com",
      "profileImage": "https://example.com/profile/123.jpg",
      "introduction": "안녕하세요! 추리소설을 좋아합니다.",
      "preferredGenres": ["추리", "SF", "에세이"]
    },
    "statistics": {
      "receivedLikes": 15,
      "participatedMeetings": 8
    },
    "meetings": [
      {
        "meetingId": 1,
        "title": "추리소설 읽기 모임",
        "date": "2024-08-15T19:00:00Z",
        "status": "completed", // "upcoming", "ongoing", "completed"
        "role": "participant", // "host", "participant"
        "book": {
            "title": "셜록 홈즈",
            "author": "아서 코난 도일"
        }
      },
      {
        "meetingId": 2,
        "title": "SF 소설 토론회",
        "date": "2024-08-20T18:00:00Z",
        "status": "upcoming",
        "role": "host",
        "book": {
            "title": "셜록 홈즈",
            "author": "아서 코난 도일"
        }
      }
    ]
  }
}
```

#### **에러 **

* **401 Unauthorized Error**: 인증 토큰이 없거나 유효하지 않음
* **404 USER_NOT_FOUND**: 사용자를 찾을 수 없음

```json
{
  "timestamp": "2025-08-06T16:00:15.1024654",
  "status": 401,
  "error": "Internal Server Error",
  "detail": "서버 오류가 발생했습니다.",
  "path": "/api/meetings"
}
```

---


## 2. 특정 모임 목록 조회 API 명세서

### **1. 개요**

모임의 목록을 조회합니다.
\*\*지역(location)\*\*과 \*\*상태(status)\*\*를 기준으로 필터링할 수 있습니다.
데이터베이스에 저장된 조건에 맞는 모임 정보를 반환합니다.

---

### **2. 요청(Request)**

* **HTTP Method:** `GET`
* **URL:** `/api/meetings`
* **Query Parameters:**

| 이름         | 타입     | 필수 여부 | 설명                                      | 예시              |
| ---------- | ------ | ----- | --------------------------------------- | --------------- |
| `location` | string | 선택    | 모임 지역 필터링                               | `서울 강남구` |
| `status`   | string | 선택    | 모임 상태 필터링 (`RECRUITING`, `COMPLETED` `CANCELLED`) | `RECRUITING`    |

---

좋아요 👍
그럼 이번에는 **정렬 조건**을 `sort`라는 Query Parameter로 받아서, `마감 임박`, `최신 작성`, `인기` 기준으로 조회하는 API 명세서를 방금과 동일한 형식으로 작성해 드릴게요.

---

## 3. 모임 목록 정렬 및 조회 API 명세서

### **1. 개요**

모든 모임의 목록을 조회합니다.
\*\*정렬 조건(sort)\*\*을 지정하여

* **마감 임박**: `meetingTime`이 가까운 순서
* **최신 작성**: `createdAt`이 가장 최근인 순서
* **인기**: `meeting_review` 테이블에서 `reviewee_id`별 좋아요 수가 많은 순서
  로 정렬할 수 있습니다.
  데이터베이스에 저장된 조건에 맞는 모임 정보를 반환합니다.

---

### **2. 요청(Request)**

* **HTTP Method:** `GET`
* **URL:** `/api/meetings`
* **Query Parameters:**

| 이름         | 타입     | 필수 여부 | 설명                                      | 예시           |
| ---------- | ------ | ----- | --------------------------------------- | ------------ |
| `sort`     | string | 선택    | 정렬 조건 (`deadline`, `latest`, `popular`) | `deadline`   |
| `location` | string | 선택    | 모임 지역 필터링                               | `서울 강남구`     |
| `status`   | string | 선택    | 모임 상태 필터링 (`RECRUITING`, `COMPLETED` 등) | `RECRUITING` |

---













