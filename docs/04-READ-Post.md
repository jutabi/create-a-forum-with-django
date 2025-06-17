## 04. READ Post

[참고 문헌: 점프 투 장고 / 2-04 조회와 템플릿](https://wikidocs.net/70736)

### 1. Post list
```
http://localhost:8000/posts/
```
위의 url 요청시 게시물들의 목록을 사용자에게 반환해주자.

#### forum/views.py 작성
```python
from django.shortcuts import render
from forum.models import *

def post_list(request):
    # order_by('-')를 통해 작성일 Desc 순서로 데이터베이스에 저장된 데이터들을 'posts'객체에 저장한다.
    # (최근에 작성된 게시물 순서대로 리스트가 표시될 수 있도록.)
    posts = Post.objects.order_by('-created_date')
    context = {'posts': posts}

    # render는 (request, template, context)를 인자로 받아 사용자에게 페이지를 서버사이드에서 렌더링하여 전달한다.
    # 템플릿은 Django에서 사용하는 html 파일이고, 전달한 context를 html 파일 내에서 접근할 수 있다.
    return render(request,
                  'forum/post_list.html',
                  context)
```

#### forumwithdjango/urls.py 작성
```python
from django.contrib import admin
from django.urls import path

# forum 앱 또한 python 패키지 임으로 views.py를 임포트한다.
from forum import views

urlpatterns = [
    path('admin/', admin.site.urls),
    # 'localhost:8000/posts/'요청이 들어오면 forum의 views.py에서 post_list() 함수를 실행한다.
    # 템플릿 태그(아래에서 언급)를 사용하여 템플릿 내에 url하드코딩을 하지 않도록 namespace를 정한다.
    path('posts/', views.post_list, name='post_list'),
]
```

#### forum/post_list.html 템플릿 작성
위의 forum/views.py 에서 우리는 render 메서드의 인자로 템플릿 파일을 전달했다. 그 템플릿 파일을 생성해보자. 

그전에 템플릿의 저장 위치에 관해 확인할 것이 있다.
##### Django 에서 템플릿이 저장되는 위치
##### 1. 각 앱별 패키지 폴더/templates (forum-with-django/forum/template/***.html)
장점: 
1. 독립적으로 앱별 templates관리가 가능하다.
2. 다른프로젝트에서 재사용 가능한 앱의 경우 템플릿 폴더와 통째로 이동 편의

단점: 
1. 템플릿 이름이 겹치면 충돌 위험
    - 앱마다 templates 폴더에 같은 이름의 템플릿이 있다면 Django는 템플릿을 찾을 때 어떤 앱의 파일을 써야할지 충돌이 생길 수 있다.
    - 기본적으로는 장고 템플릿 로더가 먼저 찾는 (INSTALLED_APPS 명시 우선순위) 템플릿을 사용한다.
2. 전체 템플릿 구조 파악 어려움
##### 2. 프로젝트 최상위 폴더/templates (forum-with-django/templates/forum/***.html)
장점:
1. 프로젝트 내의 모든 템플릿 관리 용이 (유지보수, 협업 시)
2. 공통 템플릿 (ex. base.html 추후 설명) 관리 용이

단점:
1. 앱 재사용시 템플릿 폴더 이동의 불편함함

두 방식 모두 사용되긴 하지만 2번의 프로젝트 최상위 폴더에서 템플릿 파일들을 관리하는 방법이 더 선호된다.
소규모의 프로젝트이거나 재사용을 염두한다면 앱 내부에 두는 방법도 나쁜 방법은 아니다.
다만 프로젝트의 규모가 커지면 커질수록 최상위 폴더의 templates 에서 관리하는 것이 좋은 방법이다.

##### 이제 최상위 폴더에 'templates/forum/'디렉토리를 생성하고 안에 post_list.html 템플릿을 생성해보자.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    {% if posts %}
        {% for post in posts %}
            <li>
            <a href="{% url 'post_detail' pk=post.pk %}">{{ post.title }}</a>
            </li>
        {% endfor %}
    {% endif %}
</body>
</html>
```
Django 만의 새로운 html 파일이 아닌 일반적으로 사용하는 '.html'을 사용하는데 본적 없는 코드들이 눈에 띈다. 
{% %}로 조건문과 반복문이 감싸져 있고 어디선가 post 데이터를 가져와서 사용하는 것처럼 보인다.

Django의 서버(views.py)에서 사용자에게 response로 render메서드의 결과를 리턴하는데 인자로 html파일과 context를 전달한다.
여기에서 render 메서드가 context 와 '템플릿 태그'를 통해 html를 렌더링하여 사용자에게 전달하는 것인데 {% %}가 바로 그 템플릿 태그이다.
```python
return render(request,
              'forum/post_list.html',
              context)
```

##### 템플릿 태그
템플릿 태그의 기본적인 기능 몇가지를 소개합니다.
1. Variables {{ *** }}
위의 코드처럼 render에 인자로 전달한 context의 값을 참조할 수 있습니다. 우리는 posts 쿼리셋을 전달하였음으로 템플릿 에서는 반복문으로 하나의 post 데이터에 접근하고 그 속성인 'title'의 값을 참조할 수 있습니다.

2. Tags {% *** %}
조건문, 반복문 등을 html 내부에서 사용할 수 있도록 합니다. 당연하게도 html에서 연산을 하는 것이 아닌 render 메서드 동작시 연산되어 조건에 해당하는 html 코드 (우리 코드에서는 \<li>\<a>\</a>\</li>)를 html 내부에 삽입해 줍니다.
[빌트인 태그 공식문서](https://docs.djangoproject.com/en/5.1/ref/templates/builtins/#built-in-tag-reference)

3. Filters {{ }}
단독으로 사용할 수는 없으며 variables 와 함께 사용합니다.  {{ value|add:"2" }}, {{ value|capfirst }}, {{ value|date:"D d M Y" }}와 같이 컨텍스트 변수에 간단한 기능연산을 도와주는 기능을 제공합니다. 서버에서 커스텀 필터를 제작하고 사용할 수 있습니다. (ex. 복잡한 사칙연산을 하는 태그, 문자열 재구성 또는 html 태그 삽입 등등)
[빌트인 필터 공식문서](https://docs.djangoproject.com/en/5.1/ref/templates/builtins/#built-in-filter-reference)

##### URL 템플릿 태그
```html
<li>
    <!-- 템플릿 내부에서 아래 주석처럼 url 하드코딩을 해 url을 전달할 수 있다. -->
    <!-- <a href="/posts/{{ post.pk }}">{{ post.title }}</a> -->

    <!-- 다만 그것은 url이 변경되거나 유지보수시 흩어져있는 하드코딩 url을 전부 관리해야 하는 단점이 있다. -->
    <!-- 그렇기 때문에 urls.py에서 라우팅 주소의 별칭을 지정해주고 템플릿 태그를 이용하여 접근하는 방법을 사용한다. -->
    <a href="{% url 'post_detail' pk=post.pk %}">{{ post.title }}</a>
    <!-- url 태그의 사용법: {% url 'views 함수 혹은 클래스 name' 전달 인자 %} -->
    <!-- Post detail의 urls.py 설정시 부연 설명 합니다. -->
</li>
```

##### 작동 확인
![스크린샷](/statics/04/04_01.png)
이전에 관리자 페이지(/admin/)에서 등록한 게시물들의 목록이 표시된다.

---

### 2. Post detail

#### forum/urls.py 작성 (앱별 urls.py 분리)
우리는 post_list (/posts/) 라우팅을 프로젝트의 최상위 urls.py 에서 진행했습니다. 다만 이 방식은 앱별 기능의 분리를 제공하는 Django와는 맞지 않는 방법입니다. 만약 Member앱을 포함한 추가 기능을 가진 앱들이 많아진다면 모든 앱들의 주소 라우팅을 하나의 파일에서 관리하는 사태가 벌어집니다.
그렇기 때문에 urls.py도 앱별 분리가 필요합니다.

forumwithdjango/urls.py 수정
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),

    #위의 admin 설정에서 보이는 것처럼 settings.py의 INSTALLED_APPS에 작성된 forum, admin을 통해 앱 내부의 py에 접근이 가능합니다.
    # /posts/이후의 요청에 대해서는 전부 forum 앱의 urls.py가 담당합니다. (ex. posts/12/delete ...)
    path('posts/', include('forum.urls')),
]
```

forum/에 urls.py 파일을 생성합니다.
```python
from django.urls import path
from forum import views

urlpatterns = [
    # 기존의 설정과 다르게 'posts/'가 작성되있지 않습니다.
    # 위의 전역 urls.py 에서 이미 'posts/'요청에 대해 forum의 urls에서 처리하도록 작성되어 있음으로 개별 엡에서는 작성이 불필요합니다.
    # Pycharm 을 사용하신다면 /posts/ 라는 안내표시가 현재 주석의 위치에 표시되고 있는 것을 확인하실 수 있습니다.
    path('', views.post_list, name='post_list'),

    # /posts/12 처럼 단일 게시물의 상세 정보를 보여주는 페이지에 대한 라우팅
    # url에 posts와 '/'가 이미 전역설정에서 입력되어 있다. 중복되지 않도록 주의 ('/<int:pk>/' (X))
    
    # <a href="{% url 'post_detail' pk=post.pk %}">{{ post.title }}</a>
    # url 별칭을 post_detail로 설정하여 post_list.html에서 작성한 템플릿 태그와 매칭시켜 줍니다.
    path('<int:pk>/', views.post_detail, name='post_detail'),
]
```

#### forum/views.py 내용 추가
```python
from django.shortcuts import render
from forum.models import *

def post_list(request):
    ...

def post_detail(request, pk):
    post = Post.objects.get(pk=pk)
    context = {'post': post}

    return render(request, 'forum/post_detail.html', context)
```

#### post_detail.html 작성
/templates/forum/post_detail.html
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <h1>{{ post.title }}</h1>
    <h4>{{ post.created_date }}</h4>
    <hr>
    <a>{{ post.content }}</a>
</body>
</html>
```

#### 작동 확인
![스크린샷](/statics/04/04_02.png)

#### get_object_or_404
정상 작동을 확인했으나 만약 존재하지 않는 게시물에 사용자가 접근한다면 어떨까
우리의 posts의 마지막 글의 pk(id)값은 7이다. 8이상의 게시물을 요청해보자
![스크린샷](/statics/04/04_03.png)
DoesNotExit 에러를 사용자에게 반환하는데 이는 올바른 반환이 아니다. 흔히 알고있는 404 에러 (요청한 값을 찾을 수 없음)을 반환해 주어야 한다.

DoesNotExit는 '내부적으로 ORM이 데이터베이스에서 해당하는 데이터를 찾지 못했다.' 라는 내용을 '개발자'에게 알려주는 예외이다. 이것을 처리하지 않고 사용자에게 그대로 전달하면 500 Internal Server Error로 응답하게 되는데 사용자의 입장에서는 '서버에 오류가 있다고?'라고 생각할 수 있게 만들기에 404 에러로 변환 후 응답해 주어야 사용자는 비로소 '아 내가 요청한 자료가 존재하지 않는 자료이구나'라고 생각할 수 있다.

#### 위에서 작성한 views.py 수정
```python
# django.shortcuts 에서 render 함수와 같이 get_object_or_404를 불러온다.
from django.shortcuts import render, get_object_or_404
from forum.models import *

def post_list(request):
    ...

def post_detail(request, pk):
    # Post 모델에 해당하는 데이터들 중 pk값이 url패턴(/<int:pk>)에서 전달받은 pk값에 해당하는 데이터를 요청한다.
    # 만약 존재하지 않는다면 404에러를 사용자에게 반환해준다.
    post = get_object_or_404(Post, pk=pk)
    context = {'post': post}

    return render(request, 'forum/post_detail.html', context)
```

#### 작동 확인
![스크린샷](/statics/04/04_04.png)

---

### 3. URL 네임스페이스
우리는 템플릿에 url 템플릿 태그를 이용하여 하드코딩 없이 해당 주소를 입력했고 정상 작동했다. 
다만 여기서 다른 앱이 추가된다면 문제가 생길 여지가 있다. 바로 다른 앱에서도 'post_list'라는 이름의 url name을 사용할 수 있다는 것이다. 
그래서 우리는 URL 네임스페이스라는 것을 사용한다.

forum/urls.py 수정
```python
from django.urls import path
from forum import views

# 앱의 이름을 지정한다. 템플릿 내부에서 사용할 이름으로 앱 이름과 달라도 상관 없으나 권장하지 않는다.
app_name = 'forum'

urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('<int:pk>/', views.post_detail, name='post_detail'),
]
```

/templates/forum/post_list.html 수정
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    {% if posts %}
        {% for post in posts %}
            <li>
            <!-- {% url 앱 네임스페이스 명:'view함수 혹은 클래스 name' 전달 인자 %} -->
            <a href="{% url 'forum:post_detail' pk=post.pk %}">{{ post.title }}</a>
            </li>
        {% endfor %}
    {% endif %}
</body>
</html>
```