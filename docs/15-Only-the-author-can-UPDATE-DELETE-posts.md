## 15. 작성자가 아닌 사용자의 수정과 삭제 제한

### 뷰에서 제한하기

#### forum/views.py
```python
...
from django.http import HttpResponseForbidden

def post_update(request, pk):
    post = get_object_or_404(Post, pk=pk)

    if request.user != post.author:
        # HttpResponseForbidden (HTTP 403 Forbidden)
        # 주로 사용자가 서버에 접근할 권한이 없을 때 (접근은 가능하나 허가되지 않은 리소스에 접근)
        # 사용하는 오류 HTTP리스폰스 입니다.

        # Django의 대표적인 오류 응답 메서드
        # HTTPResponseBadRequest  400  잘못된 데이터 전송(파라미터 누락, 타입 오류)
        # HTTPResponseNotFound    404  존재하지 않는 페이지/리소스 요청 시
        # HTTPResponseNotAllowed  405  지원하지 않는 HTTP 메서드 요청 시
        # HTTPResponseGone        410  이전에는 존재했지만 리소스가 영구적으로 삭제됐을 떄
        # HTTPResponseServerError 500  서버 내부 오류 발생 시
        return HttpResponseForbidden("다른 사용자의 게시물은 수정하실 수 없습니다.")

    if request.method == 'GET':
        ...
    elif request.method == 'POST':
        ...

def post_delete(request, pk):
    post = get_object_or_404(Post, pk=pk)
    if request.user != post.author:
        return HttpResponseForbidden("다른 사용자의 게시물은 삭제하실 수 없습니다.")

    post.delete()

    return redirect('forum:post_list')
```
로그인 한 사용자가 작성하지 않은 게시물의 수정 또는 삭제 버튼을 클릭해 봅시다.
##### 결과 확인
![스크린샷](/statics/15/15_01.png)
![스크린샷](/statics/15/15_02.png)

---

### 템플릿에서 제한하기
현재 자신의 게시물이 아니더라도 수정과 삭제 버튼이 사용자에게 표시되고 있습니다. 그리고 수정이나 삭제에 대한 요청을 보내면 서버에서 403 에러를 리스폰스 해주고 있습니다.
다만 UX의 관점에서 보았을 때 사용자에게 권한이 없는 기능(다른 사용자의 게시물 수정, 삭제)을 보여주고 실제 동작하는 버튼으로 생성했다면(결과가 어떻더라도) 이것은 잘못된 UX 디자인 입니다.

이러한 이유로 로그인 한 사용자의 게시물이 아니라면 수정과 삭제 버튼을 표시하지 않도록 합시다.

#### post_detail.html
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
        <div class="col">
            <a href="{% url 'forum:post_list' %}"
               class="btn btn-secondary mt-2 justify-content-around">글 목록</a>
        </div>
        <div class="col text-end">
            <!--뷰와 똑같이 request.user를 이용해 세션의 로그인된 사용자 객체를 불러와 비교합니다.-->
            <!--글 목록 버튼이 왼쪽 정렬이기 때문에 <dic class="col">까지 if문 안에 포함시켜 
                col 자체를 출력하지 않아도 괜찮습니다.-->
            {% if post.author == request.user %}
            <a href="{% url 'forum:post_update' post.pk %}"
               class="btn btn-outline-success mt-2">수정</a>

            <a href="javascript:void(0)"
               class="btn btn-danger mt-2 delete-link"
               data-url="{% url 'forum:post_delete' post.pk %}">삭제</a>
            {% endif %}
        </div>
    </div>
