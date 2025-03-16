# 2-gray-chang-community-be
kakaotech bootcamp community be(SpringBoot)
## 회고
커뮤니티 제작의 프론트엔드에 이어 이번에는 서버를 제작했다.
### 1. 테이블을 이상하게 설계하면 나중에 고생한다.
후술하겠지만, 엔티티 이름을 변경한 죄로 엄청 고생했다.
jpa의 편리함으로 초기에 엔티티 설정과 동시에 테이블이 설정되었지만, 이름을 변경하니까 기존 테이블명이 변경되는 대신 새로운 테이블이 생성되는 문제가 발생했다.<br>
단순한 테이블이면 문제가 되지 않지만, 문제는 이 테이블의 pk를 바라보고 있는 다른 테이블이 있어 한개의 외래키만 갖던 테이블이 두개의 외래키를 갖게 되는 상황이 발생한 것이다.<br>
db를 초기화하고 jpa를 재실행하는 것으로 해결했지만, 초기 테이블 설계의 중요성을 다시 깨달은 순간이었다.

### 2. 아직은 백엔드가 재미있다.
프론트엔드와는 확실히 다른 부분이 있다. 프론트엔드는 html의 요소에 값들을 잘 넣어주는 역할이라면 백엔드는 실제 서비스 로직을 실행시켜 결과값을 전달하는 역할로 이해가 된다.<br>
요소에 일일히 값들을 설정해주는 부분보다는 논리를 구현하고 값들을 전달하는 부분이 더 매력적으로 느껴지고 더 재미있다.

## 과제 해결하면서 발생한 문제, 해결방법
### 문제 1
1. 세션 생성은 성공적이고 클라이언트로 전달됨(로그인 메서드 로그에서 확인)
2. 그러나 세션 검증api에서 자꾸 null을 반환하는 현상 발생

=> 생각한거
   1. 처음에는 ```HttpServletRequest```를 파라미터로 받아 서버 인메모리를 사용하려고 했음
    
      ```java
       @GetMapping("/auth/session-user")
       public ResponseEntity<?> getSessionUser(HttpServletRequest req) {
           HttpSession session = req.getSession(false);
           //session 있는지 검증로직
           Object userId = session.getAttribute("userId");
           // userId가 있는지 검증하는 로직
           logger.info("세션 유지 중 - 유저 ID: {}", userId);
           return ResponseEntity.ok().body(userId);
       }
      ```
   2. 로그 확인 결과 다음과 같은 결과가 나오는 것을 확인
       ```
      세션은 유효한데 userId 없음 : 7
      ```
   3. 그러니까 세션은 클라이언트가 서버로 잘 전달을 해주는데, 이를 매치하지 못하는 것으로 추정했다.
   4. userId를 검증하는 로직은 아래와 같다.
       ```java
      Object userId = session.getAttribute("userId");
       ```
   5. 세션에서 userId를 조회하지 못해 null을 반환하는 상황이었다.
   6. 당장 세션 발급 방식을 JWT방식으로 변경할 수도 있었지만, 세션기반 로그인의 본질에 대해 조금 더 집중했다.
       * 로그인 시 사용자에게 세션을 발급하고, 사용자는 인가가 필요한 모든 로직에 세션을 제출하면 그걸 **세션스토리지** 에서 비교하고 확인하여 접근 여부를 결정한다.
   7. userId를 세션에 포함하여 발급하면 향후 유저아이디가 노출된다는 단점이 있어, 세션발급에 userId를 추가하는 방식은 제외했다.
   8. 대신 서버에서 세션스토리지를 생성하여 거기에서 sessionId와 userId를 매핑하는 것으로 결정했다.
       * 지금은 권한에 따른 별도 인가 과정이 없다. 그래서 sessionId-userId 매핑만으로 가능하지만, 만약 권한이 세분화된다면 새로운 방식을 고민해야된다.(Redis 도입?)\
   9. sessionId는 uuid로 생성하여 128비트 16진수 난수로 생성하였다.
       * 혹시 세션아이디가 겹치지 않을까? 하는 만약의 불안감에 sessionId + userId를 sha256으로 해싱하여 세션아이디로 리턴하는 방식을 고려했으나,
           * 단순 세션 발급에 보안 알고리즘이 들어가버리면 최초 세션 발급 시(로그인 시) 동시접속 유저가 많아지면 db점유시간이 길어져 병목현상이 발생할거라고 판단했다.
           * 그리고 어차피 기존 uuid 기반 발급이 의미없는 난수 생성이므로 보안성, 식별성에 해싱 도입은 큰 이점이 없다고 생각했다.

### 문제 2
1. 최초 엔티티 명명 시 게시글 엔티티를 ```post```라고 지었다가 http method명과 혼돈할 수 있겠다고 판단하고 ```posts```로 변경했다.
2. 그런데 jpa에서 엔티티에 맞추어 자동 생성해준 ```post``` 테이블과 ```posts```테이블이 중복되어 삭제 문제가 발생했다.
   * 구체적으로, posts의 데이터를 삭제하려면 외래키로 의존하고 있는 comment, like 데이터를 삭제해야 하는데, comment, like는 post를 바라보고 있기 때문에 삭제가 되지 않는 문제가 발생했다.
=> 해결법
   1. 다행히 개발중이어서 테스트 데이터가 많지 않아 db를 초기화하고 jpa가 테이블을 다시 만들도록 했다.
      * 만약 데이터가 많아서 초기화가 어려웠다면(끔찍하지만) DDL로 수동 조작하는 방법이 있었다.
        * like, comment 테이블의 post foreign key를 찾아서 이를 삭제
        * posts의 id를 참조하는 새로운 foreign key를 추가
        * post 테이블 완전 삭제
   2. 명명법에 대해서는 "나중에 명명하는거 바꿔도 되지 않나?"하고 가볍게 생각하고 있었는데 이번 경험을 통해 정확한 명명법도 중요하다는 것을 깨달았다.