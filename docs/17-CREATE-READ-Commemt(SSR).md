## 17. 댓글 생성과 조회 (서버사이드 렌더링)
드디어 post_detail에서 댓글의 생성과 조회 입니다.
이번 차시에서는 전통적인 SSR방식을 구현해 보고 다음 차시에서 비동기 통신을 이용한 구현을 진행할 예정입니다.

### SSR
먼저 SSR에 대해 알아보겠습니다. 우리 프로젝트에서 Post의 CRUD작업이 이에 해당합니다.
서버에서 사용자에게 보여줄 html의 내용을 완성하여 전달하고 클라이언트는 그 html을 사용자에게 출력해주는 방식을 서버사이드 렌더링이라고 합니다.

댓글 생성을 예로 들면 사용자가 웹페이지에서 댓글 작성 폼(\<form>)을 전송하면 서버는 데이터베이스에 댓글을 생성한 뒤, 게시물의 내용과 댓글들의 내용이 전부 포함된 html을 사용자에게 전달(return render())해 줍니다. 

그렇다 보니 댓글을 생성하면 새로운 페이지를 요청하게 됩니다(페이지 이동). 이로 인한 문제점이 여러가지 있는데
- 웹페이지 내용 전부를 전달받다 보니 서버의 트래픽에 부담을 줍니다.
- 댓글이 생성된 페이지(post_detail)에서 사용자가 뒤로가기 키를 눌러도 이전의 글 목록 페이지로 돌아가는 것이 아닌 댓글 생성 하기 전 페이지(post_list)로 돌아가게 되어 글 목록으로 돌아가려면 뒤로가기 키를 두번 눌러야 하는 상황이 발생합니다.

이러한 현상을 방지하기 위해 비동기 처리를 이용한 클라이언트 사이드 렌더링을 구현합니다.
다만 이러한 문제점들을 체험해보고 비동기 처리를 익히기 위해 전통적인 서버사이드 렌더링 기법으로 댓글 생성,조회를 실습해 봅시다.

---

### 서버 측 구현

#### forum/models.py
Comment 모델을 생성해 봅시다.
```python
from django.contrib.auth.models import User
from django.db import models
from django.utils import timezone

from member.models import ForumUser

class Post(models.Model):
    ...

class Comment(models.Model):
    author = models.ForeignKey(ForumUser, on_delete=models.CASCADE)
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    content = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)
```
makemigrations / migrate 진행해 줍니다.

#### forum/forms.py
urls.py에서 사용할 Django Form을 작성합시다.
```python
from django import forms
from forum.models import Post, Comment

class PostForm(forms.ModelForm):
    ...
        
class CommentForm(forms.ModelForm):
    class Meta:
        model = Comment
        fields = ['content']
```

#### forum/urls.py
이전 차시에서 배운 shallow 구조를 사용하여 구현합니다.
```python
urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('new/', views.post_create, name='post_create'),
    path('<int:post_pk>/', views.post_detail, name='post_detail'),
    path('<int:post_pk>/update/', views.post_update, name='post_update'),
    path('<int:post_pk>/delete/', views.post_delete, name='post_delete'),

    path('<int:post_pk>/comments/create/', views.comment_create, name='comment_create'),
]
```

#### forum/views.py
```python
def comment_create(request, post_pk):
    if request.method == 'POST':
        post = get_object_or_404(Post, pk=post_pk)
        form = CommentForm(request.POST)
        if form.is_valid():
            comment = form.save(commit=False)
            comment.author = request.user
            comment.post = post
            comment.save()
            return redirect('forum:post_detail', post_pk=post_pk)
        # 폼의 값이 잘못되었다면 form에는 에러 내용과 기존 입력 값을 넣어 전달하고,
        # post의 내용을 출력하기 위해 post객체도 전달합니다.
        context = {
            'form': form,
            'post': post,
        }
        return render(request, 'forum/post_detail.html', context)
    # 허용되는 HTTP 메서드를 HTTP response Allow 헤더에 추가해 줍니다.
    # 리스트의 형태로 ['GET', 'POST']의 형태도 가능합니다.
    return HttpResponseNotAllowed(['POST'])

```
댓글 생성 html \<form>은 post_detail에 존재하기 때문에 GET요청을 받을 일이 없어 request.method 검사가 필요하지 않습니다.
하지만 예외상황을 위해 POST요청인지 묻고 아니라면 NotAllowed(405) 에러를 반환해 줍니다.

