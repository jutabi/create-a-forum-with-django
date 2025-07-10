## 13. 회원가입

[참고 문헌: 점프 투 장고 / 3-06 회원가입](https://wikidocs.net/71303)

### urls\.py
```python
from django.urls import path
from django.contrib.auth import views as auth_views
from member import views

app_name = 'member'

urlpatterns = [
    path('login/',
         auth_views.LoginView.as_view(),
         name='login'),

    path('logout/',
         auth_views.LogoutView.as_view(),
         name='logout'),

    path('signup/',
         # Django 의 auth는 인증 관련 뷰는 제공하지만 회원가입 뷰는 제공하지 않습니다.
         views.signup,
         name='signup')
]
```

### User model
|User|
|----|
|username|
|password|
|nickname|
| email  |

### member/models.py
우리 프로젝트는 User가 nickname, email 속성을 가지고 있는데 그 중 nickname 속성은 Django의 User모델이 가지고 있지 않은 속성입니다.
email 속성은 가지고 있기에 커스텀Form에서 필드로 설정해주면 되지만 nickname은 해당하지 않습니니다. 
그래서 User모델을 상속하는 커스텀 모델을 생성합니다.
##### 주의!
'AbstractUser'를 상속해야 합니다. 기본 User 모델의 필드와 메서드를 가지고 있으면서 원하는 필드를 추가할 수 있게 해주는 추상클래스 입니다. 이 클래스를 상속해야 User 모델의 필드와 메서드를 그대로 사용할 수 있습니다.
만약 User 모델을 상속하면 데이터베이스에는 id(외래키)와 nickname 속성만을 가진 테이블이 따로 생성되게 됩니다. (멀티 테이블 상속)
[https://docs.djangoproject.com/en/5.2/ref/contrib/auth/#user-model](https://docs.djangoproject.com/en/5.2/ref/contrib/auth/#user-model)
```python
from django.db import models
from django.contrib.auth.models import AbstractUser

class ForumUser(AbstractUser):
    # 글자수를 10자로 제한합니다. (영어, 한국어 언어에 가리지 않고 10글자.)
    nickname = models.CharField(max_length=10)
```
settings.py
```python
...
# 생성한 커스텀 모델을 Django의 기본 USER 모델로 지정합니다. 만약 지정하지 않은 채 마이그레이션 시 오류가 납니다.
AUTH_USER_MODEL = 'member.ForumUser'
```
```
(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py makemigrations
Migrations for 'member':
  member\migrations\0001_initial.py
    + Create model ForumUser

(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py migrate       
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, forum, member, sessions
Running migrations:
  Applying member.0001_initial... OK

```


### member/forms.py
Post를 생성할 때 처럼 폼을 생성하고 is_valid()를 활용할 것입니다.
```python
from django import forms
from django.contrib.auth.forms import UserCreationForm

from member.models import ForumUser

# UserCreationForm(BaseUserCreationForm)의 BaseUCF는 username, password1, password2 3개의 속성만을 가집니다.
# UserCreationForm은 username의 중복 검사를 수행해 줍니다.
# 이메일, 닉네임 등 새로운 필드를 추가 하기 위해 UCF를 상속하고 필드를 생성합니다.
# https://docs.djangoproject.com/en/5.2/topics/auth/default/#django.contrib.auth.forms.UserCreationForm
class RegisterForm(UserCreationForm):
    # 새로운 필드를 생성했을 때는 메타 클래스에서 라벨을 지정하지 않고 선언 시 지정합니다.
    # id와 password는 UserCreationForm 에서 설정한 지역에 맞게 라벨을 지정해 줍니다.
    email = forms.EmailField(label='이메일')
    nickname = forms.CharField(max_length=10, label='닉네임')

    class Meta:
        model = ForumUser
        fields = ['username', 'password1', 'password2', 'nickname', 'email']

    # 중복을 검사하는 메서드 입니다. UserCreationForm의 clean함수와 거의 유사한 구조로 작성했습니다.
    def clean_email(self):
        """Reject emails that differ only in case."""
        email = self.cleaned_data['email']
        if ForumUser.objects.filter(email=email).exists():
            # forms의 validationError 메서드를 통해 중복값 검출시 에러메세지를 raise 해줍니다.
            raise forms.ValidationError('이미 가입된 이메일 주소 입니다.')
        else: return email

    def clean_nickname(self):
        """Reject nicknames that differ only in case."""
        nickname = self.cleaned_data['nickname']
        # 이미 ForumUser을 임포트 했으니 위의 이메일처럼 그대로 사용해도 되고
        # 아니면 아래의 코드처럼 사용해 self의 model을 불러올 수도 있습니다.
        if self._meta.model.objects.filter(nickname=nickname).exists():
            raise forms.ValidationError('이미 사용 중인 닉네임 입니다.')
        else: return nickname
```

### views\.py
```python
from django.contrib.auth import authenticate, login
from django.shortcuts import render, redirect
from member.forms import RegisterForm

def signup(request):
    if request.method == 'GET':
        form = RegisterForm()
        return render(request, 'registration/signup.html', {'form': form})
    elif request.method == "POST":
        form = RegisterForm(request.POST)
        if form.is_valid():
            form.save()
            # 회원가입 한 사용자는 바로 로그인 시켜줍니다.
            # login() 메서드는 User 객체가 필요함으로 authenticate() 메서드를 통해 User객체를 불러옵니다.
            user = authenticate(
                request,
                username=form.cleaned_data['username'],
                password=form.cleaned_data['password2']
            )
            login(request, user)
            return redirect('forum:post_list')
        return render(request, 'registration/signup.html', {'form': form})
```

### templates/registration/signup.html
```html
{% extends 'base.html' %}
{% block title %}
Register
{% endblock title %}

{% block body %}
<div class="container">
    {% include 'form_errors.html' %}
</div>
<div class="container justify-content-center" style="display: flex">
    <form method="POST">
        {% csrf_token %}
        <div class="form-group">
            <label>ID:
                <input type="text" name="username" class="form-control" required
                       value="{{ form.username.value|default_if_none:"" }}">
            </label>
        </div>
        <div class="form-group">
            <label>Password:
                <input type="password" name="password1" class="form-control" required>
            </label>
        </div>
        <div class="form-group">
            <label>Password 확인:
                <input type="password" name="password2" class="form-control" required>
            </label>
        </div>
        <div class="form-group">
            <label>닉네임 (최대 10자):
                <input type="text" name="nickname" class="form-control" required
                       value="{{ form.nickname.value|default_if_none:"" }}">
            </label>
        </div>
        <div class="form-group">
            <label>이메일:
                <input type="email" name="email" class="form-control" required
                       value="{{ form.email.value|default_if_none:"" }}">
            </label>
        </div>
        <input type="submit" class="btn btn-primary mt-3" value="Register">
    </form>
</div>
{% endblock body %}
```

### navbar 링크 추가
```html
<nav class="navbar navbar-expand-lg bg-body-tertiary">
    <div class="container-fluid">
        ...
        <div class="collapse navbar-collapse" id="navbarSupportedContent">
            <ul class="navbar-nav me-auto mb-2 mb-lg-0">
            {% if request.user.is_authenticated %}
                <li class="nav-item dropdown">
                    <button class="btn dropdown-toggle" data-bs-toggle="dropdown" aria-expanded="false">
                        {{ request.user.nickname }} <!--기존의 아이디 대신 닉네임으로 변경-->
                    </button>
                    <ul class="dropdown-menu">
                        ...
                    </ul>
                </li>
            {% else %}
                <li class="nav-item">
                    ...
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="{% url 'member:signup' %}">회원가입</a>
                </li>
                <li class="nav-item">
                    ...
                </li>
            {% endif %}
            </ul>
        </div>
    </div>
</nav>
```

결과 확인
![스크린샷](/statics/13/13_01.png)
빈 값 전송

![스크린샷](/statics/13/13_02.png)
중복 값, 일치하지 않는 비밀번호 1, 2 전송

![스크린샷](/statics/13/13_03.png)
가입 완료

---

### 슈퍼 유저 재생성
회원가입한 아이디를 로그아웃 하고 /admin/페이지에 접속해 관리자 계정에 로그인 해봅시다.
![스크린샷](/statics/13/13_04.png)

로그인이 안될 것입니다. 이유는 우리가 커스텀 유저 모델(ForumUser)을 생성하고(테이블 생성) 그 모델(테이블)을 Django 시스템의 기본 USER 모델로 지정했기 때문입니다.

데이터베이스 테이블을 확인해 봅시다.
```sql
select * from member_forumuser;
select * from auth_user;
```

![스크린샷](/statics/13/13_05.png)
![스크린샷](/statics/13/13_06.png)

이제 시스템의 User 테이블은 auth_user가 아닌 member_forumuser로 변경되어 기존의 테이블에서 가지고 있던 is_superuser==1 admin 계정을 사용할 수 없는 것입니다.

데이터베이스이기에 쿼리문을 사용하여 해결하는 방법이 있겠지만 간단하게 슈퍼유저를 재생성 합시다.
```
(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py createsuperuser
사용자 이름: admin
이메일 주소: admin@localhost.com
Password: 
Password (again):
This password is too short. It must contain at least 8 characters.
비밀번호가 너무 일상적인 단어입니다.
비밀번호가 전부 숫자로 되어 있습니다.
Bypass password validation and create user anyway? [y/N]: y
Superuser created successfully.
```


#### admin\.py
간단하게 회원 데이터를 관리하는 admin 기능을 연결합시다.
```python
from django.contrib import admin
from member.models import ForumUser

class ForumUserAdmin(admin.ModelAdmin):
    search_fields = ['username']
    list_display = ['username', 'nickname', 'email', 'date_joined']
    list_display_links = ['username']
    list_per_page = 10
    ordering = ['-date_joined']

admin.site.register(ForumUser, ForumUserAdmin)
```

![스크린샷](/statics/13/13_07.png)
![스크린샷](/statics/13/13_08.png)