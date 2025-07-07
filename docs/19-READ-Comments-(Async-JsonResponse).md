## 19. 비동기 요청으로 댓글 조회
이제 17차시에서 진행했던 SSR 댓글 조회를 ajax, fetch를 이용하여 클라이언트 사이드 렌더링으로 바꿔 봅시다.

### Django 서버측 작업

#### urls\.py
```python
urlpatterns = [
    ...

    path('<int:post_pk>/comments/new/', comment_views.comment_create, name='comment_create'),
    path('<int:post_pk>/comments/',     comment_views.comment_list,   name='comment_list'),
]
```

#### Django의 쿼리셋 직렬화(쿼리셋<->JSON)
ajax, axios, fetch 어떤 비동기 요청 API를 사용하더라도 JSON 형식으로 전달 받을 것입니다.
우리의 Comments 쿼리셋 데이터를 JSON으로 변환하여 클라이언트에게 전달해 봅시다.

forum/views/comment_views.py
```python
from django.core import serializers
from django.http import HttpResponseNotAllowed, JsonResponse, HttpResponse
from django.shortcuts import render, get_object_or_404, redirect

from forum.forms import CommentForm
from forum.models import *

def comment_create(request, post_pk):
    ...

def comment_list(request, post_pk):
    post = get_object_or_404(Post, pk=post_pk)

    # 1. JsonResponse(list(data))
        # 1. 파이썬 dictionary나 list, 기본 타입만 직렬화 가능
            # 쿼리셋을 바로 직렬화 하진 못하고 .values()를 사용하여 dictionary/list로 바꿔 전송
        # 2. Json 단일 response로써 content_type 지정 불필요
        # 3. Response 할 때 직렬화가 이루어짐

    comments = post.comment_set.all().values()
    return JsonResponse(list(comments), safe=False)
    # JsonResponse는 기본적으로 dict만 JSON으로 리턴할 수 있습니다. 다만 safe옵션을 False로 설정하면
    # list같은 다른 타입(tuple, string...)도 리턴할 수 있도록 해줍니다.
    # 이와 같은 이유로 위의 코드에서 list()로 comments 쿼리셋을 감싸줄 필요는 없지만,
    # 쿼리셋을 JsonResponse에 담으면 내부적으로 list가 되기 때문에 명시적으로 list()를 감싸준 것입니다.

    # ------------------------------------------------------------------------

    # 2. django.core.serializers + HttpResponse
        # 1. Django 내장 시리얼라이저를 이용해 쿼리셋이나 모델 인스턴스를 JSON 문자열로 반환한다.
        # 2. HttpResponse를 사용하다 보니 content_type 지정 필요
        # 3. 이미 JSON 문자열로 변환 했기 때문에 JsonResponse로 전송시 이중 직렬화가 발생함으로 사용 X

    json_comments = serializers.serialize('json', post.comment_set.all())
    return HttpResponse(json_comments, content_type='application/json')
```
코드에서 본 것 처럼 직렬화 방법은 두가지가 존재합니다. 
1번 방법을 사용할 예정인데 결과 값(/posts/**/comments/ 의 response)을 보며 설명드리겠습니다.

먼저 1번 방법의 결과값 입니다.
```JSON
[
  {
    "id": 3,
    "author_id": 1,
    "post_id": 322,
    "content": "1234",
    "created_date": "2025-06-26T08:12:18.912Z"
  },
  ...
]
```

2번 방법의 결과값 입니다.
```JSON
[
  {
    "model": "forum.comment",
    "pk": 3,
    "fields": {
      "author": 1,
      "post": 322,
      "content": "1234",
      "created_date": "2025-06-26T08:12:18.912Z"
    }
  },
  ...
]
```

리스폰스를 보면 알 수 있듯이 2번 방법은 결과가 복잡합니다.

django.core의 시리얼라이저로 생성된 JSON구조는 Django의 내부식(모델, pk, fields)을 따르고 있어서 프론트엔드에서 다루기 불편합니다. 
1번의 방법은 content의 value값에 접근하기 위해 data['content']로 접근하면 되지만 2번은 data['fields']['content']의 중첩 구조로 객체에 접근해야 합니다. 
또한 어떠한 모델을 따르고 있는지에 대한 정보는 필요하지 않습니다.

