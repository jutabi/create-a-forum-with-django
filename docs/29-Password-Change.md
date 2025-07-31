## 29. 비밀번호 변경

### 기본 제공 템플릿 사용
먼저 장고에서 기본으로 제공되는 비밀번호 변경 템플릿을 사용해본 뒤, 커스텀 템플릿을 작성해 봅시다.

#### member/urls.py
```python
from django.contrib.auth import views as auth_views
from django.urls import path, reverse_lazy

from member import views

app_name = 'member'

urlpatterns = [
    # login logout signin

    # password-reset

    path('password-change/',
         auth_views.PasswordChangeView.as_view(),
         name='password_change'),

    path('password-change/done/',
         auth_views.PasswordChangeDoneView.as_view(),
         name='password_change_done'),
]
```

#### templates/navbar.html
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
                        ...
                        <!-- 로그인 한 사용자에게 비밀번호 변경 드롭바 메뉴 추가 -->
                        <li><a class="dropdown-item" href="{% url 'member:password_change' %}">비밀번호 변경</a></li>
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

#### 결과 확인
![스크린샷](/statics/29/29_01.png)
![스크린샷](/statics/29/29_02.png)
password-reset때와 같은 에러가 발생합니다. success_url 매개변수를 작성해 줍시다.
```python
path('password-change/',
        auth_views.PasswordChangeView.as_view(
            success_url=reverse_lazy('member:password_change_done')),
        name='password_change'),
```
#### 주의
success_url을 통해 완료 창을 보여주는 뷰가 연결되지 않았을 뿐 비밀번호 변경 요청은 정상처리 됩니다.

#### 결과 확인
![스크린샷](/statics/29/29_01.png)
![스크린샷](/statics/29/29_03.png)

---

### 커스텀 템플릿 작성하기
#### templates/registration/password_change_form.html
```html
{% extends 'base.html' %}
{% load i18n %}
{% block title %}
Password Change
{% endblock title %}

{% block body %}
<div class="error container mt-3">
    {% include 'form_errors.html' %}
    <div>
    <p>{% translate 'Please enter your old password, for security’s sake, and then enter your new password twice so we can verify you typed it in correctly.' %}</p>
    </div>
</div>
<div class="container justify-content-center d-flex">
    <form method="post">
        {% csrf_token %}

        <div class="form-group text-center">
            <label>{% translate 'Old password:' %}
                <input type="password" name="old_password" class="form-control">
            </label>
        </div>

        <div class="form-group text-center">
            <label>{% translate 'New password:' %}
                <input type="password" name="new_password1" class="form-control">
            </label>
            {% if form.new_password1.help_text %}
                <div class="help"
                        {% if form.new_password1.id_for_label %}
                     id="{{ form.new_password1.id_for_label }}_helptext"
                        {% endif %}>
                    {{ form.new_password1.help_text|safe }}
                </div>
            {% endif %}
        </div>

        <div class="form-group text-center">
            <label>{% translate 'Confirm password:' %}
                <input type="password" name="new_password2" class="form-control">
            </label>
            {% if form.new_password2.help_text %}
                <div class="help"
                        {% if form.new_password2.id_for_label %}
                     id="{{ form.new_password2.id_for_label }}_helptext"
                        {% endif %}>
                    {{ form.new_password2.help_text|safe }}
                </div>
            {% endif %}
        </div>

        <div class="mt-3 text-center">
            <button type="submit" class="btn btn-primary">{% translate 'Change my password' %}</button>
        </div>
    </form>
</div>
{% endblock %}
```

#### templates/registration/password_change_done.html
```html
{% extends 'base.html' %}
{% load i18n %}
{% block title %}
Password Change
{% endblock %}

{% block body %}
<div class="container text-center mt-3">
<p>{% translate "Your password was changed." %}</p>

<a class="btn btn-primary" href="{% url 'forum:post_list' %}">{% translate 'Home' %}</a>
</div>
{% endblock %}
```

#### 결과 확인
![스크린샷](/statics/29/29_05.png)
![스크린샷](/statics/29/29_06.png)
![스크린샷](/statics/29/29_07.png)