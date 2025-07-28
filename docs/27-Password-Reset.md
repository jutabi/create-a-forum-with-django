## 27. 비밀번호 초기화

### Django Password Reset View 사용해 보기

#### member/urls.py
Django의 auth_views는 비밀번호 초기화에 대한 기본 클래스뷰를 제공합니다. 사용해 봅시다.
```python
from django.contrib.auth import views as auth_views
from django.urls import path

from member import views

app_name = 'member'

urlpatterns = [
    path('login/',  views.custom_login,              name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    path('signup/', views.signup,                    name='signup'),

    path('password-reset/', 
         auth_views.PasswordResetView.as_view(),
         name='password_reset'),
    path('password-reset/done/', 
         auth_views.PasswordResetDoneView.as_view(),
         name='password_reset_done'),
    path('password-reset/confirm/<uidb64>/<token>/', 
         auth_views.PasswordResetConfirmView.as_view(),
         name='password_reset_confirm'),
    path('password-reset/complete/', 
         auth_views.PasswordResetCompleteView.as_view(),
         name='password_reset_complete'),
]
```
로그아웃과 마찬가지로 url 라우팅 만으로 설정할 수 있습니다. 한번 라우팅한 /member/password-reset/으로 접속해 봅시다.

![스크린샷](/statics/27/27_01.png)
장고 관리자 페이지처럼 템플릿도 제공됩니다.

이제 사용자에게 이메일을 보내기 위해서는 ***SMTP***라는 것을 설정해야 합니다만 일단 간단하게 백엔드 콘솔에 출력하도록 settings.py에서 설정해 봅시다.
#### settings\.py
```python
...
# EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
# EMAIL_HOST = 'smtp.gmail.com'
# EMAIL_PORT = 587
# EMAIL_USE_TLS = True
# EMAIL_HOST_USER = 'your-email@gmail.com'
# EMAIL_HOST_PASSWORD = 'your-email-password'
# DEFAULT_FROM_EMAIL = 'YourAppName <your-email@gmail.com>'

# 원래는 위의 코드처럼 이메일 서비스에서 SMTP 관련 키를 발급 받고 등록하여 사용자에게 메일을 보내야 합니다.
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
```
이제 메일 주소를 입력하고 비밀번호 초기화 버튼을 클릭해 보면:

![스크린샷](/statics/27/27_02.png)
위와 같은 에러가 발생하게 됩니다. 읽어보면 템플릿을 렌더링 하는 도중 password_reset_confirm을 찾을 수 없다고 하는데요.

![스크린샷](/statics/27/27_03.png)
에러가 난 템플릿 위치와 해당 파일 내의 에러가 난 템플릿 url태그가 보입니다. 

우리의 프로젝트는 urls\.py 에서 namespace를 사용하고 있습니다. 그래서 템플릿 url태그를 사용해 뷰에 접근하려고 할 때 해당하는 앱의 이름을 적어주어야 합니다. (member:password-reset)
하지만 장고가 제공하는 템플릿은 그것을 고려할 수 없기 때문에 namespace가 작성되어 있지 않아 해당 뷰를 불러올 수 없다는 에러가 발생합니다.
직접 해당 템플릿에 접속해 보면:
![스크린샷](/statics/27/27_04.png)
```html
{% load i18n %}{% autoescape off %}
{% blocktranslate %}You're receiving this email because you requested a password reset for your user account at {{ site_name }}.{% endblocktranslate %}

{% translate "Please go to the following page and choose a new password:" %}
{% block reset_link %}
{{ protocol }}://{{ domain }}{% url 'password_reset_confirm' uidb64=uid token=token %}
{% endblock %}
{% translate 'In case you’ve forgotten, you are:' %} {{ user.get_username }}

{% translate "Thanks for using our site!" %}

{% blocktranslate %}The {{ site_name }} team{% endblocktranslate %}

{% endautoescape %}
```
사용자에게 전달해 줄 이메일의 내용이 담겨있는 템플릿이며 ***비밀번호를 리셋하는 링크***에 해당하는 url 템플릿 태그에서 오류가 발생하는 것을 확인하실 수 있습니다.

기본 제공 템플릿의 기존의 내용을 이용하여 새로운 템플릿을 작성해 봅시다.