2번은 특별히 복잡한 관계나 외래키 구조를 JSON에 투영해야 할 때만 고려할 수 있는 방법이고 대부분의 상황에서는 JsonResponse를 통하여 프론트엔드에서도 간편하게 데이터에 접근할 수 있도록 하는 것이 좋습니다.

#### + values() 옵션
```python
def comment_list(request, post_pk):
    post = get_object_or_404(Post, pk=post_pk)

    # 원하는 필드의 선택이 가능합니다. 다만 필드명은 변경이 불가능합니다.
    comments = post.comment_set.all().values(
        'pk', 'author__pk', 'author__username', 'author__nickname', 'content', 'created_date')

    return JsonResponse(list(custom_field), safe=False)

    # 필드명을 변경하려면 새로운 파이썬 리스트를 생성하고 반복문을 통해 커스텀 필드명을 추가하거나
    custom_field = []
    for comment in comments:
        custom_field.append({
            'pk': comment['pk'],
            'author_pk': comment['author__pk'],
            'username': comment['author__username'],
            'nickname': comment['author__nickname'],
            'content': comment['content'],
            'created_date': comment['created_date'],
        })

    # annotate()를 사용하는 방법이 있습니다.
    # from django.db.models import F
    custom_field = post.comment_set.annotate(auth_pk=F('author__pk'),
                                            username=F('author__username'),
                                            nickname=F('author__nickname')).values(
        'pk', 'auth_pk', 'username', 'nickname', 'content', 'created_date')

    return JsonResponse(list(custom_field), safe=False)
```
```
[
  {
    "pk": 3,
    "author__pk": 1,
    "author__username": "asdf1234",
    "author__nickname": "nick",
    "content": "1234",
    "created_date": "2025-06-26T08:12:18.912Z"
  }
  // 커스텀 필드명 (파이썬 리스트 생성)
  {
    "pk": 17,
    "author_pk": 1,
    "username": "asdf1234",
    "nickname": "nick",
    "content": "1234",
    "created_date": "2025-06-30T05:51:11.092Z"
  },
  // 커스텀 필드명 (annotate() 사용)
  {
    "pk": 17,
    "auth_pk": 1,
    "username": "asdf1234",
    "nickname": "nick",
    "content": "1234",
    "created_date": "2025-06-30T05:51:11.092Z"
  },
]
```

#### + created_date 가공
여기서 문제가 하나 생깁니다. 지금까지는 템플릿 태그 (|date:"Y/m/d h:i")를 통해 날짜 형식을 보기좋게 변경했습니다. 그러나 이제는 JSON 문자열로 받아와 자바스크립트 내에서 템플릿 태그의 사용 없이 처리해야 하는데 날짜의 형식이 "2025-06-26T08:12:18.912Z" 형태의 ISO 8601 형식으로 반환됩니다.

결국 우리 프로젝트는 개발 트렌드, 유지보수, 확장성을 위해서 자바스크립트에서 날짜 객체로 변환하여 형식을 변경할 예정이지만 뷰에서 형식을 변환해 전송해주는 실습을 진행해보고 넘어갑시다.

```python
def comment_list(request, post_pk):
    post = get_object_or_404(Post, pk=post_pk)
    comments = post.comment_set.all().values(
        'pk', 'author__pk', 'author__username', 'author__nickname', 'content', 'created_date')

    for comment in comments:
        comment['created_date'] = timezone.localtime(comment['created_date']).strftime('%Y/%m/%d %p %#I:%M')

    return JsonResponse(list(date_result), safe=False)
```
```
[
  {
    "pk": 3,
    "author__pk": 1,
    "author__username": "asdf1234",
    "author__nickname": "nick",
    "content": "1234",
    "created_date": "2025/06/26 PM 5:12"
  }
]
```
---

### post_detail 에서 비동기 요청하기

