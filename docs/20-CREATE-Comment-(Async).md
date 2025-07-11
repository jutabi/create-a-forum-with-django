## 20. 비동기 요청으로 댓글 생성
이제 댓글 생성을 비동기로 요청해 봅시다.

### 서버 측 작업

#### urls\.py
기존의 SSR url을 그대로 사용합니다.

#### comment_views.py
```python
from django.http import HttpResponseNotAllowed, JsonResponse
from django.shortcuts import render, get_object_or_404

from forum.forms import CommentForm
from forum.models import *


# 댓글 생성 후에 클라이언트에게 comments 쿼리셋을 넘겨 줄 예정인데 comment_list와 중복코드를 막기 위해
# 댓글 리스트를 불러오는 함수를 따로 작성하였습니다.
    # 댓글 생성 시 상태만 전송하고 성공했다면 클라이언트에서 comment list 요청을 보낼 수도 있지만
    # 그렇게 되면 서버에 결국 두번 요청하는 것이기 때문에 리소스를 낭비하게 되고 코드도 불필요한 증가가 생깁니다.
def get_comments(post_pk):
    comments = Comment.objects.filter(post=post_pk).values(
        'pk', 'author__pk', 'author__username', 'author__nickname', 'content', 'created_date')

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
    return custom_field


@login_required
# Post의 CUD때와 마찬가지로 @login_required 데코레이터를 사용해 줍니다.
def comment_create(request, post_pk):
    if request.method == 'POST':
        post = get_object_or_404(Post, pk=post_pk)
        form = CommentForm(request.POST)
        if form.is_valid():
            comment = form.save(commit=False)
            comment.author = request.user
            comment.post = post
            comment.save()
            # 기존의 코드와 다르게 redirect 하는 것이 아닌 status와 comments 쿼리셋을 전송합니다.
            # 클라이언트 측 코드에서 댓글 생성이 완료되면 다시 getComments()작업을 하는 것이 아닌
            # 생성이 정상 처리 되면 뷰에서 바로 쿼리셋을 넘기고 클라이언트는 renderComments()를 진행합니다.
            context = {
                'status': 'success',
                'comments': get_comments(post_pk),
            }
            return JsonResponse(context)
        # 만약 폼에 문제가 있다면 status와 에러 내용, 기존의 <form>내용을 사용자에게 다시 반환합니다.
        context = {
            'status': 'error',
            'errors': form.errors,
            # SSR에서는 오류 발생 시(글자제한을 넘었거나, 특정패턴 금지에 걸렸거나) 기존의 문자열을 사용자에게 
            # 다시 보내줘야 하지만 비동기 요청에서는 창이 새로고침 되거나 리다이렉트 되지 않기 때문에 입력 내용이
            # 살아있어 오류 메세지만 보내주고, <textarea>에 기존 입력 content를 재지정 해 줄 필요가 없습니다.
            # 'content': form.cleaned_data.get('content'),
        }
        return JsonResponse(context)
    return HttpResponseNotAllowed(['POST'])


def comment_list(request, post_pk):
    comments = get_comments(post_pk)

    return JsonResponse(list(comments), safe=False)

```

---

### 클라이언트 측 작업

#### post_detail.html
새로운 댓글 갱신 작업 시 (생성, 수정, 삭제) 기존 내용을 삭제하기 위해 \<template>태그를 밖으로 빼주었습니다.
```html
    <div class="comments">

    </div>
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

    <!--기존의 action 속성 값을 데이터셋-url로 변경합니다.-->
    <form id="comment-form" class="my-4" method="POST" data-url="{% url 'forum:comment_create' post.pk %}">
        {% csrf_token %}
        <!--form_errors 템플릿을 사용하지 않습니다.-->
        <!--{% include 'form_errors.html' %}-->
        <div class="my-2">
            ...
        </div>
        <div class="text-end">
            ...
        </div>
    </form>
```

