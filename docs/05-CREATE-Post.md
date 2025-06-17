## 05. CREATE Post (+Django Form)

### 1. POST 요청을 이용해보자
지금까지 진행했던 post_list와 post_detail은 'GET' HTTP Method를 이용했다.
이번엔 'POST' HTTP Method를 사용하여 사용자가 데이터를 서버에 전송해보자.

#### 파일 작성
##### post_form.html
게시물을 작성할 수 있는 페이지(템플릿)을 생성한다. (/templates/forum/post_form.html)
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <!-- html form태그를 사용하고 method로 "POST"를 선택한다. -->
    <!-- form 태그의 요청 주소(action)을 이전 템플릿들 처럼 url 태그를 사용해 forum:post_create로 설정한다. -->
    <form method="POST" action="{% url 'forum:post_create' %}">
        <!-- Django 템플릿의 보안 기본기능이다. POST method form 요청은 csrf_token을 가지고 있어야 한다. -->
        {% csrf_token %}

        <div class="title">
        <label>Title
            <!-- 서버로 전송할 데이터 입력 필드를 생성한다. name=""속성의 값이 요청받은 서버에서 request.***로 접근할 수 있는 값이 된다. -->
             <!-- 나중에는 서버에서도 처리하겠지만 일단 값을 비워놓고 제출하지 못하도록 required 속성을 추가한다. -->
            <input type="text" name="title" placeholder="title" required>
        </label>
        </div>

        <div class="content">
        <label>Content
            <textarea name="content" placeHolder="content" required></textarea>
        </label>
        </div>
        <!-- submit input, button 태그를 통해 폼 태그의 action 주소에 요청을 한다. -->
        <input type="submit" value="제출">
    </form>
</body>
</html>
```

##### CSRF
CSRF(Cross-Site Request Forgery 사이트간 요청 위조)는 해커가 사용자의 권한을 도용하여 의도하지 않은 요청을 보내는 공격 기법.
우리의 웹사이트를 예시로 들어보면. 해커는 제목, 내용 필드를 사용자가 의도하지 않은 내용으로 채워 넣고 그 요청 주소를 사용자가 클릭하도록 유도합니다. 
사용자는 우리의 웹사이트에 로그인된 상태에서 (새탭 열기 포함(세션에 로그인 정보가 있다면 전부))그 버튼, 이미지, 링크를 클릭할 시 해커가 설정한 주소로 요청을 보내게 되고 담겨있는 내용이 게시판에 작성됩니다. 이는 사용자가 직접 요청한 것과 같기 때문입니다.

다른 예시를 들어보면 우리의 비밀번호 변경 url이 /member/password_change/라고 하면 해커는 폼 태그에 원하는 비밀번호를 미리 입력해두고 action또한 /member/password_change/로 바꾼 뒤 접속시 요청을 보내는 사이트를 만들어 놓습니다. 
그리고 사용자가 이 사이트에 접속하면 로그인 된 사용자가 자신의 비밀번호를 변경하는 요청을 서버에 보냈으니 서버는 비밀번호를 변경하고 이것을 통해 해커는 사용자의 계정 비밀번호를 자신이 원하는 값으로 변경시킬 수 있습니다.

이를 막기 위해서 csrf 토큰을 사용하는 데 사용자가 form태그가 있는 url에 접속하면 사용자의 쿠키와 form에 csrf 토큰을 발행합니다. 그리고 서버는 어떠한 POST 요청이 온다면 그 토큰 값이 서로 일치하는지 확인한 후 요청에 대한 처리를 합니다.
이 값은 html의 hidden 값으로 설정되어 있습니다. 
![스크린샷](/statics/05_06.png)
![스크린샷](/statics/05_07.png)
둘의 value가 다른 이유는 \<input>csrf 태그는 값이 마스킹 처리되어 있기 때문입니다.
쿠키의 csrftoken은 서버가 생성한 원본 토큰입니다. 브라우저에 저장되어 클라이언트와 서버가 값을 공유합니다.
폼의(input)csrfmiddlewaretoken은 원본 토큰을 서버가 마스킹 작업을 하여 변형한 값입니다. 
이 프로젝트에서 나중에는 이 input태그의 값을 페이지 내에서 활용할 예정인데 그렇다면 악의적인 스크립트 또한 이 값을 이용할 수 있습니다. 그렇기에 서버는 마스킹한 값을 input태그에 저장하고 폼이 제출될 때 이 값을 원래의 값으로 복원하여 쿠키의 csrftoken과 비교합니다.

##### forum/urls.py
```python
urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('<int:pk>/', views.post_detail, name='post_detail'),
    # posts/new/ 요청을 post 생성 view 함수로 지정한다.
    path('new/', views.post_create, name='post_create'),
]
```

##### forum/views.py
```python
from django.shortcuts import render, get_object_or_404, redirect
from forum.models import *