#### /templates/registration/password_reset_email.html
```html
{% load i18n %}{% autoescape off %}
{% blocktranslate %}You're receiving this email because you requested a password reset for your user account at {{ site_name }}.{% endblocktranslate %}

{% translate "Please go to the following page and choose a new password:" %}
{% block reset_link %}
{{ protocol }}://{{ domain }}{% url 'member:password_reset_confirm' uidb64=uid token=token %}
{% endblock %}
{% translate 'In case you’ve forgotten, you are:' %} {{ user.get_username }}

{% translate "Thanks for using our site!" %}

{% blocktranslate %}The {{ site_name }} team{% endblocktranslate %}

{% endautoescape %}
```
#### /member/urls.py
```python
urlpatterns = [
    ...
    path('password-reset/',
         auth_views.PasswordResetView.as_view(
             # 만약 템플릿 저장 위치가 templates/registration/이 아니라면 아래처럼 지정이 필요합니다.
             # 우리의 프로젝트는 장고의 컨벤션을 따르고 있기 때문에 지정해주지 않아도 됩니다.
             email_template_name='***/password_reset_email.html'
         ),
         name='password_reset'),
     ...
]
```

다시한번 비밀번호 초기화 버튼을 클릭해 봅시다.

![스크린샷](/statics/27/27_05.png)
또다른 오류가 우리를 맞이해 줍니다. 이번에도 NoReverseMath 에러가 발생하는데요. 이번엔 password_reset_done을 찾을 수 없다고 알려줍니다.
Raised during의 PasswordResetView를 Pycharm에서 Ctrl+클릭 혹은 Ctrl+B / VsCode에서 Ctrl+클릭 혹은 F12 (Mac은 Command)를 입력하여 레퍼런스(정의)로 이동해 봅시다.

![스크린샷](/statics/27/27_06.png)
바로 저 부분 success_url에서 오류가 발생하는데요. 이전의 템플릿에서 발생했던 오류와 같이 네임스페이스를 사용하여 발생하는 문제입니다. 성공 시 url을 정해주는 부분인데 이곳에도 앱의 이름을 적용시켜 주어야 합니다.

#### + reverse
reverse(), reverse_lazy()는 URL의 이름을 실제 문자열 값으로 반환해주는 함수입니다.

##### reverse()
- 클래스가 정의될 때(함수가 호출되는 즉시) URLConf를 참조하여 문자열을 반환해 줍니다.
##### reverse_lazy()
- 실제 URL이 필요할 때 까지(결과를 리턴할 때) URL 계산을 미룹니다. 즉 지연 평가를 적용하는 객체를 반환합니다.

##### 왜 reverse_lazy()를 사용하나요
- 클래스 기반 뷰(CBV)의 success_url과 같은 속성은 클래스가 정의될 때 지정되어야 합니다. 다만 장고의 URLConf를 참조하게 되는데 아직 완전히 로딩되지 않았을 수 있습니다. 
- 이때 reverse()를 사용하면 존재하지 않는 URL을 참조하는 문제가 발생할 수 있습니다.
- 반면 reverse_lazy()는 실제 URL이 필요할 때 참조하여 계산하기 때문에 이런 초기화 문제를 해결할 수 있습니다.
- 함수 기반 뷰(FBV)는 호출될 때 이미 URLConf 로딩이 보장됨으로 reverse()를 사용해도 됩니다.

#### member/urls.py
```python
urlpatterns = [
    ...
    path('password-reset/',
         auth_views.PasswordResetView.as_view(
          # reverse_lazy()를 사용하여 member앱의 password_reset_done url라우터를 가리켜 줍시다.
             success_url=reverse_lazy('member:password_reset_done')
         ),
         name='password_reset'),
    ...
]
```
위의 코드처럼 urls.py에서 클래스를 호출할 때 success_url 매개 변수를 입력해주어 문제를 해결합니다.

