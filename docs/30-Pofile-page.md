## 30. 프로필 페이지

### 서버 측 작업
#### urls\.py
```python
urlpatterns = [
    path('login/',                 views.custom_login,              name='login'),
    path('logout/',                auth_views.LogoutView.as_view(), name='logout'),
    path('signup/',                views.signup,                    name='signup'),

    path('profile/<int:user_pk>/', views.profile,                   name='profile'),
    ...
]
```

#### member/views.py
```python
def profile(request, user_pk):
    user = get_object_or_404(ForumUser, pk=user_pk)
    posts = user.posts.all().order_by('-created_date')
    # 뷰 단에서 슬라이싱을 이용해 최근 5개의 게시물과 댓글을 보여줍니다. 슬라이싱은 실제로 DB에 LIMIT을 적용합니다.
    # 그리고 우리는 여러 메서드를 체이닝 하는데 annotate를 사용해 추천 수 카운트를 하기 때문에
    # 그 전에 슬라이싱을 진행하여 DB가 모든 row에 대한 계산을 하지 않도록 합니다.
    context = {
        'profile_user': user,
        'posts': (user.posts.all()
                  .order_by('-created_date')[:5]
                  .annotate(voted_users_count=Count('voted_users'))
                  ),
        'comments': (user.comments.all()
                     .order_by('-created_date')[:5]
                     .annotate(voted_users_count=Count('voted_users'))
        )
    }

    return render(request, 'profile.html', context)
```

---

### 클라이언트 측 작업

#### profile.html
post_list에서 사용했던 테이블을 재사용 해 봅시다.
```html
{% extends 'base.html' %}
{% block title %}
{{ profile_user.nickname }}'s Profile
{% endblock title %}

{% block body %}
<div class="container mt-3">
    <div class="card">
        <div class="card-header">
            <h4>{{ profile_user.nickname }}</h4>
            <p>가입일: {{ profile_user.date_joined|date:"Y/m/d A f" }}</p>
        </div>
        <div class="card-body">
            <h5>최근 작성한 게시물</h5>
            <table class="table table-hover">
                <thead>
                    <tr class="table-secondary text-center">
                        <th style="width: 5%">추천</th>
                        <th style="width: 50%">제목</th>
                        <th style="width: 15%">작성일</th>
                    </tr>
                </thead>
                <tbody class="table-group-divider">
                    {% if posts %}
                    {% for post in posts %}
                    <tr>
                        <td class="text-center">{{ post.voted_users_count }}</td>
                        <td class="text-center">
                            <a href="{% url 'forum:post_detail' post.pk %}">{{ post.title }}</a> [{{ post.comment_set.count }}]
                        </td>
                        <td class="text-center">{{ post.created_date|date:"Y/m/d A h:i" }}</td>
                    </tr>
                    {% endfor %}
                    {% endif %}
                </tbody>
            </table>
            <hr>
            <h5>최근 작성한 댓글</h5>
            <table class="table table-hover">
                <thead>
                    <tr class="table-secondary text-center">
                        <th style="width: 5%">추천</th>
                        <th style="width: 50%">내용</th>
                        <th style="width: 15%">작성일</th>
                    </tr>
                </thead>
                <tbody class="table-group-divider">
                    {% if comments %}
                    {% for comment in comments %}
                    <tr>
                        <td class="text-center">{{ comment.voted_users_count }}</td>
                        <td class="text-center">
                            <a href="{% url 'forum:post_detail' comment.post.pk %}">{{ comment.content|slice:":100" }}</a>
                        </td>
                        <td class="text-center">{{ comment.created_date|date:"Y/m/d A h:i" }}</td>
                    </tr>
                    {% endfor %}
                    {% endif %}
                </tbody>
            </table>
        </div>
    </div>
</div>
{% endblock body %}
```

#### navbar.html
```html
<nav class="navbar navbar-expand-lg bg-body-tertiary">
    <div class="container-fluid">
        ...
        <div class="collapse navbar-collapse" id="navbarSupportedContent">
            <ul class="navbar-nav me-auto mb-2 mb-lg-0">
            {% if request.user.is_authenticated %}
                <li class="nav-item dropdown">
                    <button class="btn dropdown-toggle" data-bs-toggle="dropdown" aria-expanded="false">
                        {{ request.user.nickname }}
                    </button>
                    <ul class="dropdown-menu">
                        <!-- 드롭다운 메뉴에 프로필 링크를 추가합니다. -->
                        <li>
                            <a class="dropdown-item" href="{% url 'member:profile' request.user.pk %}">프로필</a>
                        </li>
                        ...
                    </ul>
                </li>
            {% else %}
                ...
            {% endif %}
            </ul>
        </div>
    </div>
</nav>
```

#### post_list.html
```html
<table class="table table-striped table-hover">
    <thead>
        ...
    </thead>
    <tbody class="table-group-divider">
        {% if posts %}
        {% for post in posts %}
        <tr>
            ...
            <td class="text-center">
                <!-- 작성자의 닉네임에 프로필 페이지로 이동하는 링크를 추가합니다. -->
                <a href="{% url 'member:profile' post.author.pk %}">{{ post.author.nickname }}</a>
            </td>
            ...
        </tr>
        {% endfor %}
        {% endif %}
    </tbody>
</table>
```

#### post_detail.html
```html
<div class="container">
    <h2 class="border-bottom py-2">{{ post.title }}</h2>
    <div class="card border-secondary">
        <div class="card-header">
            <div class="card-text">
                <!-- 작성자의 닉네임에 프로필 페이지로 이동하는 링크를 추가합니다. -->
                <a href="{% url 'member:profile' post.author.pk %}">{{ post.author.nickname }}</a>
            </div>
        </div>
    ...
    <template id="comment-template">
        <div class="card mb-3">
            <div class="card-header">
                <div class="row">
                    <div class="col author">
                        <!-- js에서 댓글의 작성자에게 링크를 추가하기 위해 미리 a태그를 생성해 둡시다. -->
                        <a></a>
                    </div>
```

#### post_detail.js
```js
function renderComments(comments, requestPage, lastPage) {
    const commentsDiv = document.querySelector('.comments');
    commentsDiv.innerHTML = '';
    comments.forEach(comment => {
        const cardDiv = document.querySelector('#comment-template').content.cloneNode(true);
        cardDiv.firstElementChild.id = `comment-${comment['pk']}`;

        const nicknameA = cardDiv.querySelector('.card-header .author a')
        nicknameA.append(`${comment['nickname']}`);
        nicknameA.href = `/member/profile/${comment['author_pk']}`
```

---

### 결과 확인
![스크린샷](/statics/30/30_01.png)
![스크린샷](/statics/30/30_02.png)
![스크린샷](/statics/30/30_03.png)