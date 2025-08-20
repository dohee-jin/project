# 🗓️ 마이페이지, 리뷰 API 기능 명세

## ✨ 진도희 (마이페이지, 좋아요 기능 담당)

**역할 요약:**
사용자의 정보(개인 정보, 통계 정보, 참여 모임 정보)를 확인하고 수정할 수 있는 기능과   
다른 사용자에게 리뷰(좋아요)를 남기는 기능을 구현합니다.

**주요 구현 기능:**

* 사용자의 프로필 정보 (프로필 사진, 이름, 이메일, 선호장르, 자기소개) 
* 사용자의 활동 통계 정보(참여한 모임 수, 받은 리뷰(좋아요 수)) 
* 내가 참여한 모임 목록 (마이페이지) 
* 종료된 모임에서 다른 사용자에게 리뷰(좋아요) 남기기

**관련 테이블:**

* `meeting`
* `meeting_participant`
* `meeting_review`
* `user`

---

## 1. 마이페이지 API 명세서

### **1. 개요**

사용자의 개인 정보, 통계 정보, 참여 모임 정보를 확인합니다.

---

### **2. 요청(Request)**

* **HTTP Method:** `GET`
- **url:** `/api/mypage`
- **request header:** 사용자 인증 토큰   
{
  "Content-Type": "application/json",
  "Authorization": "Bearer <Token>"
}
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

* **401 UNAUTHORIZED**: 인증 토큰이 없거나 유효하지 않음
* **404 USER_NOT_FOUND**: 사용자를 찾을 수 없음

```json
{
  "timestamp": "2025-08-06T16:00:15.1024654",
  "status": 404,
  "error": "USER_NOT_FOUND",
  "detail": "사용자를 찾을 수 없습니다.",
  "path": "/api/mypage"
}
```

---


## 2. 마이페이지 프로필 정보 수정 API 명세서

### **1. 개요**

사용자의 개인 정보를 조회하고 
이름, 프로필 이미지, 자기소개, 선호장르를 수정할 수 있습니다.

---

### **2. 요청(Request)**

* **HTTP Method:** `PUT`
* **URL:** `/api/mypage/profile`
- **request header:** 사용자 인증 토큰   
{
  "Content-Type": "application/json",
  "Authorization": "Bearer <Token>"
}
* **multipart/form-data:** 

| 이름      | 타입            | 필수 여부 | 설명                      | 예시                                        |
| ------- | ------------- | ----- | ----------------------- | ----------------------------------------- |
| `imageFile` | file          | 선택    | 사용자가 업로드한 이미지 파일        | `profile.png`                             |
| `data`  | JSON (string) | 필수    | 수정된 모임 정보 JSON (DTO 매핑) | `{"username": "새로운닉네임", "description":"매주 토요일"}, ...` |


---

### **3. 응답(Response) 예시**

#### **성공 (200 OK)**


```json
{
  "status": true,
  "message": "마이페이지 정보 수정을 성공했습니다.",
  "timestamp": "2025-08-07T12:12:12.2987169",
  "data": {
    "username": "새로운닉네임",
    "profileImage": "https://example.com/profile/123_new.jpg",
    "introduction": "새로운 자기소개입니다.",
    "preferredGenres": "추리"
  }
}
```

#### 에러 

* **400 INVAILD_INPUT**: 인증 토큰이 없거나 유효하지 않음
* **401 UNAUTHORIZED**: 인증 토큰이 없거나 유효하지 않음

```json
{
  "timestamp": "2025-08-06T16:00:15.1024654",
  "status": 404,
  "error": "USER_NOT_FOUND",
  "detail": "사용자를 찾을 수 없습니다.",
  "path": "/api/mypage/profile"
}
```


---

## 3. 다른 사용자에게 리뷰(좋아요) 남기기 기능 API 명세세

### **1. 개요**

자기 자신을 제외한 사용자에게 하트를 눌러 리류를 남기는 기능입니다.

---

### **2. 요청(Request)**

* **HTTP Method:** `POST`
* **URL:** `/api/meetings/{meetingId}/reviews`
- **request header:** 사용자 인증 토큰   
  {
    "Content-Type": "application/json",
    "Authorization": "Bearer <토큰값>"
  }
* **Query Parameters:**: 없음
* **Request body**: 
```json
{ "toUserId": 2 }
```

---

### **3. 응답(Response) 예시**

#### **성공 (200 OK)**


```json
{
  "status": true,
  "message": "다른 사용자에게 리뷰 남기기를 성공했습니다.",
  "timestamp": "2025-08-07T12:12:12.2987169",
  "data": {
    "reviewId": 33,
    "fromUserId": 1,
    "toUserId": 2,
    "createdAt": "2025-08-20T21:00:00"
    }
}
```

#### 에러 

* **400 MEETING_NOT_COMPLETED**: 모임이 완료되지 않음
* **400 USER_NOT_PARTICIPANT**: 해당 사용자는 모임 참여자가 아님
* **400 LIKE_SELF_NOT_ALLOWED**: 본인에게 좋아요(리뷰)를 남길 수 없음
* **400 DUPLICATE_LIKE**: 좋아요(리뷰)는 최대 한 번만 누를 수 있음
* **401 Unauthorized Error**: 인증 토큰이 없거나 유효하지 않음
* **404 USER_NOT_FOUND**: 사용자를 찾을 수 없음

```json
{
  "timestamp": "2025-08-06T16:00:15.1024654",
  "status": 400,
  "error": "LIKE_SELF_NOT_ALLOWED",
  "detail": "자기 자신을 좋아요할 수 없습니다.",
  "path": "/api/meetings/{meetingId}/reviews"
}
```

---













