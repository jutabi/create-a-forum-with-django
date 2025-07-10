## 07. DELETE Post

[참고 문헌: 점프 투 장고 / 3-09 수정과 삭제](https://wikidocs.net/71445)

### 1. 글 삭제하기

#### urls\.py
```python
urlpatterns = [
    ...
    path('<int:pk>/', views.post_detail, name='post_detail'),
    path('<int:pk>/update/', views.post_update, name='post_update'),
    path('<int:pk>/delete/', views.post_delete, name='post_delete'),
]
```

#### views\.py
```python
from django.shortcuts import render, get_object_or_404, redirect

from forum.forms import PostForm
from forum.models import *

def post_list(request):
    ...

def post_detail(request, pk):
    ...

def post_create(request):
    ...

def post_update(request, pk):
    ...

def post_delete(request, pk):
    post = get_object_or_404(Post, pk=pk)
    post.delete()
    return redirect('forum:post_list')
```

#### post_detail.html
```html
<body>
    <h2>{{ post.title }}</h2>
    <h5>작성일: {{ post.created_date }}</h5>
    {% if post.updated_date %}
        <h5>수정일: {{ post.updated_date }}</h5>
    {% endif %}
    <hr>
    <a>{{ post.content }}</a>
    <hr>
    <a href="{% url 'forum:post_list' %}">글 목록</a>
    <a href="{% url 'forum:post_update' post.pk %}">수정</a>
    <a href="{% url 'forum:post_delete' post.pk %}">삭제</a>
</body>
```
삭제 링크를 클릭하면 게시물의 삭제가 완료됩니다.

---

### 2. 자바스크립트를 이용하여 삭제 전 확인창 띄우기
실수로 삭제 버튼을 클릭했을 때를 대비하여 사용자에게 재확인 모달창을 띄워봅시다.

#### post_detail.html
```html
<body>
    ...
    <a href="{% url 'forum:post_update' post.pk %}">수정</a>

    <!-- 삭제 링크에 data-url속성을 추가하고 기존의 href속성의 값인 삭제링크 url 태그를 입력한다. -->
    <!-- 자바스크립트에서 이 태그에 접근하기 위해 클래스명을 추가한다. -->
    <!-- href에는 클릭 기능(링크라는 것을 나타낼 수 있는 텍스트 데코레이션과 마우스 호버기능)을 살리기 위해 
    javascript:void(0) (빈 함수)를 추가해준다. -->
    <a data-url="{% url 'forum:post_delete' post.pk %}" class="delete-link"
       href="javascript:void(0)">삭제</a>

    <!-- 만약 정말 링크 이동이 필요하지 않고 클릭 이벤트만 필요하다면 button 태그를 쓰는 것이 바람직하다. -->
    <!-- html 태그 내에 스크립트를 삽입하는 것은 (인라인 스크립트) XSS 공격에 노출될 가능성이 있기 때문이다. -->
    <button data-url="{% url 'forum:post_delete' post.pk %}" class="delete-link">삭제</button>

    <!-- body의 맨 마지막에 삭제 링크(버튼)에 클릭 이벤트 리스너를 추가하는 스크립트를 작성한다. -->
    <script type="text/javascript">
        // 쿼리 셀렉터를 통해 "delete-link"클래스명을 가진 엘리먼트를을 불러온다.
        // 현재 삭제 버튼은 Post에만 존재하지만 후에는 댓글 여러개에도 삭제버튼이 추가될 예정이기에 SelectorAll사용
        const deleteLink = document.querySelectorAll('.delete-link');
        // 각각의 엘리먼트들에 함수들을 추가한다.
        deleteLink.forEach(function(element) {
            element.addEventListener('click', function() {
                // 클릭 이벤트가 발생하면 confirm 모달창으로 확인을 묻고
                if(confirm("정말 삭제하시겠습니까?")) {
                    // 사용자가 확인을 눌렀다면 해당 엘리먼트의 data-url 속성 값으로 이동한다. (삭제 요청 url)
                    // 데이터셋의 값을 불러오는 방법은 아래의 두 방법 중 어떤것을 사용해도 무방하다.
                    location.href = element.getAttribute('data-url');
                    // location.href = this.dataset.url;
                }
            })
        })
    </script>
</body>
```
스크립트 (Django 스태틱 스크립트 포함)는 \<body>태그의 맨 밑, \</body> 바로 위에 적는 것이 좋습니다.
HTML DOM 엘리먼트들이 전부 로딩이 되고 난 후 자바스크립트가 해당 값을 읽게 하여 로딩중인 엘리먼트를 참조하다 오류가 나지 않을 수 있도록 합니다.

#### HTML dataset
HTML 엘리먼트에 추가 데이터를 저장하고 싶을 때 사용하는 기능입니다.
예전에는 개발자들이 엘리먼트에 id, class값 이외에 다른 데이터를 추가하고 싶을 때 class명에 넣거나 hidden타입의 input태그를 넣는 등 유지보수 하기 어려운 방식으로 데이터를 첨부했습니다.
그래서 누구나 원하는 이름으로 속성을 만들고 데이터를 추가하기 위해서 HTML5부터 data-속성이 추가되었습니다. 
##### 사용법
저장
```html
<a data-user-id="qwer1234" data-nickname="erik">
```
자바스크립트에서 접근
```javascript
const a = document.querySelector('a');
// 두개의 '-'가 사용된 경우 카멜케이스로 접근한다.
a.dataset.userId
a.dataset.nickname
```

#### 결과 확인
![스크린샷](/statics/07/07_01.png)
![스크린샷](/statics/07/07_02.png)

---

### 3. POST 요청으로 게시물 삭제하기
이제 GET, POST 두 방식 모두 허용되었던 뷰 함수를 POST방식의 요청만 처리할 수 있도록 바꿀 것 입니다.

POST 요청을 사용하는 이유 중 하나는 사용가 의도한 요청만 처리하기 위함 입니다. 단순한 링크 클릭, url 직접 접속이 아닌 폼 제출, Javascript 비동기 제출 등 명확한 사용자의 액션이 있어야만 요청을 하는 것인데요. 게시물 삭제와 같은 서버 내의 데이터를 변경하는 기능은 url 접속만으로 요청이 정상처리되면 안됩니다.


재확인 모달창과 함께 '의도한 요청만 처리'를 강화해 봅시다.

#### views\.py
```python
from django.http import HttpResponseNotAllowed
from django.shortcuts import render, get_object_or_404, redirect

from forum.forms import PostForm
from forum.models import *

def post_list(request):
    ...

def post_detail(request, pk):
    ...

def post_create(request):
    ...

def post_update(request, pk):
    ...

# POST 요청으로 삭제 요청을 처리합니다.
# POST 방식(혹은 REST API에서의 DELETE)을 이용하는게 권장되는 이유
    # 1. 보안성
        # GET 방식은 URL에 모든 정보가 노출됩니다. 
        # 그래서 삭제같은 민감한 작업에서 링크만으로 요청되는 GET방식은 의도치 않은 삭제가 일어날 수 있습니다.
    # 2. HTTP 명세 준수
        # HTTP 표준에서는 데이터를 변경하는 작업(C, U, D)은 POST(PUT, DELETE)를 권장합니다.
        # GET 방식은 오로지 조회, 변경을 할때는 사용하지 않습니다.
    # 3. CSRF 방어
        # POST 요청은 csrf토큰을 사용하여 위조 요청을 막을 수 있습니다.
        # 악의적인 의도를 가진 누군가의 게시물 삭제를 방지할 수 있습니다.
    # 4. 브라우저의 캐싱
        # GET 방식: 브라우저가 캐싱을 적극적으로 사용합니다.
            # GET 요청은 URL에 데이터를 담아 전송합니다. (/posts/123/ or /posts?id=123)
            # 그래서 브라우저는 모든 정보가 URL에 담겨져 있기 때문에 기존의 GET 요청 결과를 캐싱해 둡니다.
            # ('같은 url, 같은 데이터를 요청하는 것이니 다음에 사용할 때도 같은 결과를 보여주겠지.')
            # 브라우저는 위처럼 가정하고 뒤로가기 같은 작업 시 결과물을 재사용하여 같은 요청을 다시 보내지 않습니다.
            
            # 다만 그 사이에 다른 게시물이 삭제되었을 수도 있고 캐시된 데이터와 서버의 데이터가 다를 수 있습니다.
            # 이때 삭제된 게시물이 사용자에게는 삭제되지 않은 것 처럼 보여지고 클릭하여 GET 요청을 보내면 게시물이 
            # 삭제되었다고 서버에서 응답하는 경우가 있습니다.

        # POST 방식: 브라우저가 캐싱을 거의 하지 않습니다.
            # 주로 폼 데이터를 전송할 때 사용되고 데이터가 본문에 담기기 때문에 캐싱하지 않고 매번 새로운 요청을 
            # 보내도록 합니다.
            # 데이터의 상태를 변경하는 작업에 사용되므로, 캐싱이 되면 중복 요청이 발생할 수 있습니다.
            
            # 캐싱을 하지 않는다는 것은 사용자가 POST 요청을 보내고 결과 페이지(render or redirect)에서 뒤로가기를
            # 누르면 브라우저가 그 POST 요청 결과가 아닌 그 이전의 캐싱 되었던 GET 요청 결과를 보여주는 것입니다.
            # /posts/123/ =(삭제 POST요청)=> /posts/123/delete/ =(서버의 redirect응답)=> /posts/
            # 에서 뒤로가기를 하면 삭제된 /posts/123/의 캐싱된 DOM을 출력
            
            # 다만 return render를 응답했을 경우 브라우저는 bfcache(Back/Forward cache)라는 것을 사용할 수 있습니다.
            # post를 삭제하고 return render(post_list)를 응답하면 /posts/***/delete/의 주소로 리스트를 보여줍니다.
            # 화면에는 post_list.html을 출력하고 있지만 실제로 url을 보면 /delete/가 작성되어 있습니다. 
            # 이때 다른 게시물을 요청하고 뒤로가기를 눌러보면 캐싱된 /delete/주소를 가진 post_list가 출력됩니다.
            # 이렇듯 브라우저들 중 bfcache를 사용하는 경우 POST 요청의 렌더 응답 또한 뒤로/앞으로 가기 동작을 위해
            # bfcache에 페이지 전체의 스냅샷을 저장해 놓습니다.
            # /posts/123/ ==> /posts/123/delete/ =(render)=> /posts/123/delete/ => 다른 게시물 요청(클릭)
            # 에서 뒤로가기를 하면 /posts/123/delete/(post_list)의 POST의 결과값이 캐싱되어 있음

            # 이를 방지하려면 render가 아닌 Post Redirect GET(PRG 패턴 / POST 요청의 응답은 GET 리다이렉트)을 
            # 사용하여 삭제 후에는 사용자를 리다이렉트 시켜주면 됩니다.

def post_delete(request, pk):
    post = get_object_or_404(Post, pk=pk)
    if request.method == 'POST':
        post.delete()
        return redirect('forum:post_list')

    # Django의 대표적인 오류 HTTP 응답 메서드
    # HTTPResponseBadRequest  400  잘못된 데이터 전송(파라미터 누락, 타입 오류)
    # HttpResponseForbidden   403  주로 사용자가 접근할 권한이 없을 때 (접근은 가능하나 허가되지 않은 리소스에 접근)
    # HTTPResponseNotFound    404  존재하지 않는 페이지/리소스 요청 시

    # HTTPResponseNotAllowed  405  지원하지 않는 HTTP 메서드 요청 시

    # HTTPResponseGone        410  이전에는 존재했지만 리소스가 영구적으로 삭제됐을 떄
    # HTTPResponseServerError 500  서버 내부 오류 발생 시

    # 만약 POST 방식으로 접근되지 않았다면 (url /posts/***/delete/ 직접 입력 또는 링크)
    return HttpResponseNotAllowed(['POST'])
```

#### post_detail.html
```html
<body>
    ...
    <a href="{% url 'forum:post_update' post.pk %}">수정</a>
    <!-- GET 방식 요청 -->
    <!-- <a href="{% url 'forum:post_delete' post.pk %}">삭제</a> -->
    <!-- POST 요청을 전송하기 위해 form 태그를 생성합니다. -->
    <form action="{% url 'forum:post_delete' post.pk %}" method="POST" class="delete-link">
        {% csrf_token %}
        <input type="submit" value="삭제">
    </form>

    <script type="text/javascript">
        const deleteLinks = document.querySelectorAll('.delete-link');
        deleteLinks.forEach(function(element) {
            // submit(폼 제출) 이벤트가 발생하면
            element.addEventListener('submit', function(event) {
                event.preventDefault();
                // event를 매개변수로 받아와 preventDefault 메서드를 사용합니다.
                // 이 메서드는 해당 엘리먼트의 기본 동작을 제한하는 메서드로 우리 프로젝트의 경우 <form>의 
                // submit 버튼을 클릭 시 submit 요청을 보내지 않도록 합니다.
                
                // confirm이 True값을 반환했을 때만 submit을 하도록 해야 함으로 기본 이벤트를 제한했습니다.
                // 만약 이 메서드를 사용하지 않는다면 사용자가 '아니요(No)'를 선택하여도 <form>태그의 
                // action 속성에 따라 삭제 요청을 서버에 보낼 것 입니다.

                // 위에서 배운 데이터셋과 같이 사용하여 action태그를 제거하고 element.action 속성을 사용해 
                // 이중 안전장치를 마련해도 좋습니다.
                if(confirm("정말 삭제하시겠습니까?")) {
                    // 사용자가 Yes를 선택했다면 submit 메서드를 이용하여 폼을 제출합니다.
                    element.submit();
                }
            })
        })
    </script>
</body>
```
