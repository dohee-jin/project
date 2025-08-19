## 리뷰(좋아요) 

- **url:** `GET /api/meetings/{meetingId}/reviews`
- **request header:** 사용자 인증 토큰   
  {
    "Content-Type": "application/json",
    "Authorization": "Bearer <토큰값>"
  }
- **설명:** 사용자의 마이페이지 전체 정보(프로필, 활동 통계, 참여한 모임 정보)를 조회합니다.   

### 요청(Request) 예시
```
json
{ "toUserId": 2 }
```

### 응답(Response) 예시

#### 성공 (200 OK)
```
json
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

