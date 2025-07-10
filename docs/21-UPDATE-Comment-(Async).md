## 21. 비동기 요청으로 댓글 수정
이제 댓글 생성을 비동기로 요청해 봅시다. 비동기 요청인 만큼 수정버튼 클릭 시 댓글 수정 \<form>을 해당 댓글 아래에 나타나게 해봅시다.

### 서버 측 작업

#### models\.py
```python
class Comment(models.Model):
    author = models.ForeignKey(ForumUser, on_delete=models.CASCADE)
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    content = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)
    # null=True => null값 허용 / blank=True => is_valid() 입력 검증 시 값이 없어도 통과
    updated_date = models.DateTimeField(null=True, blank=True)
```
속성이 추가되었으니 마이그레이션을 진행합니다.

#### urls\.py
```python
from django.urls import path
from forum.views import base_views, post_views, comment_views

app_name = 'forum'

urlpatterns = [
    ...
    path('<int:post_pk>/comments/new/',                     comment_views.comment_create, name='comment_create'),
    path('<int:post_pk>/comments/',                         comment_views.comment_list,   name='comment_list'),
    path('<int:post_pk>/comments/<int:comment_pk>/update/', comment_views.comment_update, name='comment_update'),
]
```

#### comment_views.py
```python
def get_comments(post_pk):
    # 수정일을 댓글 리스트에 포함합니다.
    comments = Comment.objects.filter(post=post_pk).values(
        'pk', 'author__pk', 'author__username', 'author__nickname', 'content', 'created_date', 'updated_date')

    custom_field = []
    for comment in comments:
        custom_field.append({
            'pk': comment['pk'],
            'author_pk': comment['author__pk'],
            'username': comment['author__username'],
            'nickname': comment['author__nickname'],
            'content': comment['content'],
            'created_date': comment['created_date'],
            # 수정일을 댓글 리스트에 포함합니다.
            'updated_date': comment['updated_date'],
        })
    return custom_field

def comment_update(request, post_pk, comment_pk):
    comment = get_object_or_404(Comment, pk=comment_pk)

    if request.user != comment.author:
        return HttpResponseForbidden("다른 사용자의 댓글은 수정하실 수 없습니다.")

    if request.method == 'POST':
        form = CommentForm(request.POST, instance=comment)
        if form.is_valid():
            comment = form.save(commit=False)
            comment.updated_date = timezone.now()
            comment.save()
            context = {
                'status': 'success',
                'content': comment.content,
                'updated_date': comment.updated_date,
                # 'comments': get_comments(comment.post.pk)
                # 수정과 삭제의 경우 최신 리스트를 전송하여 새로 렌더링, JS를 통한 해당 엘리먼트만 수정/삭제 작업
                # 이 두 가지의 방법을 고민 할 수 있습니다. 

                # 우리 프로젝트는 다른 사용자의 신규 댓글 발생 시 댓글 리스트를 새로고침 해주는 Short Polling을
                # 사용 할 예정이기 때문에 댓글 리스트를 전송하지 않고 해당 html엘리먼트를 수정하는 작업만 진행합니다.

                # 다만 이것은 웹 어플리케이션 서비스 설계에 대한 내용으로 어떠한 방법을 사용하더라도 틀린 것이 아니며 
                # 각각의 운영관리자들의 설계 방향에 따라 달라질 수 있는 판단의 영역입니다.
            }
            return JsonResponse(context)
        context = {
            'status': 'error',
            'errors': form.errors,
        }
        return JsonResponse(context)
    return HttpResponseNotAllowed(['POST'])
```

---

### 클라이언트 측 작업

#### post_detail.html
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

    </div>
    <template id="comment-template">
        <div class="card mb-3">
            <div class="card-header"></div>
            <div class="card-body">
                <div class="card-text" style="white-space: pre-wrap"></div>
            </div>
            <div class="card-footer" style="font-size: small">
                <!--한 라인에 작성/수정 날짜와 수정/삭제 버튼을 배치하기 위하여 bootstrap의 row col을 사용합니다.-->
                <div class="row">
                    <div class="col">
                        <div class="comment-created-date"></div>
                        <!--업데이트 날짜-->
                        <div class="comment-updated-date"></div>
                    </div>
                    <!--우측 정렬(text-end), 수정 삭제 링크(버튼)의 제어를 위해 부모 div에 클래스명을 추가해 줍니다.-->
                    <div class="col text-end upd-del-link">
                        <a class="btn btn-outline-success btn-sm update-link">수정</a>
                    </div>
                </div>
            </div>
        </div>
    </template>

    <form id="comment-form" class="my-4" method="POST" data-url="{% url 'forum:comment_create' post.pk %}">
        ...
    </form>
</div>

