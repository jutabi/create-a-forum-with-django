## 10. 네비게이션 바, bootstrap.js

[참고 문헌: 점프 투 장고 / 3-01 Bootstrap 내비게이션바](https://wikidocs.net/71108)

[https://getbootstrap.com/docs/5.3/components/navbar/](https://getbootstrap.com/docs/5.3/components/navbar/)
네비게이션 바에 대한 기본적인 설명은 공식문서로 대체합니다.

### 1. 템플릿 작성
templates/navbar.html
```html
<nav class="navbar navbar-expand-lg bg-body-tertiary">
    <div class="container-fluid">
        <a class="navbar-brand" href="{% url 'forum:post_list' %}">게시판</a>
        <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarSupportedContent"
                aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
            <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse" id="navbarSupportedContent">
            <ul class="navbar-nav me-auto mb-2 mb-lg-0">
            {% if request.user.is_authenticated %}
                <li class="nav-item dropdown">
                    <button class="btn dropdown-toggle" data-bs-toggle="dropdown" aria-expanded="false">
                        {{ request.user.username }}
                    </button>
                    <ul class="dropdown-menu">
                        <li><a class="dropdown-item" href="">프로필</a></li>
                        <li>
                            <form action="" method="POST">
                                {% csrf_token %}
                                <input class="dropdown-item" type="submit"
                                       value="{{ request.user.username }}(로그아웃)">
                            </form>
                        </li>
                        <li><a class="dropdown-item" href="">비밀번호 변경</a></li>
                    </ul>
                </li>
            {% else %}
                <li class="nav-item">
                    <a class="nav-link" href="">로그인</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="">회원가입</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link" href="">비밀번호 찾기</a>
                </li>
            {% endif %}
            </ul>
        </div>
    </div>
</nav>
```
부트스트랩을 완전히 이해하고 갈 것이 아니라면 예제를 복사 붙여넣기 후 공식문서의 각 클래스 설명을 보며 이해하는 것이 좋습니다.
필자도 공식문서의 예제를 복사 붙여넣기 하여 엘리먼트들을 수정해 나갔습니다.

위의 예제 코드에서 {% if request.user.is_authenticated %} 부터 로그인에 관련한 코드들이 작성되어 있습니다. 현재는 자세하게 설명하지 않고 로그인 기능 구현시 설명합니다. 

templates/base.html
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    ...
</head>
<body>
<!-- extends가 아닌 include를 사용하여 navbar만 구현한 템플릿의 내용을 추가한다. -->
{% include 'navbar.html' %}

{% block body %}
{% endblock body %}
</body>
</html>
```

### 2. 드롭다운 컴포넌트
Django-administrator에서 했던 것 처럼 /admin/에 접속하여 로그인을 한 뒤 post_list로 돌아와 봅시다.
![스크린샷](/statics/10/10_01.png)
```html
<ul class="navbar-nav me-auto mb-2 mb-lg-0">
<!-- 로그인 한 사용자인가요? -->
{% if request.user.is_authenticated %}
    <li class="nav-item dropdown">
        <button class="btn dropdown-toggle" data-bs-toggle="dropdown" aria-expanded="false">
            {{ request.user.username }}
        </button>
```
Django는 기본적으로 Member 모델(사용자)이 제공되고 그 모델을 Admin이 참조하고 있기 때문에 로그인 된 사용자의 username {{ request.user.username }}으로 admin이 출력되고 드롭다운 메뉴도 정상적으로 출력되어 있는 것을 확인하실 수 있습니다.

그런데 막상 드롭다운 메뉴를 클릭해보면 아무런 반응이 없는 것을 확인할 수 있습니다. 그 이유는 bootstrap의 dropdown 기능은 css 만으로 구현할 수 없는, bootstrap.js가 필요하기 때문입니다.

#### bootstrap.bundle.min.js 탑재
##### I. 직접 다운로드
기존의 부트스트랩.css를 탑재할 때 다운받았던 압축파일의 js/bootstrap.bundle.min.js를 프로젝트의 static/js/ 내에 위치시키고 우리가 post_detail.js 스태틱을 불러왔던 것처럼 base.html에 불러와 봅시다.
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    ...
</head>
<body>
{% include 'navbar.html' %}
{% block body %}
{% endblock body %}

{% load static %}
<!-- 주의할점: bootstrap.min.js (X) 번들 버전으로 탑재해야 dropdown 관련 기능이 동작합니다. -->
<script src="{% static 'js/bootstrap.bundle.min.js' %}"></script>
</body>
</html>
```

##### II. CDN 이용
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    ...
</head>
<body>
{% include 'navbar.html' %}
{% block body %}
{% endblock body %}

<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.7/dist/js/bootstrap.bundle.min.js" 
        integrity="sha384-ndDqU0Gzau9qJ1lfW4pNLlhNTkCfHzAVBReH9diLvGRem5+R9g2FzA8ZGN954O5Q" 
        crossorigin="anonymous"></script>
</body>
</html>
```

##### 결과 확인
![스크린샷](/statics/10/10_02.png)
네비게이션 바의 드롭다운 컴포넌트 정상 작동

![스크린샷](/statics/10/10_03.png)
관리자 페이지에 접속해 로그아웃을 진행합니다. 

![스크린샷](/statics/10/10_04.png)
비로그인 ( {% else %} )구문이 정상적으로 실행되는 것을 확인할 수 있습니다.