이제 다시 한번 비밀번호 초기화 버튼을 클릭해 봅시다.
![스크린샷](/statics/27/27_07.png)
```bash
Content-Type: text/plain; charset="utf-8"
MIME-Version: 1.0
Content-Transfer-Encoding: 8bit
Subject: =?utf-8?b?bG9jYWxob3N0OjgwMDDsnZgg67mE67CA67KI7Zi4IOyerOyEpOyglQ==?=
From: webmaster@localhost
To: jason@localhost.com
Date: Mon, 28 Jul 2025 08:26:04 -0000
Message-ID: <175369116457.32432.6630758281887860900@JUTABI.GRAX66>


localhost:8000의 계정 비밀번호를 초기화하기 위한 요청으로 이 이메일이 전송되었습니다.

다음 페이지에서 새 비밀번호를 선택하세요.

http://localhost:8000/member/password-reset/confirm/Mw/ctnv3g-4ac299d4939a80da58881641ccef5205/

In case you’ve forgotten, you are: qwer1234

사이트를 이용해 주셔서 고맙습니다.

localhost:8000 팀



-------------------------------------------------------------------------------
[28/Jul/2025 17:26:04] "POST /member/password-reset/ HTTP/1.1" 302 0
[28/Jul/2025 17:26:04] "GET /member/password-reset/done/ HTTP/1.1" 200 3085


```

브라우저와 백엔드 콘솔에 결과가 정상적으로 출력되는 것을 확인하실 수 있습니다.

이제 콘솔의 이메일 내용에 있는 비밀번호 초기화 링크에 접속해 봅시다.

![스크린샷](/statics/27/27_08.png)
새로운 비빌번호를 작성 후 변경 버튼을 클릭해 봅시다.

![스크린샷](/statics/27/27_09.png)
같은 에러가 발생합니다. PasswordResetConfirmView 정의로 이동해 봅시다.

![스크린샷](/statics/27/27_10.png)
이번에도 success_url이 존재합니다. 매개변수를 삽입 후 이메일을 재전송 하여 결과를 확인해 봅시다.
#### member/urls.py
```python
urlpatterns = [
    ...
     path('password-reset/confirm/<uidb64>/<token>/',
         auth_views.PasswordResetConfirmView.as_view(
             success_url=reverse_lazy('member:password_reset_complete')
         ),
         name='password_reset_confirm'),
    ...
]
```
#### 결과 확인
![스크린샷](/statics/27/27_11.png)


#### 최종 코드
```python
urlpatterns = [
    path('login/',  views.custom_login,              name='login'),
    path('logout/', auth_views.LogoutView.as_view(), name='logout'),
    path('signup/', views.signup,                    name='signup'),

    path('password-reset/',
         auth_views.PasswordResetView.as_view(
             success_url=reverse_lazy('member:password_reset_done')),
         name='password_reset'),
    
    path('password-reset/done/',
         auth_views.PasswordResetDoneView.as_view(),
         name='password_reset_done'),
    
    path('password-reset/confirm/<uidb64>/<token>/',
         auth_views.PasswordResetConfirmView.as_view(
             success_url=reverse_lazy('member:password_reset_complete')),
         name='password_reset_confirm'),
    
    path('password-reset/complete/',
         auth_views.PasswordResetCompleteView.as_view(),
         name='password_reset_complete'),
]
```

---

### 커스텀 템플릿 작성
기능적으로는 정상 작동하지만 해당 템플릿에는 게시판으로 향하는 링크가 아닌 Django관리자 페이지로 향하는 링크가 작성되어 있습니다. 커스텀 템플릿을 작성하여 프론트엔드 디자인을 변경해 봅시다.

간단히 해당 템플릿을 프로젝트의 템플릿 폴더에 붙여넣기 하여 수정합니다.
![스크린샷](/statics/27/27_12.png)

#### password_reset_form.html
```html
{% extends 'base.html' %}
{% load i18n %}
{% block title %}
Password reset
{% endblock %}

{% block body %}
<div class="error container">
    {% include 'form_errors.html' %}
</div>
<div class="container justify-content-center d-flex">
    <form method="POST">
        {% csrf_token %}
        <div class="form-group">
            <label>{% translate 'Email address' %}:
                <input type="email" name="email" class="form-control" required value="{{ request.POST.email }}">
            </label>
        </div>

        <div class="mt-3">
            <button type="submit" class="btn btn-primary">{% translate 'Reset my password' %}</button>
        </div>
    </form>
</div>
{% endblock %}
```