#### post_detail.js
```javascript
// 여러 함수에서 같은 api를 사용하는 비동기 요청이 있을 때는 요청 로직을 함수로 만들어 재사용 합니다.
// method와 body(ajax의 data)인자를 기본값을 설정하여 입력값이 없더라도 기본 get요청으로 동작하게 합니다.
async function apiFetch(url, method = 'get', body = null) {
    try {
        const options = {
            method: method,
        }
        if (body) options.body = body; //options['body'] = body;
        // body인자의 값이 존재한다면 => 아래 객체 접근 항목 참조
        // options = {
        //     method: method,
        //     body: body,
        // }

        const response = await fetch(url, options);
        if (!response.ok) throw new Error(response.statusText);

        // apiFetch()를 호출할 때 await를 사용하기 때문에 await없이 리턴했습니다.
        return response.json();
    } catch (error) {
        console.error(`Fetch: Failed to fetch: ${error}`);
    }
}

async function getComments(post_pk) {
    // 프로미스가 아닌 JSON데이터를 받아오기 위해 await, 를 사용하기 위해 함수를 async로 선언
    const comments = await apiFetch(`/posts/${post_pk}/comments/`);
    renderComments(comments);
}

function formatISODate(date) {
    ...
}

function renderComments(comments) {
    const commentsDiv = document.querySelector('.comments');
    // 이미 렌더링 되어 있는 comments에서 다시 렌더링할 때를 위해서 innerHTML(하위 노드들)을 제거합니다.
    // <template>를 <div comments>의 하위에서 밖으로 빼내줍니다.
    commentsDiv.innerHTML = '';

    comments.forEach(comment => {
        ...
        // 커스텀 필드명을 적용했기 때문에 key의 이름을 변경했습니다.
        cardDiv.querySelector('.card-header').append(`${comment['nickname']} (${comment['username']})`);
        ...
    })
}

document.querySelector('#comment-form')
    // await를 사용하기 위해 이벤트 리스너 함수도 async로 선언합니다.
    .addEventListener('submit', async function (event) {
        // 기존 요소의 동작을 막아 주는 메서드 입니다.
        // a 태그 클릭 시 링크 이동, <form>의 submit 시 새로고침 하며 데이터 전송 등
        // 메서드를 작성 하지 않고 confirm('hello')를 입력해 봅시다. (confirm 후 페이지가 새로고침 됩니다.)
        event.preventDefault();

        // 아래 context.status === 'error'일때 생성한 경고창 div를 삭제합니다.
        // jQuery 사용 시 부모 노드에서 삭제하지 않고, 엘리먼트.remove()사용이 가능합니다.
        const commentFormAlert = document.querySelector('#comment-form-alert');
        if (commentFormAlert) {
            // 여기서 this는 document.querySelector('#comment-form') 입니다.
            this.removeChild(document.querySelector('#comment-form-alert'));
        }

        // JSON으로 content와 csrf토큰을 넘기는게 아닌 기존의 <form>태그의 작동과 동일한 FormData 객체로 전송합니다.
        // JSON으로 전송 시:
        // const body2 = {
        //     content: this.querySelector('textarea').value,
        //     csrfmiddlewaretoken: document.querySelector('input[name="csrfmiddlewaretoken"]').value,
        // }
        const body = new FormData();
        body.append('content', this.querySelector('textarea').value);
        body.append('csrfmiddlewaretoken', document.querySelector('input[name="csrfmiddlewaretoken"]').value)

        // await를 사용하지 않으면 Promise 객체를 반환합니다.
        // .then을 사용하지 않고 서버에서 전달받은 context를 바로 리턴받기 위해 await를 사용합니다.
        const context = await apiFetch(this.dataset.url, 'POST', body);
        if (context['status'] === 'success') {
            // 서버에서 comment 생성 성공과 쿼리셋(JSON)을 전달받았다면 렌더 함수를 진행합니다. (기존 내용 삭제)
            renderComments(context['comments']);
            // 비동기 처리이기 때문에 textarea에 작성했던 댓글의 내용이 지워지지 않고 살아있습니다. 지워줍시다.
            this.querySelector('textarea').value = '';
        }
        else if (context['status'] === 'error') {
            // alert(`${Object.keys(context.errors)}: ${context['errors']['content']}`)

            // this(comment create <form>)에 경고창을 띄워줍시다.
            // 기존의 form_errors.html 은 사용하지 않습니다.
            const div = document.createElement('div');
            // 또 한번의 요청(성공, 실패 둘 다)을 했을 때 기존의 오류 메세지를 제거하기 위해 id속성을 추가합니다.
            div.id = 'comment-form-alert';
            div.className = 'alert alert-danger';
            const errMsg = div.appendChild(document.createElement('strong'));
            // 장고 폼의 라벨은 템플릿 내에서만 적용됩니다.
            errMsg.textContent = "내용: ";
            div.append(context['errors']['content']);
            // <form>의 맨 위에 경고창을 추가해 줍니다.
            this.prepend(div);
        }
    })
```

##### 결과 확인
![스크린샷](/statics/20/20_01.png)
![스크린샷](/statics/20/20_02.png)

##### 자바스크립트의 객체 접근
```javascript
// 점 표기법
object.pk
object.name
// 키 이름이 유효한 자바스크립트 식별자(문자, _, $ 등으로 시작하고 공백이나 특수문자가 없음)일 때 사용 가능합니다.

// 대괄호 표기법
object['pk']
object['author-name']
const keyName = 'name';
object[keyname]
// 키 이름이 변수를 사용하여 동적으로 결정될 때, 공백 하이픈 숫자 등으로 시작하는 키 이름일 때도 사용 가능합니다.

// 키는 문자열이고 전달받은 곳에 따라 유효한 식별자로 시작하지 않을 수 있기 때문에 두 방식을 모두 지원합니다.
// 저는 JSON 객체의 키 값이라는 것을 강조하고 싶어서 대괄호 표기법을 사용하였습니다.
```

##### 자바스크립트 클릭 이벤트 리스너
```javascript
// jQuery 사용 시:
$('#comment-form').submit(function (event) {

})

// 한번만 지정 가능 (핸들러 개수: 1개), 두번 이상 지정시 덮어쓰기 됨
document.querySelector('#comment-form').onsubmit = function (event) {

}

// 여러 기능 등록 가능 (핸들러 개수: 여러 개)
document.querySelector('#comment-form').addEventListener('submit', function (event) {

})
```