def post_list(request):
    ...

def post_detail(request, pk):
    ...

def post_create(request):
    # GET요청과 POST요청으로 분기가 이루어진다.
    # GET요청시 '/posts/new/'의 게시물 작성 페이지(템플릿)를 사용자에게 렌더해주기 위함이고
    # POST요청시 'posts/new/'페이지의 <form>태그에서 POST요청이 들어왔을 때 사용자가 입력한 데이터를 저장한다.

    # 만약 어떤 페이지(ex. post_list에서 글 작성 버튼을 눌렀다면)
    # 서버에 POST요청으로 어떠한 데이터를 전달한 것이 아닌 단순한 url 이동(/posts/new/ GET 요청)이기 때문에 
    # 신규 post를 작성할 수 있는 템플릿을 사용자에게 렌더해준다.
    if request.method == 'GET':
        return render(request, 'forum/post_form.html')
    # 여기서는 이미 사용자가 '/posts/new/'페이지에 접속한 상태에서 html의 <form>태그를 통해 입력데이터와 
    # 같이 보낸 POST 요청을 처리한다.
    elif request.method == 'POST':
        # Post 객체를 생성하는 방법은 이미 Django shell을 이용할 떄 배웠다.
        post = Post(
            # POST 요청의 ['title']접근으로 폼 태그의 <input name='title'>값에 접근할 수 있다.
            title=request.POST['title'],
            content=request.POST['content'],
            # created_date는 모델을 선언할 때 자동으로 모델이 생성될 떄 삽입될 수 있도록 해두었다.
        )
        # 아직 데이터베이스에 저장된 것이 아니다. 생성한 post객체를 .save()를 통해 데이터베이스에 저장한다.
        post.save()
        # redirect 메서드는 템플릿 태그때 배웠던 url 태그를 사용할 수 있다.
        # 게시물 생성이 완료되었다면 사용자를 본인이 작성한 게시물의 상세 페이지로 이동시킨다.
        return redirect('forum:post_detail', pk=post.pk)
```

##### post_list
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <h2>글 목록</h2>
    <hr>
    {% if posts %}
        {% for post in posts %}
            <li>
            <a href="{% url 'forum:post_detail' pk=post.pk %}">{{ post.title }}</a>
            </li>
        {% endfor %}
    {% endif %}
    <hr>
    <!-- 메인 페이지 (post_list)에 글 작성 링크를 삽입한다. -->
    <a href="{% url 'forum:post_create' %}">글 작성</a>
</body>
</html>
```

#### 결과 확인
![스크린샷](/statics/05/05_01.png)
![스크린샷](/statics/05/05_02.png)
```
[16/Jun/2025 14:25:17] "GET /posts/new/ HTTP/1.1" 200 677
[16/Jun/2025 14:25:29] "POST /posts/new/ HTTP/1.1" 302 0
[16/Jun/2025 14:25:29] "GET /posts/8/ HTTP/1.1" 200 229
```
서버 메세지와 화면에서 정상적으로 데이터가 삽입된 것을 볼 수 있다.

##### +form action
우리가 작성한 form 태그에는 action속성을 통해 어느 주소로 POST요청을 보낼지를 명시했다. 하지만 생략이 가능하다.
```html
<form method="POST">
        {% csrf_token %}
        ...
        <input type="submit" value="제출">
    </form>
```
위의 코드처럼 action 속성을 삭제해도 정상적으로 게시물이 작성되는 것을 확인할 수 있는데 그 이유는 form태그는 현재 접속한 url에 요청을 보내기 때문이다. 
GET 요청을 통해 /posts/new/에 들어왔고 그에따라 form은 /posts/new/에 POST 요청을 보낸다.
게시물 수정기능을 구현할 때 템플릿을 재사용하기 위해 생략한 채로 사용.