#### I. ajax
ajax는 jQuery의 기능 중 하나입니다. 템플릿에 CDN을 사용하여 jQuery를 불러와 봅시다.
[https://releases.jquery.com/jquery/](https://releases.jquery.com/jquery/)
```html
{% extends 'base.html' %}
{% block title %}
Forum: {{ post.title }}
{% endblock title %}

{% block body %}
<div class="container">
    ... 
</div>

<script src="https://code.jquery.com/jquery-3.7.1.js"  
integrity="sha256-eKhayi8LEQwp4NKxN+CfCh+3qOVUtJn3QNZ0TciWLP4=" crossorigin="anonymous"></script>

{% load static %}
<script src="{% static 'js/post_detail.js' %}?{% now 'U' %}"></script>
<script type="text/javascript">
    const loginUrl = "{% url 'member:login' %}";
    // post_detail에서 작성 할 함수를 불러옵니다.
    requestComments({{ post.pk }})
</script>
{% endblock body %}
```
##### post_detail.js
```javascript
function requestComments(post_pk) {
    $.ajax({
        url: `/posts/${post_pk}/comments/`,
        method: 'GET',
        dataType: 'json',
        success: function(data) {
            console.log(data);
        }
    })
}
```
![스크린샷](/statics/19/19_01.png)

#### II. fetch
ajax(jQuery)는 이제 레거시의 영역에 들어가고 새로운 프로젝트의 비동기 요청 방식으로는 사용되지 않습니다.
대신 fetch나 Axios같은 더 현대적인 도구를 사용하는데요. 그 이유를 간략하게 알아보겠습니다.

##### fetch의 장점
1. 네이티브 지원: fetch는 모든 최신 브라우저에 내장되어 있어 jQuery같은 별도 라이브러리 설치가 필요하지 않습니다.
  성능에도 영향을 줍니다.
2. 간결한 문법: Promise를 기반하여 async/await와 함께 사용 시 코드를 동기 처리처럼 작성할 수 있어 간단합니다.
3. 최근에는 React, Vue, Angular 같은 프론트엔드 프레임워크를 사용하며 이들은 jQuery를 필요로 하지 않습니다.

##### 01. fetch 기본 사용법
```javascript
function requestComments(post_pk) {
  fetch(`/posts/${post_pk}/comments/`)
        .then(response => response.json()) //응답을 json으로 변환해야 합니다.
        .then(data => console.log(data)) //변환된 데이터를 콘솔에 출력합니다.
        .catch(error => console.log(error)); //에러 처리
}
```
![스크린샷](/statics/19/19_02.png)

##### 02. async/await 사용하기
```html
{% block body %}
...
<script src="{% static 'js/post_detail.js' %}?{% now 'U' %}"></script>
<script type="text/javascript">
    const loginUrl = "{% url 'member:login' %}";
    // requestComments({{ post.pk }})
    getComments({{ post.pk }})
</script>
{% endblock body %}
```
```javascript
async function getComments(post_pk) {
    try {
        const response = await fetch(`/posts/${post_pk}/comments/`);
        if (!response.ok) throw new Error('Network response was not ok');
        const data = await response.json();
        console.log(data);
    } catch (error) {
        console.error(`Fetch: Failed to fetch: ${error}`);
    }
}
```
![스크린샷](/statics/19/19_03.png)

---

### DOM 객체 출력하기
이제 전달받은 JSON 내부의 데이터를 통해 자바스크립트에서 HTML DOM 객체를 생성하고 출력해 봅시다.
기존의 \<div class="comments'>내부의 내용들을 주석처리 합니다.

post_detail.html
```html
{% extends 'base.html' %}
{% block title %}
Forum: {{ post.title }}
{% endblock title %}

{% block body %}
<div class="container">
    ...
    <hr>
    <div class="comments">
        <!--
        {% for comment in post.comment_set.all %}
        <div class="card mb-3">
            <div class="card-header">
                {{ comment.author }}
            </div>
            <div class="card-body">
                <div class="card-text" style="white-space: pre-wrap">{{ comment.content }}</div>
            </div>
            <div class="card-footer" style="font-size: small">
                <div>작성일: {{ comment.created_date|date:"Y/m/d A h:i" }}</div>
                {% with temp=comment.updated_date|date:"Y/m/d A h:i" %}
                    <div>수정일: {{ temp|default:"수정 사항 없음" }}</div>
                {% endwith %}
            </div>
        </div>
        {% endfor %}
        -->
    </div>
    ...
</div>

<script src="https://code.jquery.com/jquery-3.7.1.js"  integrity="sha256-eKhayi8LEQwp4NKxN+CfCh+3qOVUtJn3QNZ0TciWLP4="
        crossorigin="anonymous"></script>
{% load static %}
<script src="{% static 'js/post_detail.js' %}?{% now 'U' %}"></script>
<script type="text/javascript">
    const loginUrl = "{% url 'member:login' %}";
    // jQuery, fetch 어떤 것을 사용해도 상관없습니다.
    requestCommentsAjax({{ post.pk }})
    requestCommentsFetch({{ post.pk }})
    getComments({{ post.pk }})
</script>
{% endblock body %}
```

#### I. document.createElement를 이용해 DOM 객체 생성
post_detail.js
```javascript
// 1. ajax를 이용한 요청
function requestCommentsAjax(post_pk) {
    $.ajax({
        url: `/posts/${post_pk}/comments/`,
        method: 'GET',
        dataType: 'json',
        success: function(data) {
            console.log(data);
            renderComments(data)
        }
    })
}

// 2. fetch 기본 사용법
function requestCommentsFetch(post_pk) {
    fetch(`/posts/${post_pk}/comments/`)
        .then(response => response.json())
        // async/await 없이 사용 시 fetch는 비동기 처리이기 때문에 then 내부에서만 해당 변수 사용가능 합니다.
        .then(data => {
            console.log(data);
            renderComments(data);
        })
        .catch(error => console.log(error));
}

// 3. async/await를 이용한 코드 동기 처리 ('동기 처럼'쓰는 것이지 실제로는 비동기 처리입니다.)
async function getComments(post_pk) {
    try {
        const response = await fetch(`/posts/${post_pk}/comments/`);
        if (!response.ok) throw new Error('Network response was not ok');
        const data = await response.json();
        console.log(data);
        renderComments(data)
    } catch (error) {
        console.error(`Fetch: Failed to fetch: ${error}`);
    }
}

// 자바스크립트의 Date 객체를 사용하여 ISO형식의 날짜를 보기 편한 로컬타임으로 변경하는 함수
function formatISODate(date) {
    const dateObj = new Date(date);
    return dateObj.toLocaleString('ko-KR', {
        year: 'numeric', month: '2-digit', day: '2-digit',
        hour: '2-digit', minute: '2-digit',
    });
}

function renderComments(comments) {
    // <div class="comments">를 제외한 하위 태그 모두 createElement합니다.
    const commentsDiv = document.querySelector('.comments')
    // 전달받은 JSON의 리스트를 돌며
    comments.forEach(comment => {
        const cardDiv = commentsDiv.appendChild(document.createElement('div'));
        cardDiv.className = 'card mb-3';

        const cardHeader = cardDiv.appendChild(document.createElement('div'));
        cardHeader.classList.add('card-header');
        cardHeader.append(`${comment['author__nickname']} (${comment['author__username']})`);

        const cardBody = cardDiv.appendChild(document.createElement('div'));
        cardBody.classList.add('card-body');
        const bodyText = cardBody.appendChild(document.createElement('div'));
        bodyText.style.whiteSpace = 'pre-wrap';
        bodyText.append(comment['content']);

        const cardFooter = cardDiv.appendChild(document.createElement('div'));
        cardFooter.classList.add('card-footer');
        cardFooter.style.fontSize = 'small';
        const created_date = cardFooter.appendChild(document.createElement('div'));
        const formattedDate = formatISODate(comment['created_date']);
        created_date.append(`작성일: ${formattedDate}`);

        // 부모, 자식 요소 검색 매서드
        // 노드는 엘리먼트의 상위 개념입니다.
        // 노드: DOM 계층구조 내의 어떠한 요소라도 가리킬 수 있습니다.
        // 엘리먼트: 노드의 하위 개념으로써 html tag 요소(엘리먼트)를 가리킵니다.

        // 1. parentNode (childNodes)
        // html 태그 밖도 접근할 수 있습니다.
        // cNodes 의 경우 <>태그로 감싸있지 않은 플레인 텍스트, 주석 등도 검색됩니다.

        // 2. parentElement (children)
        // html 태그 밖은 접근하지 못합니다.
        // 태그로 감싸져 있지 않은 플레인 텍스트의 경우 검색되지 않습니다.

        // 자바스크립트의 요소 추가 매서드
        // 1. prepend(삽입 노드)
        // 자식 노드들 중 맨 앞에 삽입합니다.

        // 2. append(삽입 노드 or DOM String)
        // 노드 객체 뿐만 아니라 문자열 또한 추가할 수 있습니다.
        // 두개 이상의 자식 엘리먼트를 추가할 수 있습니다.

        // 3. appendChild(삽입 노드)
        // 기본적으로 맨 뒤에 삽입합니다.
        // append()에 비해 부족하다고 생각할 수 있지만 리턴값을 지원합니다.
        // 만약 삽입된 객체를 변수에 저장하여 재사용 해야 한다면 유용합니다.
        //const childDiv = parentDiv.appendChild(document.createElement('div'));

        // 4. insertBefore(삽입 노드, 기준 노드)
        // appendChild 같이 리턴값을 전달 해 주지만 전달받는 인자가 두개입니다.
        // 자식 노드들 중 기준 노드 바로 앞에 노드를 삽입합니다.
        // parentDiv.insertBefore(child2(기준), child1);
    })
}
```
##### 결과 확인
![스크린샷](/statics/19/19_04.png)

#### II. \<template>와 cloneNode()를 이용하기
위의 코드처럼 createElement를 사용하는 방법도 있지만 우리 프로젝트처럼 얼마 안되는 comments div태그라도 스타일과 클래스 명을 적용하면 코드가 길어져 가독성이 떨어지고 유지보수 시 알아볼 수 없게 됩니다.
당연히 웹 프레임워크 (프론트엔드)를 사용하면 좋겠지만 우리는 Django의 템플릿을 사용하고 있기 때문에 뼈대를 작성하고 내용을 채워넣는 \<template>를 사용해 봅시다.
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
        ...
    </div>
    <div class="row">
        ...
    </div>
    <hr>
    <div class="comments">
        <!--
        html의 template 태그를 사용해 클래스명과 스타일 속성이 적용된 템플릿을 생성합니다.
        <div hidden>을 사용하는 것도 가능하지만 둘은 차이점이 존재합니다.
        <template>
          렌더링 시점에 DOM에 포함되지 않습니다.
          내부 콘텐츠가 실제 DOM 트리의 일부가 아닙니다. 렌더링, 스크립트, 스타일 모두 적용되지 않습니다.
            ('틀'을 제공하는 것 입니다.)
          JS로 동적 복제/삽입에 최적화, 재사용을 위해 만들어진 태그입니다.
          어떤 위치에서도 사용 가능하고 <div><tr></tr><div>같은 부모가 필요한 태그도 자유롭게 사용 가능합니다.
          ('template').content.cloneNode(true) 로 접근합니다.
        <div hidden>
          hidden으로 가려져 있을 뿐 렌더링 시점에 DOM에 포함됩니다.
          내부 콘텐츠가 실제 DOM 트리의 일부입니다. 보이지 않을 뿐 스크립트, 스타일 모두 적용됩니다.
          div는 재사용을 위한 태그가 아닙니다. 접근성과 성능 저하의 우려가 있습니다.
          <div><tr></tr></div> 당연하게도 사용이 불가능합니다.
          ('div').cloneNode(true) 로 접근합니다.
        -->
        <template id="comment-template">
            <div class="card mb-3">
                <div class="card-header"></div>
                <div class="card-body">
                    <div class="card-text" style="white-space: pre-wrap"></div>
                </div>
                <div class="card-footer" style="font-size: small">
                    <div class="comment-created-date"></div>
                    <div class="comment-modified-date"></div>
                </div>
            </div>
        </template>
    </div>

    <form action="{% url 'forum:comment_create' post.pk %}" method="POST" class="my-4">
        ...
    </form>
</div>

<script src="https://code.jquery.com/jquery-3.7.1.js"  integrity="sha256-eKhayi8LEQwp4NKxN+CfCh+3qOVUtJn3QNZ0TciWLP4="
        crossorigin="anonymous"></script>
{% load static %}
<script src="{% static 'js/post_detail.js' %}?{% now 'U' %}"></script>
<script type="text/javascript">
    const loginUrl = "{% url 'member:login' %}";
    getComments({{ post.pk }})
</script>
{% endblock body %}
```
##### post_detail.js
```javascript
function renderComments(comments) {
    const commentsDiv = document.querySelector('.comments')
    comments.forEach(comment => {
        const cardDiv = document.querySelector('#comment-template').content.cloneNode(true);

        cardDiv.querySelector('.card-header').append(`${comment['author__nickname']} (${comment['author__username']})`);
        cardDiv.querySelector('.card-body .card-text').append(comment['content']);
        const formattedDate = formatISODate(comment['created_date']);
        cardDiv.querySelector('.card-footer .comment-created-date').append(`작성일: ${formattedDate}`);

        commentsDiv.appendChild(cardDiv);
    })
}
```
##### 결과 확인
![스크린샷](/statics/19/19_05.png)

#### + III. 템플릿 리터럴 이용하기
```javascript
async function getComments(post_pk) {
    try {
        const response = await fetch(`/posts/${post_pk}/comments/`);
        if (!response.ok) throw new Error('Network response was not ok');
        const data = await response.json();
        console.log(data);
        // renderComments(data)
        templateLiteral(data)
    } catch (error) {
        console.error(`Fetch: Failed to fetch: ${error}`);
    }
}

function templateLiteral(comments) {
    // 문자열을 한번에 innerHTML에 입력하기 위해 forEach 대신 map을 사용했습니다.
    const innerHTMLTest = comments.map(comment => `
        <div class="card mb-3">
            <div class="card-header">${comment['author__nickname']}</div>
            <div class="card-body">
                <div class="card-text" style="white-space: pre-wrap">${comment['content']}</div>
            </div>
            <div class="card-footer" style="font-size: small">
                <div class="comment-created-date">${comment['created_date']}</div>
                <div class="comment-modified-date"></div>
            </div>
        </div>
    `).join('');
    // innerHTML에는 하나의 문자열만 넣을 수 있음으로 join('')을 통해 문자열 배열을 하나의 문자열로 합쳐줍니다.
    document.querySelector('.test').innerHTML = innerHTMLTest;
}
```
html 코드를 백틱(`)내부에 작성하고 innerHTML를 통해 통째로 집어 넣는 방법입니다. 코드를 한눈에 알아보기에 편합니다.
일반적인 텍스트 content에서는 정상작동을 하는 모습을 보여줍니다. 다만 게시판 이용자가 아래와 같은 댓글을 작성했다면?
![스크린샷](/statics/19/19_06.png)