{% load static %}
<script src="{% static 'js/post_detail.js' %}?{% now 'U' %}"></script>
<script type="text/javascript">
    const loginUrl = "{% url 'member:login' %}";
    // 자바스크립트에서 로그인 한 사용자에 따라 UX를 설정해 주기 위해 userPk를 설정해 줍니다.
    const userPk = "{{ request.user.pk }}";
    getComments({{ post.pk }})
</script>
{% endblock body %}
```

#### post_detail.js
```javascript
function bindDeleteListeners() {
    const deleteLinks = document.querySelectorAll('.delete-link');
    deleteLinks.forEach(function(element) {
        element.addEventListener('submit', function(event) {
            event.preventDefault();
            if(confirm("정말 삭제하시겠습니까?")) {
                element.submit();
            }
        })
    })
}

// 수정, 삭제 버튼을 클릭 시 비동기 요청을 보내게 하는 이벤트 리스너를 함수화 하였습니다.
// 아래 renderComments()에서 설명합니다.
function bindUpdateListeners() {
    const updateLinks = document.querySelectorAll('.update-link');
    updateLinks.forEach(function (element) {
        element.addEventListener('click', async function () {
            // 필요한 DOM 엘리먼트들을 미리 선언 해 줍니다.
            // 수정과 삭제 시 사용하기 위해 아래 renderComments()에서 부모 엘리먼트(div col)에
            // 댓글의 pk값을 데이터셋 변수에 할당했습니다.
            const commentPk = this.parentElement.dataset.commentPk;
            // pk 값을 통해 수정 대상 댓글의 <div card>를 찾습니다.
            // 이 또한 renderComments()에서 card(bootstrap)클래스에 id값으로 할당했습니다.
            const cardDiv = document.querySelector(`#comment-${commentPk}`)
            const cardTextDiv = cardDiv.querySelector('.card-text');
            const updatedDateDiv = cardDiv.querySelector('.comment-updated-date');

            // 수정 작업 취소(수정 html <form> 삭제) 버튼을 생성합니다.
            const cancelLink = document.createElement("a");
            cancelLink.className = "btn btn-outline-danger btn-sm";
            cancelLink.textContent = "취소";
            // this = element = 삭제버튼 의 바로 위에(옆에) 취소 버튼을 삽입합니다.
            this.parentElement.insertBefore(cancelLink, this);
            // 수정 버튼을 중복클릭해 여러 <form>이 생기지 않도록 display none 처리 합니다.
            this.style.display = 'none';

            // JS에서 직접 <form>을 생성하지 않고 댓글을 생성하는 <form>을 클론노드로 복사해 사용합니다.
            const updateForm = document.querySelector('#comment-form').cloneNode(true);
            updateForm.classList.remove('my-4');
            updateForm.querySelector('label').textContent = '';
            // 기존 댓글의 내용을 value에 작성해 줍니다.
            updateForm.querySelector('textarea').value = cardTextDiv.textContent;
            // <textarea>의 실제 값은 .value로 지정해야 합니다. (innerText, textContent (X))
                // value: 실제 내부 스트링
                // textContent, innerText: 태그 사이의 값을 변경 (<textarea>"here"</textarea>)
                    // 태그 사이에 스트링을 직접 입력하는 것은 초기 값을 설정하는 것입니다.
                    // 초기의 DOM에 <textarea>가 생성되고 사용자의 입력이나 submit이 일어났다면
                    // 값이 적용되지 않습니다.

                    // 우리 프로젝트는 댓글 생성 <form>을 생성하지 않고 클론노드하여 사용합니다.
                    // 아직 사용자가 <textarea>에 값을 입력을 하지 않았고 제출되지 않았다면
                    // 클론 노드또한 초기 값으로 인식되어 값이 입력되게 됩니다.
                    // 하지만 사용자가 댓글 생성 <form>의 <textarea>에 값을 입력했다면:
                        // 수정 <form> 출력 시 사용자가 입력했던 값이 적용되어 있고
                    // 사용자가 댓글을 생성(제출)했다면 (비동기):
                        // 초기 상태가 아니기 때문에 수정 <form>에 값이 적용되지 않습니다.

            // url String의 맨 앞에 '/'를 입력하지 않으면 현재 url(/posts/***/)에 append 됩니다.
            // (/posts/***/comments/***/update/)
            updateForm.dataset.url = `comments/${commentPk}/update/`;
            cardDiv.querySelector('.card-footer').append(updateForm);

            updateForm.addEventListener('submit', async function (event) {
                // form태그 이기때문에 submit의 클릭 이벤트를 실행하지 않도록 합니다.
                event.preventDefault();

                const body = new FormData();
                body.append('content', this.querySelector('textarea').value);
                body.append('csrfmiddlewaretoken',
                    document.querySelector('input[name="csrfmiddlewaretoken"]').value)
                // 클론노드로 복사했기 때문에 document내에서 찾아도 되고 updateForm에서 찾아도 됩니다.

                const context = await apiFetch(this.dataset.url, 'POST', body);
                if (context.status === 'success') {
                    cardTextDiv.textContent = context.content;
                    updatedDateDiv.textContent = `수정일: ${formatISODate(context['updated_date'])}`
                    // display none 처리 했던 수정 버튼을 다시 사용자에게 보여줍니다.
                    element.style.display = 'inline-block';
                    // 취소 버튼과 수정 <form>을 DOM 내에서 제거합니다.
                    cancelLink.parentElement.removeChild(cancelLink);
                    updateForm.parentElement.removeChild(updateForm);
                } else if (context.status === 'error') {
                    alert(context.errors.content);
                }
            })

            cancelLink.addEventListener('click', async function () {
                // 취소 버튼을 클릭 시에도 수정이 성공했을 때와 같은 작업을 진행합니다.
                element.style.display = 'inline-block';
                cancelLink.parentElement.removeChild(cancelLink);
                updateForm.parentElement.removeChild(updateForm);
            })
        })
    })
}

