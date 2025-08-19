## 마이페이지 정보 조회

- **url:** `GET /api/mypage`
- **request header:** 사용자 인증 토큰   
{
  "Content-Type": "application/json",
  "Authorization": "Bearer <Token>"
}
- **설명:** 사용자의 마이페이지 전체 정보(프로필, 활동 통계, 참여한 모임 정보)를 조회합니다.

### 응답(Response) 예시

#### 성공 (200 OK)
```
json
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
#### 에러 
* **401 Unauthorized Error**: 인증 토큰이 없거나 유효하지 않음
* **404 USER_NOT_FOUND**: 사용자를 찾을 수 없음

---

## 마이페이지 프로필 정보 수정

- **url:** `PUT /api/mypage/profile`
- **request header:** 
  - 사용자 인증 토큰   
  {
    "Content-Type": "multipart/form-data",
    "Authorization": "Bearer <토큰값>"
  }
  - content-type: multipart/form-data
- **request body:**
  - profileImage: 새로운 프로필 이미지 파일 (선택 사항)
  - UpdateProfileRequestDto   
    - username: 새로운 닉네임 (선택 사항)
    - introduction: 새로운 자기소개 (선택 사항)
    - preferredGenres: 선호 장르 수정(선택 사항)
- **설명:** 사용자의 프로필 정보를 수정합니다. 이미지와 텍스트 정보를 함께 처리할 수 있습니다.

#### 성공 (200 OK)
```
json
{
  "status": true,
  "message": "마이페이지 정보 조회를 성공했습니다.",
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
* **401 Unauthorized Error**: 인증 토큰이 없거나 유효하지 않음