## 08. Django 스태틱, 부트스트랩

[참고 문헌: 점프 투 장고 / 2-07 스태틱](https://wikidocs.net/70804)
[참고 문헌: 점프 투 장고 / 2-08 부트스트랩](https://wikidocs.net/70838)

### 1. Static
지금까지 우리는 게시물의 CRUD(CREATE, READ, UPDATE, DELETE)를 완성하였습니다. 
다만 백엔드 엔지니어라고 하더라도 HTML 태그에 의존한 디자인은 게시판으로 부르기 민망합니다. 
그래서 우리는 css 파일을 템플릿에 적용합니다.

#### 스타일시트, 자바스크립트 파일은 static 폴더에
장고의 템플릿에서 사용하는 css, js, img 파일들은 최상위 폴더 아래 'static'디렉토리 안에서 관리됩니다. 
(템플릿과 마찬가지로 앱 하위 폴더에서도 지원하지만 권장되지 않습니다.)

#### 스타일 시트를 생성하고 적용해보자.

##### forumwithdjango/settings.py
스태틱 관련 설정을 추가합니다.
```python
# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/5.2/howto/static-files/

STATIC_URL = 'static/'
# 프로젝트의 스태틱 파일들의 저장 위치를 설정해주는 리스트 변수입니다.
STATICFILES_DIRS = [
    # BASE_DIR(최상위 폴더)/static 폴더를 지정합니다.
    BASE_DIR / 'static',
]
```

##### forum-with-django/static/css/base.css 생성
static 폴더를 생성하고 base.css 파일을 작성합니다.
```css
body {
    display: flex;
    justify-content: center;
}

.title {
    width:50vw;
}
.title input {
    width:100%
}

.content {
    margin-top:10px;
    width:50vw;
}
.content textarea {
    width:100%;
    height:50vh;
}
```

##### post_form.html
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <!-- CSS 파일은 스크립트 파일과 다르게 head태그 내에 넣는 것이 표준입니다. -->
    <!-- 페이지가 렌더링 되기 전에 스타일이 먼저 적용되어야 그 내용에 맞춰 엘리먼트들이 렌더링되기 때문입니다. -->
    {% load static %}
    <link rel="stylesheet" href="{% static 'css/base.css' %}">
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <form method="POST">
        ...
    </form>
</body>
</html>
```

##### 결과 확인
![스크린샷](/statics/08/08_01.png)
조금 그럴듯 한 게시물 작성 페이지가 완료되었습니다.

---

### 2. Bootstrap
우리가 스타일 시트를 생성하고 적용하였다 하더라도 수많은 인터넷 커뮤니티들에 비하면 너무 빈약합니다.
우리가 프론트엔드, 퍼블리셔는 아니지만 풀스택으로 현재 프로젝트를 완성하려면 기본적인 디자인은 완성해야 합니다.
그래서 우리는 부트스트랩 프레임워크를 사용합니다.

#### 부트스트랩 설치
##### I. static 폴더 내 다운로드
[https://getbootstrap.com/docs/5.3/getting-started/download/](https://getbootstrap.com/docs/5.3/getting-started/download/)
위의 링크에서 파일을 직접 다운로드 합니다.
![스크린샷](/statics/08/08_02.png)

bootstrap-5.3.x-dist\css\bootstrap.min.css 파일을 우리 프로젝트의 static/css 위치에 복사합니다.
![스크린샷](/statics/08/08_03.png)

위에서 설정했던 css를 불러오는 방법처럼 템플릿에 스태틱 태그를 이용해 부트스트랩을 적용합니다.
post_list.html, post_detail.html, post_form.html
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    {% load static %}
    <link rel="stylesheet" href="{% static 'css/bootstrap.min.css' %}">
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    ...
</body>
```

##### II. CDN을 통해 다운로드 없이 불러오기
![스크린샷](/statics/08/08_04.png)

post_list.html, post_detail.html, post_form.html
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <!-- {% load static %} -->
    <!-- <link rel="stylesheet" href="{% static 'css/bootstrap.min.css' %}"> -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.6/dist/css/bootstrap.min.css" rel="stylesheet"
          integrity="sha384-4Q6Gf2aSP4eDXB8Miphtr37CMZZQ5oXLH2yaXMJ2w8e2ZtHTl7GptT4jmndRuHDT" crossorigin="anonymous">
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    ...
</body>
```

##### 결과 확인
![스크린샷](/statics/08/08_05.png)
부트스트랩을 적용했을 뿐인데 기존의 칙칙함을 어느정도 덜어냈습니다.

##### 직접 다운로드와 CDN의 차이
직접 다운로드
- 장점: 
인터넷 연결이 없어도 작동
외부 사이트에 의존하지 않아 항상 작동 (CDN 서버에 장애가 생기는 일은 드물긴 합니다.)
부트스트랩 파일 커스텀 가능
- 단점:
프로젝트 용량에 영향을 줌
부트스트랩에 업데이트가 있을때마다 새로 다운로드가 필요

CDN 방식
- 장점:
설치가 매우 간편하고 약간의 코드 수정만으로 최신버전 업데이트 가능
다른 사이트에서 부트스트랩을 사용한 이용자는 브라우저 캐시를 사용하여 빠르게 로드가 가능
서버 트래픽 관리에 유리
- 단점:
인터넷 연결이 필요함. (다만 웹 어플리케이션이기 때문에 큰 단점이 된다고 보기에는 어려움)
CDN 서버에 장애가 생기면 로딩할 수 없음

두 방식 모두 장단점이 있습니다. 사용하시기 편한 방법을 선택해 사용하셔도 됩니다.
(실무에서는 보통 CDN 방식을 많이 사용한다고 합니다.)

#### 부트스트랩 활용
포스트 리스트를 테이블화 하여 출력해 봅시다.
post_list.html
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.6/dist/css/bootstrap.min.css" rel="stylesheet"
          integrity="sha384-4Q6Gf2aSP4eDXB8Miphtr37CMZZQ5oXLH2yaXMJ2w8e2ZtHTl7GptT4jmndRuHDT" crossorigin="anonymous">
    <meta charset="UTF-8">
    <title>포럼</title>
</head>
<body>
    <div class="container">
    <table class="table table-striped table-hover">
        <thead>
            <tr class="table-secondary">
                <th style="width: 10%">글번호</th>
                <th style="width: 60%">제목</th>
                <th style="width: 15%">작성일</th>
            </tr>
        </thead>
        <tbody class="table-group-divider">
            {% if posts %}
            {% for post in posts %}
            <tr>
                <td>{{ post.pk }}</td>
                <td>
                    <a href="{% url 'forum:post_detail' post.pk %}">{{ post.title }}</a>
                </td>
                <!-- date 템플릿 태그를 사용하여 날짜형식을 변경 -->
                <td>{{ post.created_date|date:"Y/m/d A h:i" }}</td>
            </tr>
            {% endfor %}
            {% endif %}
        </tbody>
    </table>
    <a href="{% url 'forum:post_create' %}" class="btn btn-primary">게시물 작성</a>
</div>
</body>
</html>
```
Django date template filter
[https://docs.djangoproject.com/en/5.2/ref/templates/builtins/#date](https://docs.djangoproject.com/en/5.2/ref/templates/builtins/#date)

##### 결과 확인
![스크린샷](/statics/08/08_06.png)
깔끔해진 post_list를 볼 수 있다.

위의 html 코드를 보면 태그들에 클래스명이 container, table, table-group-divider 같이 설정되어있는 것을 볼 수 있는데 이처럼 부트스트랩은 해당 엘리먼트에 특정한 클래스명을 입력하여 해당하는 스타일을 적용합니다.
공식문서의 table 문서
[https://getbootstrap.com/docs/5.3/content/tables/](https://getbootstrap.com/docs/5.3/content/tables/)

##### 나머지 post_detail과 post_form에도 부트스트랩을 적용하자

post_form.html
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.6/dist/css/bootstrap.min.css" rel="stylesheet"
          integrity="sha384-4Q6Gf2aSP4eDXB8Miphtr37CMZZQ5oXLH2yaXMJ2w8e2ZtHTl7GptT4jmndRuHDT" crossorigin="anonymous">
    <meta charset="UTF-8">
    <title>게시물 작성</title>
</head>
<body>
    <div class="container">
        <form method="POST" class="my-5">
            <h2 class="border-bottom pb-3">게시물 작성</h2>
            {% csrf_token %}
            {% if form.errors %}
                <div class="alert alert-danger" role="alert">
                {% for field in form %}
                {% if field.errors %}
                    <strong>{{ field.label }}</strong>
                    {{ field.errors }}
                {% endif %}
                {% endfor %}
                </div>
            {% endif %}
            <div class="my-3">
                <label class="form-label" for="title">제목</label>
                <input type="text" id="title" name="title" class="form-control mb-3" required
                       value="{{ form.title.value|default_if_none:'' }}">
            </div>
            <div class="mb-3">
                <label class="form-label" for="content">내용</label>
                <textarea id="content" name="content" class="form-control" required
                          style="min-height: 50vh">{{ form.content.value|default_if_none:'' }}</textarea>
            </div>
            <input type="submit" class="btn btn-primary">
        </form>
    </div>
</body>
</html>
```

##### 결과 확인
![스크린샷](/statics/08/08_07.png)
![스크린샷](/statics/08/08_08.png)

post_detail.html
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.6/dist/css/bootstrap.min.css" rel="stylesheet"
          integrity="sha384-4Q6Gf2aSP4eDXB8Miphtr37CMZZQ5oXLH2yaXMJ2w8e2ZtHTl7GptT4jmndRuHDT" crossorigin="anonymous">
    <meta charset="UTF-8">
    <!-- 탭 명 지정 -->
    <title>Forum: {{ post.title }}</title>
</head>
<body>
    <div class="container">
        <h2 class="border-bottom py-2">{{ post.title }}</h2>
        <div class="card border-secondary">
            <div class="card-body" style="min-height: 30vh">
                <!-- 게시물 content의 들여쓰기와 줄바꿈을 출력하기 위해 style 적용 -->
                <div class="card-text" style="white-space: pre-wrap">{{ post.content }}</div>
            </div>
            <div class="card-footer">
                <div class="card-text text-end">
                    <a>작성일: </a>
                    {{ post.created_date|date:"Y/m/d A h:i" }}
                </div>
                <div class="card-text text-end">
                    <a>수정일: </a>
                    <!-- with 템플릿 태그를 사용하여 변수를 생성하고 수정일을 삽입합니다. -->
                    {% with temp=post.updated_date|date:"Y/m/d A h:i" %}
                        <!-- default 템플릿 필터를 통해 값이 있다면(수정 이력 존재) 그대로 출력하고
                         값이 없었다면 "수정 사항 없음" 텍스트를 temp 변수에 삽입 후 출력합니다. -->
                        {{ temp|default:"수정 사항 없음" }}
                    {% endwith %}
                </div>
            </div>
        </div>
        <div class="row">
            <div class="col">
                <a href="{% url 'forum:post_list' %}"
                   class="btn btn-secondary mt-2 justify-content-around">글 목록</a>
            </div>
            <div class="col text-end">
                <a href="{% url 'forum:post_update' post.pk %}"
                   class="btn btn-outline-success mt-2">수정</a>

                <a href="javascript:void(0)"
                   class="btn btn-danger mt-2 delete-link"
                   data-url="{% url 'forum:post_delete' post.pk %}">삭제</a>
            </div>
        </div>
    </div>

    <script type="text/javascript">
        ...
    </script>
</body>
</html>
```

##### 결과 확인
![스크린샷](/statics/08/08_09.png)
공식문서의 card 문서
[https://getbootstrap.com/docs/5.3/components/card/](https://getbootstrap.com/docs/5.3/components/card/)

---

### 3. 스크립트 분리
post_detail.html 의 스크립트를 .js파일로 분리시켜 봅시다.

/static/js/post_detail.js 생성 
(post_detail.html에서만 사용되며 Post, Comment의 수정과 삭제를 담당할 예정입니다.)
```javascript
const deleteLink = document.querySelectorAll('.delete-link');
deleteLink.forEach(function(element) {
    element.addEventListener('click', function() {
        if(confirm("정말 삭제하시겠습니까?")) {
            //location.href = element.getAttribute('data-url');
            location.href = this.dataset.url;
        }
    })
})
```

post_detail.html
```html
<!DOCTYPE html>
<html lang="ko">
<head>
    ...
</head>
<body>
    <div class="container">
        ...
    </div>

    {% load static %}
    <script src="{% static 'js/post_detail.js' %}"></script>
</body>
</html>
```