const redirectLogin = document.querySelectorAll('.redirect-login');
redirectLogin.forEach(function (element) {
    ...
})

async function apiFetch(url, method = 'get', body = null) {
    ...
}

async function getComments(post_pk) {
    ...
}


function formatISODate(date) {
    const dateObj = new Date(date);
    // 만약 수정일 데이터가 없다면
    if (date === null) {
        return '수정 사항 없음';
    }
    return dateObj.toLocaleString('ko-KR', {
        year: 'numeric', month: '2-digit', day: '2-digit',
        hour: '2-digit', minute: '2-digit',
    });
}


function renderComments(comments) {
    const commentsDiv = document.querySelector('.comments');
    commentsDiv.innerHTML = '';
    comments.forEach(comment => {
        const cardDiv = document.querySelector('#comment-template').content.cloneNode(true);
        // 수정과 삭제 시 댓글을 구별하기 위해 카드(bootstrap)에 id속성 값을 할당합니다.
        // firstChild()메서드는 모든 자식 노드(공백, 줄바꿈, 요소, 주석, 텍스트 ...)를 반환합니다.
        // 그렇기 때문에 엘리먼트만 지정해 줍니다. (<div card>)
        cardDiv.firstElementChild.id = `comment-${comment['pk']}`;

        cardDiv.querySelector('.card-header').append(`${comment['nickname']} (${comment['username']})`);
        cardDiv.querySelector('.card-body .card-text').append(comment['content']);
        const formattedCDate = formatISODate(comment['created_date']);
        cardDiv.querySelector('.card-footer .comment-created-date').append(`작성일: ${formattedCDate}`);
        // 수정일도 날짜 형식을 변환합니다. 수정 이력이 없다면 format..()함수에서 '수정 사항 없음'을 반환합니다.
        const formattedMDate = formatISODate(comment['updated_date']);
        cardDiv.querySelector('.card-footer .comment-updated-date').append(`수정일: ${formattedMDate}`);

        // 수정/삭제 버튼의 부모 <div>
        const updDelLink = cardDiv.querySelector('.card-footer .upd-del-link');
        // 문자열 끼리의 비교를 위해 toString()
        if (comment['author_pk'].toString() !== userPk) {
            // 로그인 한 사용자가 댓글의 작성자가 아니라면 수정버튼을 출력하지 않습니다.
            updDelLink.innerHTML = '';
            // updDelLink.style.display = 'none';
        } else {
            // 댓글의 작성자라면 부모 div의 데이터셋에 comment_pk값을 저장해 줍니다.
            updDelLink.dataset.commentPk = comment['pk'];
        }
        commentsDiv.appendChild(cardDiv);
    })
    // 비동기 요청을 통해 렌더 되는 댓글들의 수정과 삭제 버튼은 초기 페이지 접속 시
    // 비동기 요청 함수로 댓글이 렌더되기 전에 쿼리셀렉터가 수정/삭제 버튼에 이벤트 리스너 할당을 해버립니다.
    // 또한 댓글 생성 후 새로운 댓글 엘리먼트들이 기존의 댓글들을 대체했을 때도 이벤트리스너 할당이 되지 않습니다.
    // 그래서 함수화를 한 뒤 댓글을 렌더할 때마다 이벤트 리스너를 할당해 줍니다.
    bindDeleteListeners();
    bindUpdateListeners();
}


document.querySelector('#comment-form')
    .addEventListener('submit', async function (event) {
        ...
    })

```

#### 결과 확인
![스크린샷](/statics/21/21_01.png)
![스크린샷](/statics/21/21_02.png)