---

### 클라이언트 측 구현

#### post_detail.html
```html
{% extends 'base.html' %}
{% block title %}
Forum: {{ post.title }}
{% endblock title %}

{% block body %}
<div class="container">
    <h2 class="border-bottom py-2">{{ post.title }}</h2>
    <div class="card border-secondary">
        ...
    </div>
    <div class="row">
        ...
    </div>
    <hr>
    <div class="comments">
        {% for comment in post.comment_set.all %}
        <div class="card mb-3">
            <div class="card-header">
                {{ comment.author }}
            </div>
            <div class="card-body">
                <!-- Post의 content처럼 들여쓰기와 개행을 표시하기 위해 pre-wrap스타일을 적용합니다.-->
                <div class="card-text" style="white-space: pre-wrap">{{ comment.content }}</div>
            </div>
            <div class="card-footer" style="font-size: small">
                <div>작성일: {{ comment.created_date|date:"Y/m/d A h:i" }}</div>
                {% with temp=comment.updated_date|date:"Y/m/d A h:i" %}
                    <div>수정일: {{ temp|default:"수정 사항 없음" }}</div>
                {% endwith %}
            </div>
        </div>
        {% endfor %}
    </div>

    <form action="{% url 'forum:comment_create' post.pk %}" method="POST" class="my-4">
        {% csrf_token %}
        {% include 'form_errors.html' %}
        <div class="my-2">
            <label for="comment-content" class="form-label h4">댓글 작성</label>
            <!--로그인 하지 않은 사용자라면 text-area클릭 시 로그인 페이지 안내를 진행합니다.-->
            {% if not request.user.is_authenticated %}
                <!--클릭 시 로그인 페이지로 이동하기 위해 클래스명을 추가합니다 (redirect-login).-->
                <textarea id="comment-content" name="content" class="form-control redirect-login"
                          style="min-height: 10vh">로그인이 필요합니다.</textarea>
            {% else %}
                <textarea id="comment-content" name="content" class="form-control" required
                          style="min-height: 10vh"></textarea>
            {% endif %}

        </div>
        <div class="text-end">
            <!--로그인 하지 않은 사용자라면 disabled 속성을 통햇 버튼의 동작을 제한합니다.-->
            <input type="submit" class="btn btn-primary" value="제출"
                    {% if not request.user.is_authenticated %} disabled {% endif %}>
        </div>
    </form>
</div>

<script type="text/javascript">
    // js에서 url 템플릿 태그를 사용하기 위해
    const loginUrl = "{% url 'member:login' %}";
</script>
{% load static %}
<script src="{% static 'js/post_detail.js' %}?{% now 'U' %}"></script>
{% endblock body %}
```

#### post_detail.js
```javascript
...
// textarea만 redirect-login 클래스명을 가지고 있지만 게시글, 댓글 추천에도 사용하기 위해 SelectorAll을 사용합니다.
const redirectLogin = document.querySelectorAll('.redirect-login');
redirectLogin.forEach(function(element){
    element.addEventListener('click', function() {
        if (confirm("로그인 페이지로 이동하시겠습니까?")) {
            // location.href = "/member/login/?next=" + location.pathname;
            location.href = loginUrl + "?next=" + location.pathname;
        }
    })
})
```

#### post_list.html
```html
{% extends 'base.html' %}
{% block title %}
Forum
{% endblock title %}

{% block body %}
<div class="container">
    <table class="table table-striped table-hover">
        <thead>
            ...
        </thead>
        <tbody class="table-group-divider">
            {% if posts %}
            {% for post in posts %}
            <tr>
                <td class="text-center">{{ post.pk }}</td>
                <td class="text-center">
                    <!--게시물 리스트에서 댓글 수를 표시해 줍시다.-->
                    <a href="{% url 'forum:post_detail' post.pk %}">{{ post.title }}</a> [{{ post.comment_set.count }}]
                </td>
                <td class="text-center">{{ post.author }}</td>
                <td class="text-center">{{ post.created_date|date:"Y/m/d A h:i" }}</td>
            </tr>
            {% endfor %}
            {% endif %}
        </tbody>
    </table>

    ...
</div>
{% endblock body %}
```

---

### 결과 확인
![스크린샷](/statics/17/17_01.png)
![스크린샷](/statics/17/17_02.png)
![스크린샷](/statics/17/17_03.png)
![스크린샷](/statics/17/17_04.png)
![스크린샷](/statics/17/17_05.png)