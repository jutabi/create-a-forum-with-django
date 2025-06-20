## 12. 로그인, 로그아웃

[참고 문헌: 점프 투 장고 / 3-05 로그인과 로그아웃](https://wikidocs.net/71259)

드디어 User 모델을 사용하여 과 로그인, 로그아웃 기능을 구현합니다.

### member django app
우리가 지금까지 작성해온 forum앱에 회원 CRUD 기능을 탑재할 수는 없습니다.
forum은 Post와 Comment의 CRUD를 위한 앱인 것 처럼 User의 CRUD를 담당하는 'member'앱을 생성해 봅시다.

#### 1. 앱 생성
##### I. django-admin startapp *** / python manage.py startapp ***
터미널에 직접 명령어를 입력하여 앱을 생성합니다. 다만 앱을 생성하는 명령어가 두가지 인데
1. django-admin startapp
    - Django가 설치된 환경 어디서는 실행 가능합니다.
    - 프로젝트의 설정을 불러오지 않기 때문에 가상환경이나 여러 프로젝트가 섞여 있으면 원하는 프로젝트에 앱이 만들어지지 않을 수도 있습니다.
    - 여러 프로젝트의 세팅을 바꿔가며 작업할때 사용합니다.
2. python manage.py startapp
    - 현재 Django 프로젝트의 설정을 함께 불러와서 실행합니다.
    - manage.py에서 실행하다 보니 자동으로 프로젝트의 settings.py 경로와 환경변수를 세팅해줍니다.
    - 프로젝트 내부에서 작업할 때는 이 방식을 쓰는 것이 권장됩니다.

결과로만 보면 만들어지는 앱은 동일합니다. 다만 manage.py를 사용하는 것이 현재 프로젝트를 인식하고 앱을 생성하기 때문에 오작동을 방지하기 위해서라도 manage.py startapp이 권장되는 방법입니다.

##### II. Pycharm (IDE)를 이용한 앱 생성
![스크린샷](/statics/12/12_01.png)

#### 2. settings.py 앱 등록
파이참을 통해 생성했다면 자동으로 등록되어 있을 것 입니다.
```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'forum',
    # or
    # 'forum.apps.ForumConfig',
    'member',
]
```

#### 3. forumwithdjango/urls.py 라우팅
```python
urlpatterns = [
    path('', forum_views.post_list),
    path('admin/', admin.site.urls),
    path('posts/', include('forum.urls')),
    path('member/', include('member.urls')),
]
```
member 앱 내에 urls.py 가 없다는 에러가 표시될 것 입니다. urls.py 를 생성해 줍시다.

##### member/urls.py
```python
from django.urls import path

app_name = 'member'

urlpatterns = [
    
]
```

---

### 로그인 구현

#### member/urls.py
사용자가 로그인 할 url을 지정해 줍시다.
```python
from django.urls import path

#Django의 기본 인증 시스템. 로그인과 관련된 기능을 기본적으로 제공합니다.
from django.contrib.auth import views as auth_views

app_name = 'member'

urlpatterns = [
    path('login/',
         # 위에서 Django의 auth 인증 시스템을 불러왔고 그 내부에는 views.py가 기본적으로 작성되어 있습니다.
         # 다만 우리가 forum앱을 사용할 때와 달리 function의 이름이 파스칼 케이스로 작성되어 있고
         # 뒤에 '.as_view()'가 작성되어 있음을 볼 수 있는데 이는 forum/views.py의 Function-Based View(FBV)
         # 가 아닌 Class-Based View(CBV)방식으로 views.py가 작성되어 있기 때문입니다. 

         # FBV와 CBV의 차이점:
         # FBV (Python 함수로 뷰를 구현)
            # 초보자 친화적 (구현이 쉽고, 코드가 직관적)
            # 단순한 로직이나 1회성, 특수 비즈니스 로직, 예외처리에 유리
            # 데코레이터를 활용한다. (@login_required, @csrf_exempt, @api_view ...) (추후 설명합니다.)

            # 코드 중복 (비슷한 기능이 많아질수록 중복코드가 늘고 유지보수가 힘듬)
            # CRUD 구현시 HTTP 메서드 별로 분기 처리해야 하며 유사한 뷰 반복됨
            # 함수 기반이다 보니 코드 재사용성이 떨어지고 객체지향 패턴 적용이 어려움
                # 확장성 제한 (상속 구조가 존재하지 않아 재사용과 유지보수에 불편함이 존재)

         #CBV (Python 클래스로 뷰를 구현)
            # 클래스를 기반으로 하기 때문에 객체지향적 설계를 합니다.
                # 상속, 오버라이딩 등 객체지향 프로그래밍의 장점 활용
                # 재사용성, 유지보수에 큰 도움을 줍니다.
            # DRY(Don't Repeat Yourself) 
                # (Django에서 기본적으로 제공하는 제네릭뷰를 상속받아 반복코드를 최소화)
                # => 하나하나 기본적인 코드를 작성할 필요가 없고 개발시간이 단축됩니다.
            # REST API HTTP 메서드 분리의 이점 
                # (FBV에서는 @api_view(['GET', 'PUT', 'DELETE']) 데코레이터를 작성하고 함수 내에서
                # 또 if request.method == '***' 조건문 분기로 작성하던 것을 클래스 내에 메서드로 
                # def get(self, request, pk), def put(...), def delete(...) 딸깍이 가능합니다.)

            # 초반에 배우기 어렵습니다. 여러 제네릭 뷰들을 학습해야하고 객체지향을 기반하여 작성해야 합니다.
            # 복잡한 기능 구현시 클래스를 기반하기 때문에 코드가 길어질 수 있습니다.

        # 서로의 장단점을 비교하면 왜 이러한 장점이 있는지를 확인하실 수 있습니다.
        # 기본적인 CRUD는 CBV, 1회성 복잡한 로직, 예외처리는 FBV로 처리하는 방식이 자주 사용됩니다.
            # 단순하고 특별한 로직: FBV / 반복적이고 확장성, 재사용성이 중요한 경우: CBV
        # 결국 Django를 사용하려면 두 방식 모두 알고있어야 합니다. 다만 이번프로젝트는 FBV방식을 사용합니다.
         auth_views.LoginView.as_view(),
         name='login'),
]
```

#### 로그인 템플릿 생성
/templates/에 registration/login.html을 생성합니다.
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
    <!-- 이번에도 action을 지정하지 않았습니다. 
    /member/login/주소에 대한 GET, POST 요청을 view가 따로 처리해 줍니다. -->
    <!-- GET: 로그인 페이지 렌더 / POST: 로그인 요청 받음 -->
    <form method="POST">
        {% csrf_token %}
        <div class="form-group">
            <label>ID:
                <input type="text" name="username"class="form-control" required
                       value="{{ form.username.value|default_if_none:'' }}">
            </label>
        </div>
        <div class="form-group">
            <label>Password:
                <input type="password" name="password" class="form-control" required>
            </label>
        </div>
        <input type="submit" class="btn btn-primary mt-3" value="Login">
    </form>
</div>
{% endblock body %}
```
/templates/form_error.html
```html
{% if form.errors %}
<div class="alert alert-danger" role="alert">
    {% for field in form %}
        <!--필드 에러-->
        <!--
        특정 필드(input)에 직접 연결된 에러.
        사용자가 입력한 값이 is_valid() 메서드에서 에러가 도출 되었을 때
        ex. 공백 제출, 이메일 형식이 잘못됨, 비밀번호가 너무 짧음 ...
        -->
        {% if field.errors %}
        <div>
            <strong>{{ field.label }}</strong>
            {{ field.errors }}
        </div>
        {% endif %}
    {%  endfor %}

    <!--넌필드 에러-->
    <!--
    특정 필드(input)이 아니라 폼 전체의 상태나 필드 간의 관계에서 발생하는 에러
    여러 필드의 값을 조합하여 검증할 때, 특정 필드에 귀속시키기 어려운 에러가 필요할 때
    ex. 비밀번호, 비밀번호 재확인 두 값이 일치하지 않을 때, 
        1 과 2 둘 중 한 필드는 꼭 입력이 필요한데 둘 다 입력되지 않았을 때,
        아이디나 비밀번호가 틀렸을 때(무엇이 틀렸는지 설명 X)
    -->
    {% for error in form.non_field_errors %}
        <script>
        console.log("넌필드 에러")
        </script>
        <div>
            <strong>{{ error|escape }}</strong>
        </div>
    {% endfor %}
</div>
{% endif %}
```
기존에 form.field.errors를 사용하던 post_form.html도 {% include 'form_errors.html' %} 재사용 합시다.

#### + 왜 templates/member/login.html이 아닌가
Django 의 인증 시스템은 로그인에 관련한 템플릿은 /registration/ 에서 템플릿을 찾도록 설계되어 있습니다.
이것을 바꿀 수 없는 것은 아니지만 기본 컨벤션을 따르는게 유지보수나 협업 측면에서 권장되는 방법입니다.
그렇더라도 바꾸고 싶다면.
```python
urlpatterns = [
# https://docs.djangoproject.com/en/5.1/topics/auth/default/#django.contrib.auth.views.LoginView
    path('signin/',
         auth_views.LoginView.as_view(
             # 템플릿 경로 설정
             template_name='member/signin.html'),
         name='sign_in'),
```
settings.py
```python
...

# django 의 login required 사용 시 로그인 페이지 명명 (default setting: /accounts/login/)
# 우리의 프로젝트도 /member/로 라우팅을 하기때문에 이후에 설정해줘야 함.
LOGIN_URL = '/member/signin/'
```
위의 코드처럼 템플릿 url 태그 이름을 변경하고, 라우트 주소를 signin/, signout/으로 변경해도 괜찮습니다. 
다만 Django의 컨벤션과는 다르다라는 사실만 기억하고 프로젝트를 진행하신다면 문제될 것은 없습니다.

#### navbar.html 에 로그인 링크를 추가
navbar.html
```html
<nav class="navbar navbar-expand-lg bg-body-tertiary">
    <div class="container-fluid">
        ...
        <div class="collapse navbar-collapse" id="navbarSupportedContent">
            <ul class="navbar-nav me-auto mb-2 mb-lg-0">
            {% if request.user.is_authenticated %}
                ...
            {% else %}
                <li class="nav-item">
                    <a class="nav-link" href="{% url 'member:login' %}">로그인</a>
                </li>
                ...
            {% endif %}
            </ul>
        </div>
    </div>
</nav>
```

#### 로그인 해봅시다.
navbar의 로그인 링크를 클릭하여 로그인 페이지로 이동합니다.
![스크린샷](/statics/12/12_02.png)

required 속성을 제거하고 빈 값을 제출했을 때 (필드 에러):
![스크린샷](/statics/12/12_03.png)

비밀번호를 틀렸을 때 (넌필드 에러):
![스크린샷](/statics/12/12_04.png)

![스크린샷](/statics/12/12_05.png)
정상적으로 로그인을 완료했는데 404 에러가 표시됩니다.
주소창을 확인해보니 /accounts/profile/로 연결되어 있습니다. 여기에서 우리는 2가지를 알 수 있는데:
1. Django의 인증 시스템은 사용자에 대한 기능 라우팅 주소로 /accounts/를 사용한다.
    - 템플릿 저장 위치처럼 컨벤션을 따르지 않은 이유는 LOGIN_URL 설정을 체험하기 위함입니다.
    앱이름과 라우팅 url을 'accounts'로 설정하는 것이 코드의 길이를 줄이고 유지보수 하기에는 최선의 방법입니다.
        - Django 인증 시스템의 컨벤션
            1. 앱 이름: accounts
            2. url path: /accounts/***
            3. template 경로: /templates/registration/***
2. 로그인시 /accounts/profile/로 이동한다.

##### 해결방법
1. views.py 에서 클래스뷰를 불러올 때 next_page 속성 전달
```python
from django.urls import path
from django.contrib.auth import views as auth_views

app_name = 'member'

urlpatterns = [
    path('login/',
         auth_views.LoginView.as_view(next_page='/'),
         name='login'),
]
```
해당 뷰에만 적용되는 옵션입니다. 
여러 로그인 뷰가 있다면 각각 다르게 설정 가능하고, 뷰에서 next_page를 지정하면 아래의 LOGIN_REDIRECT_URL보다 우선 적용됩니다.

2. settings.py LOGIN_REDIRECT_URL = '/' 변수 삽입
```python
...

# 추후 @login_required 어노테이션을 사용할때 사용합니다.
LOGIN_URL = '/member/login/'
LOGIN_REDIRECT_URL = '/'
```
프로젝트 전체의 모든 로그인 뷰에 공통 적용합니다. 

#### 결과 확인
![스크린샷](/statics/12/12_06.png)

---

### 로그아웃 구현
member/urls.py
```python
from django.urls import path
from django.contrib.auth import views as auth_views

app_name = 'member'

urlpatterns = [
    path('login/',
         auth_views.LoginView.as_view(),
         name='login'),
    
    path('logout/',
         # 로그인과 같이 뷰함수를 작성하지 않고 Django auth의 로그아웃뷰를 재사용 합니다.
         auth_views.LogoutView.as_view(),
         name='logout'),
]
```

url라우팅을 완료했으니 'http://localhost:8000/member/logout/' 주소로 접속해 봅시다.
![스크린샷](/statics/12/12_07.png)
분명히 로그인과 같은 방식으로 작성했음에도 로그아웃은 커녕 페이지가 작동하지 않는다는 405 에러가 발생합니다.

그 이유는 장고 5 버전 이후로는 auth 의 로그아웃 뷰가 GET 방식의 요청은 받지 않습니다.
직접 뷰 함수를 작성 (/member/views.py)하여 사용하거나
```python
def logout(request):
    auth.logout(request)
    return redirect('forum:post_list')
```
LogoutView를 사용하고 싶다면 전부 POST 요청해야만 합니다.

#### navbar에 로그아웃 POST요청 작성
```html
<nav class="navbar navbar-expand-lg bg-body-tertiary">
    <div class="container-fluid">
        ...
        <div class="collapse navbar-collapse" id="navbarSupportedContent">
            <ul class="navbar-nav me-auto mb-2 mb-lg-0">
            {% if request.user.is_authenticated %}
                <li class="nav-item dropdown">
                    ...
                    <ul class="dropdown-menu">
                        <li><a class="dropdown-item" href="">프로필</a></li>
                        <!-- 기존에 적어두었던 navbar 코드에 POST form이 적혀져 있던 이유입니다. -->
                        <li>
                            <!-- action에 로그아웃 뷰 네임을 작성 후 로그아웃 해봅시다. -->
                            <form action="{% url 'member:logout' %}" method="POST">
                                {% csrf_token %}
                                <input class="dropdown-item" type="submit"
                                       value="{{ request.user.username }}(로그아웃)">
                            </form>
                        </li>
                        <li><a class="dropdown-item" href="">비밀번호 변경</a></li>
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

##### 결과 확인
![스크린샷](/statics/12/12_08.png)
![스크린샷](/statics/12/12_09.png)
로그인 했을 때를 떠올려 보면 무엇을 해야할지 감이 잡히실 것 같습니다.

settings.py
```python
...

LOGIN_URL = '/member/login/'
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'
```

다시 사이트에 접속하여 로그인 후 navbar의 로그아웃을 클릭해 봅시다.
정상적으로 로그아웃이 작동하는 것을 보실 수 있습니다.