아래 스크린샷과 같이 자바스크립트 코드를 통한 XSS 공격, 허용되지 않은 이미지를 작성할 수 있게됩니다.
![스크린샷](/statics/19/19_07.png)
![스크린샷](/statics/19/19_08.png)

innerHTML을 사용시에는 반드시 사용자 입력을 HTML 이스케이프 해야 합니다.
당연하게도 우리가 사용했던 createElement방법과 \<template>방법도 사용할 이유가 없었을 뿐 append, createElement나 innerText, textContent가 아닌 innerHTML를 사용하면 같은 문제가 발생합니다.

##### HTML escape 예시
```javascript
function htmlEscape(str) {
    return str.replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;');
}

function templateLiteral(comments) {
    const innerHTMLTest = comments.map(comment => `
        <div class="card mb-3">
            <div class="card-header">${htmlEscape(comment['author__nickname'])}</div>
            <div class="card-body">
                <div class="card-text" style="white-space: pre-wrap">${htmlEscape(comment['content'])}</div>
            </div>
            <div class="card-footer" style="font-size: small">
                <div class="comment-created-date">${htmlEscape(comment['created_date'])}</div>
                <div class="comment-modified-date"></div>
            </div>
        </div>
    `).join('');
    document.querySelector('.test').innerHTML = innerHTMLTest;
}
```