---

### 2. Django Form
위의 실습에서는 post = Post() 객체를 선언하여 데이터를 저장하는 법을 배웠다.
다만 이 방법은 객체의 속성값이 많거나 오류 처리(Null값 전송, 특수문자, 쿼리문 입력 등등)가 있을 떄는 사용하기 불편해진다.
그래서 Django에서는 'Form'기능을 제공한다.

#### Django Form 생성
forum/forms.py 파일을 생성하고 아래의 내용을 작성하자.
```python
from django import forms
from forum.models import *

# 명명 규칙은 사용할모델Form의 파스칼케이스로 작성한다.
class PostForm(forms.ModelForm):
    # 필수 클래스. 사용할 모델과 해당 모델의 속성을 정의
    class Meta:
        model = Post
        fields = ['title', 'content']
```

#### forum/views.py 수정
```python
from django.shortcuts import render, get_object_or_404, redirect

# PostForm 불러오기
from forum.forms import PostForm
from forum.models import *

...

def post_create(request):
    if request.method == 'GET':
        return render(request, 'forum/post_form.html')
    elif request.method == 'POST':
        # post = Post(
        #     title=request.POST['title'],
        #     content=request.POST['content'],
        # )
        # post.save()

        # 이전의 코드는 request.POST['title'], ['content']처럼 각자 저장해야 했다면
        # Django Form은 선언했던 fields = ['title', 'content']의 이름에 자동으로 매칭하여 객체를 생성해준다.
        form = PostForm(request.POST)
        # is_valid()는 폼 객체 내의 값이 오류가 있을 경우 그 오류를 form 객체에 삽입해 준다. (유효성 검사)
        if form.is_valid():
            # 만약 유효한 값이 전달되었다면:
            # 추가 작업이 필요한 경우 commit=False 폼에 데이터를 임시 저장할 수 있다. 
                # (post = form.save(commit=False), ..., post.save())
            post = form.save()
            # 저장이 완료되었음으로 사용자를 해당 게시글 페이지로 이동시킨다.
            return redirect('forum:post_detail', pk=post.pk)
        # 만약 폼의 값에 문제가 있다면 위의 save와 redirect가 실행되지 않고
        # 해당 폼에 오류 메세지를 담은 채 사용자자에게 GET 요청때 처럼 post_form.html 을 다시 렌더해준다.
        return render(request, 'forum/post_form.html', {'form': form})
```

#### post_form 수정
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <form method="POST">
        {% csrf_token %}
        <!-- context 로 전달받은 form 객체에 오류가 포함되어 있다면: -->
        {% if form.errors %}
            <div class="error-message">
            <!-- form의 필드 각각['title', 'content']에 접근하여 -->
            {% for field in form %}
            <!-- 그 필드에 에러가 있다면 -->
            {% if field.errors %}
                <!-- 라벨(Django Form 에서 설정가능. 기본적으는 필드 이름 파스칼케이스)과 에러 메세지를 화면에 출력한다. -->
                <strong>{{ field.label }}</strong>
                {{ field.errors }}
            {% endif %}
            {% endfor %}
            </div>
        {% endif %}
        <div class="title">
        <label>Title
            <!-- 유효한 값이 전달되지 않아 사용자가 다시 입력해야 하는 경우 기존의 입력 데이터를 사용자에게 다시 전달하기 위하여 value값을 설정합니다. -->
            <input type="text" name="title" required value="{{ form.title.value }}">
        </label>
        </div>
        <div class="content">
        <label>Content
            <textarea name="content" required>{{ form.content.value }}</textarea>
        </label>
        </div>
        <input type="submit" value="제출">
    </form>
</body>
</html>
```

#### 결과 확인
개발자 도구에 들어가 input, textarea의 required 속성을 지우고 제출버튼을 눌러보자.
![스크린샷](/statics/05/05_03.png) 
![스크린샷](/statics/05/05_04.png) 
오류메세지가 정상적으로 출력된다.

값을 입력한 뒤 제출하여도 정상적으로 게시물 작성이 완료된다.
![스크린샷](/statics/05/05_05.png) 