#### password_reset_done.html
```html
{% extends 'base.html' %}
{% load i18n %}
{% block title %}
Password reset
{% endblock %}

{% block body %}
<div class="container text-center mt-3">
    <p>{% translate 'We’ve emailed you instructions for setting your password, if an account exists with the email you entered. You should receive them shortly.' %}</p>
    <p>{% translate 'If you don’t receive an email, please make sure you’ve entered the address you registered with, and check your spam folder.' %}</p>
</div>
{% endblock %}
```

#### password_reset_confirm.html
```html
{% extends 'base.html' %}
{% load i18n static %}
{% block title %}
Password reset confirmation
{% endblock %}

{% block body %}

{% if validlink %}
<div class="error container">
    {% include 'form_errors.html' %}
</div>
<div class="container text-center">
    <p>{% translate "Please enter your new password twice so we can verify you typed it in correctly." %}</p>
    <form method="post">
        {% csrf_token %}
        <input type="hidden" autocomplete="username" value="{{ form.user.get_username }}">

        <div class="form-group">
            <label>{% translate 'New password:' %}
                <input type="password" name="new_password1" class="form-control">
            </label>
        </div>

        <div class="form-group">
            <label>{% translate 'Confirm password:' %}
                <input type="password" name="new_password2" class="form-control">
            </label>
        </div>

      <div class="mt-3">
            <button type="submit" class="btn btn-primary">{% translate 'Change my password' %}</button>
      </div>
    </form>
</div>
{% else %}
<div class="text-center container">
<p>{% translate "The password reset link was invalid, possibly because it has already been used.  Please request a new password reset." %}</p>
</div>
{% endif %}

{% endblock %}
```

#### password_reset_complete.html
```html
{% extends 'base.html' %}
{% load i18n %}
{% block title %}
Password reset
{% endblock %}

{% block body %}
<div class="container text-center mt-3">
<p>{% translate "Your password has been set.  You may go ahead and log in now." %}</p>

<a class="btn btn-primary" href="{% url 'member:login' %}">{% translate 'Log in' %}</a>
</div>
{% endblock %}
```

#### navbar.html
```html
{% if request.user.is_authenticated %}
     ...
{% else %}
     <li class="nav-item">
          <a class="nav-link" href="{% url 'member:login' %}">로그인</a>
     </li>
     <li class="nav-item">
          <a class="nav-link" href="{% url 'member:signup' %}">회원가입</a>
     </li>
     <li class="nav-item">
          <!-- 로그아웃 사용자에게 비밀번호 찾기 링크 추가 -->
          <a class="nav-link" href="{% url 'member:password_reset' %}">비밀번호 찾기</a>
     </li>
{% endif %}
```

### 결과 확인
![스크린샷](/statics/27/27_13.png)
![스크린샷](/statics/27/27_14.png)
![스크린샷](/statics/27/27_15.png)
![스크린샷](/statics/27/27_16.png)
![스크린샷](/statics/27/27_17.png)

#### 문제 발생
정상 작동을 확인했으나 complete 페이지에서 로그인 링크를 클릭하여 로그인을 하면 다시 complete 페이지로 돌아오는 현상이 발생합니다.
이는 우리가 비동기 로그인 구현 시 작성했던 코드 때문인데요
##### login.js
```js
const loginForm = document.querySelector("#login-form");
const queryParams = new URLSearchParams(window.location.search);
loginForm.addEventListener("submit", async function (event) {
    event.preventDefault();
    const formData = new FormData(loginForm);
    try {
        ...

        if (data.status === "success") {
          //if (document.referrer)
          // 기존 코드의 referrer에 complete페이지가 걸려 이전 페이지로 이동하게 되었습니다.
          // 대신 URLSearchParams(window.location.search)에 next값이 있을때만 이동하도록 변경합니다.
            if (queryParams.has("next")) {
                // console.log(queryParams.get("next"));
                location.replace(document.referrer);
            } else {
                // url 입력, 즐겨찾기, 새탭 열기 같은 '이전 페이지'가 없는 경우
                location.replace('/');
            }
        }
        else {
            ...
        }
    } catch (error) {
        console.log(error);
    }
})
```
이제 정상작동이 되는 것을 확인할 수 있습니다.

---

다음 차시에서 Gmail의 smtp를 등록하고 django-environ을 이용해 settings에서 사용하는 주요 키 값을 별도의 파일로 분리해, 버전 관리에 포함되지 않도록 구성해 봅시다.