</div>
```

##### 결과 확인
![스크린샷](/statics/15/15_03.png)
![스크린샷](/statics/15/15_04.png)

---

### 비로그인 사용자에 대한 접근 제한
이전 차시에서 게시물 생성의 경우 @login_required를 통해 비로그인 사용자의 게시물 작성 페이지 접근을 제한하고 로그인 페이지로 안내했습니다.
다만 수정과 삭제의 경우 템플릿에서 게시물의 작성자가 아니라면(비로그인 사용자 포함) 수정과 삭제 버튼을 표시하지 않고 있습니다.
실제로 비로그인 상태에서 게시물에 접근해 보아도 html 링크(버튼) 엘리먼트가 생성되어 있지 않아 웹 어플리케이션 상에서 접근 자체가 불가능 합니다.

그러나 /posts/\<post_pk>/update/ /posts/\<post_pk>/delete/ url을 사용하여 직접 접속한다면 어떨까요?
![스크린샷](/statics/15/15_01.png)
![스크린샷](/statics/15/15_02.png)

request.user != post.author 구문에 의해 비로그인 사용자 임에도 불구하고 403에러가 응답됩니다.
request.user는 로그인 한 사용자는 User객체를 반환하지만 비로그인 사용자는 AnonymousUser 객체를 반환합니다.
하지만 조건문에는 post.author와 같지 않다면 모두 403에러를 응답하도록 작성되어 있습니다.

얼핏 보면 비로그인 사용자의 url을 통한 접근을 뷰에서 작성자 체크를 통해 에러가 났다고 응답 했고, 프론트 엔드에서 수정과 삭제버튼을 출력하지 않도록 처리했기 때문에 충분하다고 생각할 수 있습니다.

#### 그럼에도 @login_required 혹은 뷰 함수 작성을 통해 비로그인 사용자에 대한 처리가 필요한 이유
##### 1. 보안 안전장치
프론트엔드에서 버튼을 숨겼더라도 직접 url입력을 통해, 혹은 프로그램을 통해 HTTP 요청을 보내면 서버는 응답을 해주어야 합니다.
우리 프로젝트의 뷰가 응답하는 것은 403 Forbidden에러, 접근은 가능하나 허가되지 않은 리소스에 접근 할 때 보내주는 응답입니다.

그러나 비로그인 사용자(인증이 되지 않은 사용자)에게는 '접근(요청) 자체가 허용되지 않는다'라는 메세지를 명확하게 전송해 주는 것이 좋습니다.

##### 2. 세션과 UX
로그인 한 사용자가 자신의 게시물을 수정하고 있습니다. 그런데 도중에 세션이 만료되어 로그아웃 처리되었는데 이미 수정 \<form> HTML은 사용자에게 렌더되었고 사용자는 자신이 로그아웃 처리된 것을 인지하지 못한 채 게시물을 수정 중인 상태입니다. 
그 상황에서 게시물 수정 버튼을 누른다면 사용자는 403에러를 응답받게 됩니다.
이럴때는 에러 페이지를 응답하는 것이 아닌 로그인 페이지로 사용자를 이동시켜 로그아웃 처리 되었음을 사용자에게 알리고 기존의 작성했던 내용을 다시 \<form>의 value값으로 넣어 주는 것이 올바른 UX라고 할 수 있습니다.

##### 3. 코드내에 명시
뷰 함수를 작성할 때 @login_required 데코레이터는 '이 뷰는 로그인 한 사용자만 사용할 수 있다'라는 것을 명시해 줍니다. 
간단하게 코드 한 줄만으로 로그인 요구 기능 추가, 유지보수 시 이점을 가져다 줍니다.

#### 이제 데코레이터를 뷰 함수에 추가 해 봅시다.
```python
from django.contrib.auth.decorators import login_required
from django.core.paginator import Paginator
from django.http import HttpResponseForbidden
from django.shortcuts import render, get_object_or_404, redirect
from django.views.decorators.http import require_POST

from forum.forms import PostForm
from forum.models import *


def post_list(request):
    ...

# 당연하게도 누구나 볼 수 있는 post_list와 detail에는 추가하지 않습니다.
def post_detail(request, pk):
    post = get_object_or_404(Post, pk=pk)
    context = {'post': post}

    return render(request, 'forum/post_detail.html', context)


@login_required
def post_create(request):
    ...


@login_required
def post_update(request, pk):
    ...


@require_POST
@login_required
def post_delete(request, pk):
    ...
```

#### 결과 확인
##### 로그인을 하지 않고 직접 url을 입력하여 수정과 삭제에 접근해 봅시다.
![스크린샷](/statics/15/15_05.png)
![스크린샷](/statics/15/15_06.png)

##### 탭 2개를 활용하여 로그아웃 된 상황을 재현해 봅시다.
![스크린샷](/statics/15/15_07.png)
로그아웃 된 상태에서 수정과 삭제 버튼 클릭

![스크린샷](/statics/15/15_08.png)
이미 수정 페이지에 접근 한 상태에서 로그아웃 되었을 때 수정버튼 클릭
두 상황 모두 정상적으로 로그인 페이지로 이동하는 것을 확인하실 수 있습니다.

#### 나타나는 문제점
1. 수정 페이지에서 제출을 클릭하고 로그인을 하면 기존에 수정했던 내용이 사라져 있는 것을 확인하실 수 있습니다.
물론 뒤로가기를 두번 클릭하면 수정했던 내용과 함께 렌더되었던 페이지가 보이고 제출버튼을 클릭해도 수정이 정상적으로 이루어집니다만 당연히 좋은 방법이라고는 할 수 없습니다.
2. 수정 버튼을 클릭하고 로그인을 하면 수정 페이지로 이동하게 됩니다. 그러나 삭제 요청의 경우 POST 방식으로 처리해야 하는데 next 쿼리 파라미터 때문에 GET 방식으로 요청하게 되어 405에러를 반환합니다. 

다음 차시에서 이를 해결할 수 있는 방법들에 대해 알아보겠습니다.