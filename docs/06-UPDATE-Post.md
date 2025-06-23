## 06. UPDATE Post

[참고 문헌: 점프 투 장고 / 3-09 수정과 삭제](https://wikidocs.net/71445)

#### 1. 모델 속성 추가
forum/models.py
```python
class Post(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)

    # null=True => null값 허용 / blank=True => is_valid() 입력 검증 시 값이 없어도 통과
    updated_date = models.DateTimeField(null=True, blank=True)
    ...
```
##### 마이그레이션
```
(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py makemigrations
Migrations for 'forum':
  forum\migrations\0002_post_updated_date.py
    + Add field updated_date to post

(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, forum, sessions
Running migrations:
  Applying forum.0002_post_updated_date... OK

(.venv) C:\Users\***\PycharmProjects\forum-with-django>

```

#### 2. post_detail.html 수정
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <h2>{{ post.title }}</h2>
    <h5>작성일: {{ post.created_date }}</h5>
    <!-- 만약 글이 수정된 적 있다면 수정일 출력 -->
    {% if post.updated_date %}
        <h5>수정일: {{ post.updated_date }}</h5>
    {% endif %}
    <hr>
    <a>{{ post.content }}</a>
    <hr>
    <a href="{% url 'forum:post_update' post.pk %}">수정</a>
    <a href="{% url 'forum:post_list' %}">글 목록</a>
</body>
</html>
```

#### 3. forum/urls.py 수정
```python
from django.urls import path
from forum import views

app_name = 'forum'

urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('new/', views.post_create, name='post_create'),
    path('<int:pk>/', views.post_detail, name='post_detail'),
    path('<int:pk>/update/', views.post_update, name='post_update'),
]
```

#### 4. forum/views.py 수정
```python
def post_update(request, pk):
    post = get_object_or_404(Post, pk=pk)
    if request.method == 'GET':
        # 위에서 불러온 post객체의 값을 폼에 저장하고 그 폼을 컨텍스트로 전달한다. (instance=post)
        # 템플릿 내에서 post, form 두개를 동시에 대응하는 코드를 짜는 것 보다 GET, POST 둘다 form객체를 전달한다.
        form = PostForm(instance=post)
        return render(request, 'forum/post_form.html', {'form': form})
    # 수정 페이지 내에서 '작성'버튼 (POST)요청이 왔을 때
    elif request.method == 'POST':
        # 위에서 불러온 post를 기반하여 폼을 생성하지만 request.POST내의 값으로 덮어 씌운다.
        form = PostForm(request.POST, instance=post)
        if form.is_valid():
            # 수정한 날짜를 넣어 주기 위해 commit=False
            post = form.save(commit=False)
            post.updated_date = timezone.now()
            post.save()
            return redirect('forum:post_detail', pk=post.pk)
        
        return render(request, 'forum/post_form.html', {'form': form})
```

#### 결과 확인
![스크린샷](/statics/06/06_01.png)
![스크린샷](/statics/06/06_02.png)
![스크린샷](/statics/06/06_03.png)