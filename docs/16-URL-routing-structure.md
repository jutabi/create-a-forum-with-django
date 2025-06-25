## 16. URL 라우팅 구조
이제 댓글 기능을 프로젝트에 추가할 예정입니다.
다만 그 전에 URL 패턴 (라우팅 구조)의 종류에 대해 잠깐 알아보고 갑시다.


### Nested
>>/posts/\<int:post_id>/comments/create/
/posts/\<int:post_id>/comments/
/posts/\<int:post_id>/comments/\<int:comment_id>/update/
/posts/\<int:post_id>/comments/\<int:comment_id>/delete/

#### 장점
1. URL을 보면 댓글과 게시물의 종속성을 한눈에 알아볼 수 있습니다.
2. 종속 관계가 명확히 보여 유지보수, 사용자 경험에 유리합니다.
3. RESTful 설계 원칙에 부합합니다. 리소스 간의 관계를 명확히 표현합니다.
    - RESTful URL 규칙에 '리소스 중심의 URI'
        URL 는 리소스의 위치를 명확히 식별해야 하며 동사가 아닌 명사를 사용한다.

#### 단점
1. 중첩이 깊어질 수록 URL이 길어지고 복잡해 집니다.
2. 댓글의 pk만으로는 접근할 수 없고 상위 리소스(Post)의 id도 같이 전송되어야 합니다.

##### 관계(종속)를 명확히 보여주고 싶다면:
댓글이 반드시 게시물에 종속되어 있고, 독립적으로 존재하지 않는다면(models.CASCADE) Nested구조가 적합합니다.

---
### Flat 
>>/comment/create
/comment/
/comment/\<int:comment_id>/update/
/comment/\<int:comment_id>/delete/

#### 장점
1. URL이 짧고 간결해 관리와 사용이 쉽습니다.
2. 댓글의 pk만으로 CRUD 요청이 가능합니다.
3. 여러 엔티티 (게시물, 작성자)에서 댓글을 참조할 때 일관적입니다.
    - >>/meber/\<int:m_id>/comments/\<int:c_id>
        /posts/\<int:p_id>/comments/\<int:c_id> (X)
        /comments/\<int:c_id> (O)

#### 단점

1. URL 만으로는 어떠한 종속관계가 있는지 파악할 수 없습니다.
2. 유지보수, 사용자 경험 측면에서 불리합니다.

##### 여러 엔티티에 종속, 혹은 독립적인 엔티티:
댓글이 여러 엔티티에 속할 수 있거나, 독립적으로 자주 다루어진다면 Flat구조가 적합합니다.
동일 리소스에 하나의 URL만 가질 수 있도록 주의

---

### shallow
>>/posts/\<int:post_id>/comments/
/posts/\<int:post_id>/comments/create/
/comments/\<int:comment_id>/update/
/comments/\<int:comment_id>/delete/

#### 조회와 생성은 Nested, 수정과 삭제는 Flat
두 구조를 혼합하여 사용하는 하이브리드 방식

##### 게시글 detail 내에서 댓글 리스트를 불러오고, 작성하는 것은 Nested
- 게시글의 댓글 목록 조회와 신규 댓글 작성은 '***게시글에 종속되는 댓글들을 불러오거나 게시물에 종속하는 새로운 댓글을 생성한다.***' 라는 의미가 강합니다.

##### detail 내에서나 사용자 프로필 페이지의 '작성한 댓글들' 에서 수정 삭제할 때는 Flat
- 그러나 수정과 삭제는 물론 게시물에 종속 되어있는 댓글에 대한 처리이지만 '어떠한 pk값의 게시물의 어떠한 pk값을 가지는 댓글을 수정, 삭제한다.' 의 느낌 보다는 그저 ***'어떤 pk값을 가지는 댓글을 수정 삭제한다.'*** 라는 의미가 강합니다.


위의 이유처럼 URL의 의미와 종속으로 인한 불필요한 url구조를 해결할 수 있다는 장점이 있습니다.

---

### 패스 파라미터(URL 패턴) / 쿼리 파라미터(GET 쿼리스트링)
urls.py 에서 작성한 URL패턴이 '/posts/\<int:post_id>/comments/'라고 하면,

##### ex. Ajax 요청시 패스 파라미터와 쿼리 파라미터
```javascript
$.ajax(
    url: '/posts/123/comments/',
    data: {
        page: 2,
    }
)
// 실제 요청 값: /posts/123/comments/?page=2
```

#### 패스 파라미터 (필수 요소)
/posts/***123***/comments/는 패스 파라미터에 해당합니다.
URL 자체에 포함되어 있어 ***특정 리소스를 고유하게 식별***할 때 사용합니다. 위의 예제는 특정 게시물의 pk를 사용합니다.

뷰에서 접근은 self.kwargs.get('post_pk') 혹은 전달인자 function(reqeust, pk)로 전달받아 사용합니다.

#### 쿼리 파라미터 (선택 요소)
.../comments/***?page=2***는 쿼리 파라미터에 해당합니다.
URL의 끝에 ?key=value의 쌍으로 작성되며 ***필터링, 정렬, 페이징 등 부가적인 옵션***을 전달할 때 사용합니다.
/?key=value&key=value의 형태로 여러 쌍을 전달할 수 있습니다.

뷰에서 접근은 request.GET.get('page')로 전달받아 사용합니다.


##### 패스 파라미터로 페이징 구현
간혹 페이징을 패스 파라미터로 (/posts/page/2)구현하는 사이트들이 존재합니다. 이런 방식이 완전히 잘못된 것은 아니지만 REST 설계 원칙과 웹 표준에는 권장되지 않는 방법입니다.

쿼리파라미터로 전달해야 검색엔진의 크롤링이 원활하여 SEO(Search Engine Optimization)측면에서도 유리합니다.
페이지 번호가 패스 파라미터로 전달되면 URL의 의미가 모호해져 유지보수와 확장성 면에서 불리함을 가집니다.