##### 결과 확인
![스크린샷](/statics/19/19_09.png)


#### + IV. Django 템플릿 엔진 이용하기
templates/forum/comment_list.html
```html
{% for comment in post.comment_set.all %}
<div class="card mb-3">
    <div class="card-header">
        {{ comment.author }}
    </div>
    <div class="card-body">
        <div class="card-text" style="white-space: pre-wrap">{{ comment.content }}</div>
    </div>
    <div class="card-footer" style="font-size: small">
        <div>작성일: {{ comment.created_date|date:"Y/m/d A h:i" }}</div>

        {% with temp=comment.updated_date|date:"Y/m/d A h:i" %}
        <div>수정일: {{ temp|default:"수정 사항 없음" }}</div>
        {% endwith %}
    </div>
</div>
{% endfor %}
```
기존의 SSR 코드를 새로운 Django 템플릿으로 옮겨줍니다.

comment_views.py
```python
def comment_list(request, post_pk):
    post = get_object_or_404(Post, pk=post_pk)

    # render_to_string
    # 기존의 render 메서드와 다르게 HttpResponse객체를 반환하지 않고 문자열을 반환합니다.
    # 문자열: 템플릿 파일을 context를 이용하여 렌더링 하고 HTML(or Text)문자열로 반환하는 메서드 입니다.

    # 현 프로젝트 처럼 클라이언트의 비동기 요청 시 HTML조각을 서버에서 렌더링 하여 전송해 주거나
    # 이메일, 알림 등에서 템플릿을 기반하여 사용자에게 전달할 메세지를 만들 때,
    # 문자열 파일(txt, js. css)을 동적으로 생성 후 전달할 때 사용합니다.
    html = render_to_string('forum/comment_list.html', {'post': post})
    return HttpResponse(html)
```

