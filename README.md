# Lots-of-minifb
Lots of minifb implementation

동일한 백엔드 어플리케이션(조그마한 페이스북 clone)에 대해, 기술스택을 바꿔가며 수많은 구현체를 만들어 나가는 프로젝트([realworld](https://github.com/gothinkster/realworld)처럼)

## Implementations

## 전제조건
- 별도의 인프라 없이, Amazon EC2와 같은 compute cloud에서 자체적으로 구성한다고 가정 : ex) 파일 관리를 위해 S3를 별도로 사용하지 않음
- 모든 API는 테스트되어야 함(view function에 한해서는, 되도록 테스트 커버리지를 100%로 유지)
- 테스트 프레임워크와 서드파티(CI, Coverage, ...)들은 기술스택의 경우의 수에서 제외(Flask + SQLAlchemy + pytest와 Flask + SQLAlchemy + unittest를 별도의 스택으로 생각하지 않음). 따라서 취향 따라 선택하되, 다양성은 존재해야 함
- 가이드라인에 맞추어 API만 제공된다면, 내부 로직은 자유롭게 구성(동일한 API에 대해 프로젝트 A는 group by를 쓰고, 프로젝트 B는 distinct 쿼리를 쓰더라도 자유성을 인정). 하지만 단지 API를 찍어내는 형태가 아닌, 개발 과정에서 자신도 성장해야 하며 나름대로 많이 고민해 주어야 함.(예제들을 보는 입장에서 새로운 아이디어를 얻어갈 수 있도록)
- 기술스택의 범주는 언어 / 프레임워크 / 데이터베이스 / DB 클라이언트 라이브러리 / 캐시(optional) / API 아키텍처(REST, GraphQL, ...) / 사용자 인증
- 부가 기술의 범주는 테스트 프레임워크 / 의존성 관리 / 로깅 / API 문서 / CI / Coverage / Code Qualitifier. API 문서부터는 모두 optional
- 백엔드 엔지니어링적인 측면에서의 고민이 녹아들어 있으면 좋음(싱글 스레드로 동작하는 redis에 대해 실행 시간을 많이 잡아먹는 operation을 하지 않거나, 데이터베이스 부하에 대한 유연성 고려 등)

## Model(Validation 포함)
### user
- ID(string, PK, length 4~50)
- 비밀번호(string, pbkdf2_hamc hashed : like Werkzeug's generate_password_hash, length 10~100)
- 이름(string, length 2~16)
- 별명(string, nullable, length 1~30)
- 소개(string, nullable, length 1~85)
- created_at(timestamp, default now)

### post
- owner(References user)
- 내용(string, length 1~3000)
- created_at(timestamp, default now)

### reaction_post
- 사용자(References user)
- 글(References post)
- 리액션 타입(int, 1~5)
- 수정됨 여부(bool, default false)
- created_at(timestamp, default now)

### comment
- 사용자(References user)
- 글(References post)
- 내용(string, length 1~1000)
- 수정됨 여부(bool, default false)
- created_at(timestamp, default now)

### reaction_comment
- 사용자(References user)
- 댓글(References comment)
- 리액션 타입(int, 1~5)
- created_at(timestamp, default now)

### notification
- 사용자(References user)
- 알림 타입(int, 1~2, 1: 내 글에 댓글을 남김, 2: 내 글에 좋아요를 남김)
- 알림을 일으킨 사용자(References user)
- 연걸된 글(References post)
- 읽음 여부(bool, default false)
- created_at(timestamp, default now)

## API
### Frontend API
#### 계정
- ID 중복체크
  - parameters (id)
  - response code 
    - 200 ID 중복되지 않음
    - 409 ID 중복됨
- 로그인
  - parameters (id, 비밀번호)
  - response code 
    - 200 로그인 성공, JWT Token 발급
    - 401 로그인 실패   
- 회원가입
  - parameters (id, 비밀번호, 이름)
  - response code 
    - 201 회원가입 완성
    - 409 ID 중복됨
- 임시 비밀번호 발급
  - header (Auth: JWT Token)
  - parameters (?)
  - response code 
    - 201 임시 비밀번호 발급 성공
    - 403 권한 없음
- 비밀번호 변경 (POST)
  - header (Auth: JWT Token)
  - parameters (현재 비밀번호, 새 비밀번호)
  - response code 
    - 200 비밀번호 변경 성공
    - 403 틀린 비밀번호 입력 | 권한 없음
    - 409 현재와 새 비밀번호 동일
- 별명 변경 (POST)
  - header (Auth: JWT Token)
  - parameters (새 별명)
  - response code 
    - 200 별명 변경 성공
    - 403 권한 없음
    - 409 현재 별명과 새 별명이 동일
- 소개 변경 (POST)
  - header (Auth: JWT Token)
  - parameters (새 소개)
  - response code 
    - 200 소개 변경 성공
    - 403 권한 없음
    - 409 현재 설명과 새 설명이 동일
- 재량에 따라, 인증 관련 API(JWT token refresh 등)

#### 글
- 게시글 업로드 (POST)
  - header (Auth: JWT Token)
  - parameters (내용)
  - response code 
    - 201 게시글 작성 성공
    - 403 권한 없음
- 게시글 수정 (PATCH)
  - header (Auth: JWT Token)
  - parameters (수정할 글, 새 내용)
  - response code 
    - 200 수정 성공 
    - 403 권한 없음
- 게시글 삭제 (DELETE)
  - header (Auth: JWT Token)
  - parameters (삭제할 게시글)
  - response code
    - 200 게시글 삭제 성공 
    - 403 권한 없음
- 게시글 목록 조회(pagination) (GET)
  - header (Auth: JWT Token)
  - response code 
    - 200 게시글 목록 조회 성공
    - 204 게시물이 하나도 존재하지 않음
    - 403 권한 없음
- 특정 게시글 조회 (GET)
  - header (Auth: JWT Token)
  - response code
    - 200 특정 게시글 조회 성공
    - 204 특정 게시물 존재하지 않음
    - 403 권한 없음

#### 타임라인
- 내 타임라인 (GET)
  - header (Auth: JWT Token)
  - response code
    - 200 타임라인 조회 성공
    - 403 권한 없음
- 다른 사용자의 타임라인 (GET)
  - header (Auth: JWT Token)
  - response code
    - 200 다른 사용자의 타임라인 조회 성공
    - 204 존재하지 않는 사용자

#### 댓글
- 댓글 추가 (POST)
  - header (Auth: JWT Token)
  - parameters (게시글, 내용)
  - response code
    - 201 댓글 추가 성공
    - 400 삭제된 글에 댓글 작성
    - 403 권한 없음
- 댓글 수정 (PATCH)
  - header (Auth: JWT Token)
  - parameters (게시글, 수정할 댓글, 새 내용)
  - response code
    - 200 댓글 수정 성공
    - 403 권한 없음
- 댓글 삭제 (DELETE)
  - header (Auth: JWT Token)
  - parameters (게시글)
  - response code
    - 200 댓글 삭제 성공
    - 403 권한 없음
- 특정 게시글의 댓글 목록 조회(pagination) (GET)
  - header (Auth: JWT Token)
  - parameters (게시글)
  - response code
    - 200 댓글 목록 조회 성공
    - 403 권한 없음

#### 리액션
- 글에 리액션 작성 (POST)
  - header (Auth: JWT Token)
  - parameters (게시글)
  - response code
    - 201 글에 리액션 작성 성공
    - 403 권한 없음
- 글에 리액션 취소 (DELETE)
  - header (Auth: JWT Token)
  - parameters (게시글)
  - response code
    - 200 글에 리액션 삭제 성공
    - 403 권한 없음
- 댓글에 리액션 작성 (POST)
  - header (Auth: JWT Token)
  - parameters (게시글, 댓글)
  - response code
    - 201 댓글에 리액션 작성 성공
    - 403 권한 없음
- 댓글에 리액션 취소 (DELETE)
  - header (Auth: JWT Token)
  - parameters (게시글, 댓글)
  - response code
    - 200 댓글에 리액션 삭제 성공
    - 403 권한 없음
#### 알림
- 현재까지 온 알림 목록 조회 (GET)
  - header (Auth: JWT Token)
  - response code
    - 200 알림 목록 조회 성공
    - 204 알림 없음
    - 403 권한 없음
- 알림 읽음
  - ? 이해 못함

### Back Office API
#### 서버 통계(optional)
InfluxDB+Chronograf나 ElasticSearch+Kibana와 같은 로깅 시스템과의 연동을 하는 게 좋으므로 optional

- rps
- 최근 24시간동안 요청 수
- status code별 count
- API endpoint별 count

#### 서비스 통계
- 총 게시글 수
- 총 사용자 수
- 오늘 올라온 게시글 수

## 에러 핸들링
```
{
  "result": "Forbidden",
  "hint": "You have not permission for 'DELETE /post/14'"
}
```
`result`에 status message를, 'hint'에 문제 상황을 명시. `500 Internal Server Error`의 경우, 에러 메시지(`str(e)`)를 response
