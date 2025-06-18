## 07. DELETE Post

[참고 문헌: 점프 투 장고 / 3-09 수정과 삭제](https://wikidocs.net/71445)

### 1. 글 삭제하기

#### urls.py
```python
urlpatterns = [
    ...
    path('<int:pk>/', views.post_detail, name='post_detail'),
    path('<int:pk>/update/', views.post_update, name='post_update'),
    path('<int:pk>/delete/', views.post_delete, name='post_delete'),
]
```

#### views.py
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
    <!-- href에는 링크라는 것을 나타낼 수 있는 텍스트 데코레이션과 마우스 호버기능을 살리기 위해 -->
    <!-- javascript:void(0) (빈 함수)를 추가해준다. -->
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