post_list.js
```javascript
async function loadComments(post_pk) {
    try {
        const response = await fetch(`/posts/${post_pk}/comments/`);
        if (!response.pk) throw new Error("Error");
        const htmlData = await response.text();
        renderComments(htmlData);
    } catch(error) {
        console.log(error);
    }
}

function renderComments(htmlData) {
    document.querySelector('.test').innerHTML = htmlData;
}
```

##### 결과 확인
![스크린샷](/statics/19/19_10.png)

innerHTML을 사용하였는데 XSS공격이 작동하지 않습니다. Django 템플릿 엔진은 자동 이스케이프를 지원하기 때문입니다.
[https://docs.djangoproject.com/en/5.2/ref/templates/builtins/#escape](https://docs.djangoproject.com/en/5.2/ref/templates/builtins/#escape)
모든 변수를 출력 시 HTML 특수 문자를 안전한 엔티티로 변환하여(스트링 값) 처리합니다. (&lt, &gt, &amp...)
\<script>alert(1)\</script> => \&lt;script\&gt;alert(1)\&lt;/script\&gt;

만약 신뢰된 데이터(ex. 관리자 전용 메세지 html 태그)를 포함하고 싶다면:
```python
    content = "<img src='https://uhf.microsoft.com/images/microsoft/RE1Mu3b.png'>"

    from django.utils.safestring import mark_safe
    html = render_to_string('forum/comment_list.html', {'post': post, 'content': mark_safe(content)})

    from django.utils.safestring import SafeString
    html = render_to_string('forum/comment_list.html', {'post': post, 'content': SafeString(content)})
```
혹은 템플릿 내에서:
```html
{% for comment in post.comment_set.all %}
<div class="card mb-3">
    ...
    <div class="card-body">
        <!--1. autoescape off -->
        {% autoescape off %}
        <div class="card-text" style="white-space: pre-wrap">{{ comment.content }}</div>
        {% endautoescape %}

        <!--2. 'safe' template filter -->
        <div class="card-text" style="white-space: pre-wrap">{{ comment.content|safe }}</div>
    </div>
    ...
</div>
{% endfor %}
```

##### Django template를 사용한 비동기 처리는 권장되는 방법인가
장고 템플릿을 활용하는 방법도 바닐라 장고를 사용한 웹 어플리케이션에서는 좋은 방법이 될 수 있습니다.

하지만 최근에는 REST API가 표준으로 자리잡으며 서버와 클라이언트의 역할을 명확히 분리하게 되었습니다.
서버는 데이터(JSON 등)만 제공하고 클라이언트는 데이터를 받아 화면을 렌더링 하는 서버와 클라이언트의 역할 분리가 중요합니다.

장점으로 프론트/백 엔드가 독립적인 개발/배포가 가능하며 (유지보수 까지)
렌더링시 서버의 자원을 사용하지 않고 클라이언트 디바이스에서 처리,
SPA(Single Page Application)와 같은 현대적 UI에 적합하고
복잡한 동적 UI/UX, 사용자와 웹 어플리케이션 간 상호작용,
다양한 클라이언트 디바이스(웹, 모바일)에서 같은 API를 사용할 수 있습니다.
 
장고만으로 프로젝트를 개발한다면 문제가 되지 않겠지만 백/프론트를 서로 다른 포트를 개방하여 각자의 프레임워크로 개발하고 API를 통해 정보를 전달하는 최근의 프로젝트 개발 경향과는 맞지 않는다는 단점이 있습니다.