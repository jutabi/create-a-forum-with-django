## 14. Post 모델에 author 속성 추가

[참고 문헌: 점프 투 장고 / 3-07 모델 변경](https://wikidocs.net/71306)

### author 속성 추가와 코드 연계 수정

#### forum/models.py
외래키 속성인 author 를 Post 모델에 추가합시다.
```python
from django.contrib.auth.models import User
from django.db import models
from django.utils import timezone

from member.models import ForumUser

class Post(models.Model):
    # 외래키 등록 메서드를 통해 커스텀 모델을 등록해 줍니다.
    # on_delete=models.CASCADE 메서드를 통해 외래키로 참조하는 작성자의 데이터가 없어지면 (회원탈퇴)
    # 작성한 Post(들)도 같이 삭제됩니다.
    author = models.ForeignKey(ForumUser, on_delete=models.CASCADE)

    title = models.CharField(max_length=100)
    ...
```
모델에 속성이 추가되었으니 마이그레이션을 진행하기 전 이전에 봤던 forumuser 테이블의 내용을 확인하고 진행합니다.
![스크린샷](/statics/13/13_06.png)

##### 마이그레이션 진행
```
(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py makemigrations
It is impossible to add a non-nullable field 'author' to post without specifying a default. This is because the database needs something to populate existing rows.
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows with a null value for this column)
 2) Quit and manually define a default value in models.py.
Select an option:
```
뭔가 오류가 있어보이는 코드이지만 해석해보면 Post모델에 새로운 속성이 생성되었는데 기존의 Post들 (이전에 테스트로 집어넣은 300개의 게시물들)의 author 속성에 어떤 값을 집어 넣을 것인지를 물어보는 것입니다. 1번 옵션을 선택하여 바로 author 속성값을 설정해 줍시다.
```
Select an option: 1
Please enter the default value as valid Python.
The datetime and django.utils.timezone modules are available, so it is possible to provide e.g. timezone.now as a value.
Type 'exit' to exit this prompt
>>> 3
```
이번에는 어떠한 값을 넣을지를 물어봅니다. 위의 커스텀 멤버 테이블에서 원하는 사용자의 id를 입력해 줍시다.
저는 pk(id) 3의 qwer1234 사용자를 등록했습니다.
```
Migrations for 'forum':
  forum\migrations\0003_comment_author_post_author.py
    + Add field author to post

(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, forum, member, sessions
Running migrations:
  Applying forum.0003_post_author... OK
```

#### forum/views.py
수정할 것이 하나 더 있습니다. 뷰에서 데이터베이스에 저장시 객체에 작성자를 입력해 주어야 합니다.
```python
def post_create(request):
    if request.method == 'GET':
        return render(request, 'forum/post_form.html')
    elif request.method == 'POST':
        form = PostForm(request.POST)
        if form.is_valid():
            # commit=False 옵션으로 임시저장
            post = form.save(commit=False)
            # 현재 로그인 한 사용자(request.user)를 삽입해 준다.
            post.author = request.user
            post.save()
            return redirect('forum:post_detail', pk=post.pk)
        return render(request, 'forum/post_form.html', {'form': form})
```

#### post_list, post_detail에 글쓴이 표시
##### post_list.html
```html
{% extends 'base.html' %}
{% block title %}
Forum
{% endblock title %}

{% block body %}
<div class="container">
    <table class="table table-striped table-hover">
        <thead>
            <tr class="table-secondary text-center">
                <th style="width: 6%">글번호</th>
                <th style="width: 50%">제목</th>
                <th style="width: 10%">작성자</th>
                <th style="width: 15%">작성일</th>
            </tr>
        </thead>
        <tbody class="table-group-divider">
            {% if posts %}
            {% for post in posts %}
            <tr>
                <td class="text-center">{{ post.pk }}</td>
                <td class="text-center">
                    <a href="{% url 'forum:post_detail' post.pk %}">{{ post.title }}</a>
                </td>
                <td class="text-center">{{ post.author }}</td>
                <td class="text-center">{{ post.created_date|date:"Y/m/d A h:i" }}</td>
            </tr>
            {% endfor %}
            {% endif %}
        </tbody>
    </table>
    ...
```
기존의 코드에서 테이블 헤더의 width 값을 조정하고, 테이블 데이터들과 테이블 헤더에 text-center 클래스를 삽입해(부트스트랩) 중앙 정렬을 진행했습니다.

![스크린샷](/statics/14/14_01.png)

##### post_detail.html
```html
{% extends 'base.html' %}
{% block title %}
Forum: {{ post.title }}
{% endblock title %}

{% block body %}
<div class="container">
    <h2 class="border-bottom py-2">{{ post.title }}</h2>
    <div class="card border-secondary">
        <!--카드 헤더 클래스(부트스트랩)을 통해 디자인을 합니다.-->
        <div class="card-header">
            <div class="card-text">
                {{ post.author }}
            </div>
        </div>
        <div class="card-body" style="min-height: 30vh">
            ...
        </div>
        <div class="card-footer">
            ...
        </div>
    </div>
    <div class="row">
        ...
    </div>
</div>
```
![스크린샷](/statics/14/14_02.png)

#### 게시물 작성해보기
기존 포스트들의 작성자로 등록한 사용자가 아닌 다른 사용자 계정으로 접속하여 게시물을 작성해 봅시다.
![스크린샷](/statics/14/14_03.png)

---

### 비로그인 사용자에 대한 접근 제한
일단 로그인 한 사용자에 대한 게시물 작성은 확인되었습니다. 그런데 비로그인한 사용자에 대한 게시물 작성 페이지 접근 제한이 이루어지지 않아서 페이지에 접근이 가능합니다.

한번 비로그인 상태에서 게시물을 작성해 봅시다.
![스크린샷](/statics/14/14_04.png)
ValueError가 발생하는데 우리가 Post를 create할 때 post.author 값에 request.author 값을 넣어주는데 이때 로그인이 되어있지 않다면 AnonymousUser object를 반환해 주기 때문에 오류가 발생하는 것입니다.

이를 해결하기 위해 FBV의 어노테이션 기능을 사용합니다.
```python
from django.contrib.auth.decorators import login_required
...
@login_required
def post_create(request):
    if request.method == 'GET':
        return render(request, 'forum/post_form.html')
    elif request.method == 'POST':
        ...
```
많은 어노테이션중 login_required를 사용합니다. 우리가 직접 뷰 함수 내에 로그인 하지 않은 사용자에 대한 처리를 하지 않고 이 어노테이션 선언 하나만으로 로그인 하지 않은 사용자에 대한 접근 제한과 로그인 페이지 안내를 해결할 수 있습니다.

어노테이션은 일반적인 메서드처럼 속성을 사용할 수 있습니다.
```python
@login_required(login_url='member:login')
def post_create(request):
    ...
```
우리는 settings.py 에서 LOGIN_URL을 지정해 주었기 때문에 login_url 속성을 지정하지 않아도 자동으로 Django의 컨벤션인 /accounts/login/이 아닌 우리 프로젝트의 /member/login/ 으로 이동됩니다.

이제 비로그인 상태에서 게시물 작성 페이지로 접근해 봅시다.
![스크린샷](/statics/14/14_05.png)
자동으로 로그인 페이지로 이동되며 ?next 라는 쿼리 파라미터가 작성되어 있는 것을 확인하실 수 있습니다.
이는 로그인이 완료되면 next의 값 (/posts/new/)로 이동시켜 주겠다는 뜻입니다.
#### 주의
우리는 로그인 뷰를 직접 구현하지 않고 Django.contrib.auth.views의 LoginView()를 사용하고 있습니다. 
하지만 직접 구현시에는 next를 사용하기 위해 login.html 템플릿에 한가지 설정을 해주어야 하는 경우가 있습니다.
```html
{% extends 'base.html' %}
{% block title %}
Login
{% endblock title %}

{% block body %}
<div class="container">
    {% include 'form_errors.html' %}
</div>
<div class="container justify-content-center" style="display: flex">
    <form method="POST">
        {% csrf_token %}
        <!--next input 태그 필요-->
        <input type="hidden" name="next" value="{{ next }}">
        <div class="form-group">
            ...
        </div>
        ...
    </form>
</div>
{% endblock body %}
```
![스크린샷](/statics/14/14_08.png)
위의 코드와 스크린샷을 참고하면 csrf토큰과 같이 hidden 타입의 next라는 name을 가지는 input태그를 확인하실 수 있습니다.
어노테이션은 쿼리스트링에도 next 값을 적어주고 POST 요청을 위한 {{ next }}값도 전달해 줍니다.

로그인 뷰를 직접 구현했다면 개발자는 next에 대한 코드(next 값이 존재한다면 그 페이지로 리다이렉트)를 작성해야 하는데 이 태그를 사용해서 POST 데이터로 전달받거나, GET 쿼리 파라미터로 next 값을 받아서 처리해야 합니다.
{둘 다 지원해도 좋습니다.( if request.GET.get['next'] or if request.POST.get['next'] )}
(코드를 작성하지 않는다면 당연하게도 기존의 LOGIN_REDIRECT_URL에 해당하는 주소로 이동하게 됩니다.)

하지만 Django의 LoginView()는 GET 전달, POST 전달 두 방식 모두 대응해 주기 때문에 우리는 저 input 태그가 없어도 쿼리 파라미터의 next값 만으로 정상작동 합니다.


#### 결과 확인
![스크린샷](/statics/14/14_06.png)
로그인을 완료되면 자동으로 위 스크린샷처럼 이전에 요청했던 페이지{게시물 작성 (/posts/new/)}로 이동됩니다.

![스크린샷](/statics/14/14_07.png)
게시물도 정상적으로 작성되는 것을 확인하실 수 있습니다.