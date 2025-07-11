## 22. 비동기 요청으로 댓글 삭제

### 서버 측 작업

#### urls\.py
```python
urlpatterns = [
    ...
    path('<int:post_pk>/comments/new/',                     comment_views.comment_create, name='comment_create'),
    path('<int:post_pk>/comments/',                         comment_views.comment_list,   name='comment_list'),
    path('<int:post_pk>/comments/<int:comment_pk>/update/', comment_views.comment_update, name='comment_update'),
    path('<int:post_pk>/comments/<int:comment_pk>/delete/', comment_views.comment_delete, name='comment_delete'),
]
```

#### comment_views.py
```python
@login_required
def comment_delete(request, post_pk, comment_pk):
    comment = get_object_or_404(Comment, pk=comment_pk)

    if request.user != comment.author:
        return HttpResponseForbidden("다른 사용자의 댓글을 삭제하실 수 없습니다.")

    # 삭제 작업 또한 POST, DELETE HTTP 메서드로만 처리하는 것이 안전합니다. 
    # RESTful한 프로젝트가 아님으로 POST 요청을 사용합니다.
    # (@require_POST 데코레이터으로 대체 가능합니다.)
    if request.method == 'POST':
        comment.delete()
        context = {
            'status': 'success',
        }
        return JsonResponse(context)
    return HttpResponseNotAllowed(['POST'])
```

---

### 클라이언트 측 작업

#### post_detail.html
```html
<div class="container">
    ...
    <template id="comment-template">
        <div class="card mb-3">
            ...
            <div class="card-footer" style="font-size: small">
                <div class="row">
                    ...
                    <div class="col text-end upd-del-link">
                        <a class="btn btn-outline-success btn-sm update-link">수정</a>
                        <!-- 삭제 링크를 댓글에 생성합니다. -->
                        <a class="btn btn-outline-danger btn-sm delete-link">삭제</a>
                    </div>
                </div>
            </div>
        </div>
    </template>
    ...
</div>
```

#### post_detail.js
```javascript
// csrf 토큰 <input>값을 참조하는 함수가 많아져 전역 변수(변수 캐싱) 처리 하였습니다.
const csrfTokenValue = document.querySelector('input[name="csrfmiddlewaretoken"]').value;


function bindDeleteListeners() {
    const deleteLinks = document.querySelectorAll('.delete-link');
    deleteLinks.forEach(function (element) {
        // await apiFetch를 사용하기 위한 비동기화
        // 게시물 삭제 때와 다르게 form태그가 아닌 a태그를 이용한 클릭 이벤트를 사용합니다.
        // 게시물의 삭제도 fetch를 이용하여 form태그 없이 a태그의 데이터셋-링크로 요청을 보낼 예정입니다.
        element.addEventListener('click', async function () {
            if (confirm("정말 삭제하시겠습니까?")) {
                // POST 메서드로 삭제 요청을 보내야 하기 때문에 토큰 값을 body에 삽입해 줍니다.
                const body = new FormData();
                body.append('csrfmiddlewaretoken', csrfTokenValue)
                const context = await apiFetch(this.dataset.url, 'POST', body);

                if (context.status === 'success') {
                    // 서버에서 삭제된 댓글을 사용자의 DOM에서도 삭제합니다.
                    const commentPk = this.parentElement.dataset.commentPk;
                    const cardDiv = document.querySelector(`#comment-${commentPk}`);
                    cardDiv.parentElement.removeChild(cardDiv);
                }
            }
        })
    })
}


function bindUpdateListeners() {
                ...
                // 전역 변수 값으로 대체
                body.append('csrfmiddlewaretoken', csrfTokenValue)
                ...
}

...

function renderComments(comments) {
    ...
    comments.forEach(comment => {
        ...
        if (comment['author_pk'].toString() !== userPk) {
            ...
        } else {
            updDelLink.dataset.commentPk = comment['pk'];
            // 게시물 삭제와 동일하게 삭제 링크 엘리먼트의 데이터셋에 요청 주소를 할당합니다.
            updDelLink.querySelector('.delete-link').dataset.url = `comments/${comment['pk']}/delete/`
        }
        ...
    })
    bindDeleteListeners();
    bindUpdateListeners();
}


document.querySelector('#comment-form').addEventListener('submit', async function (event) {
        ...
        // 전역 변수 값으로 대체
        body.append('csrfmiddlewaretoken', csrfTokenValue)
        ...
    })
```

#### 결과 확인:
![스크린샷](/statics/22/22_01.png)
![스크린샷](/statics/22/22_02.png)

---

### 게시물 삭제 뷰의 redirect
비동기 요청을 통한 댓글의 삭제가 정상 작동하는 것을 확인했습니다. 
이제 게시물 삭제도 같은 삭제 함수를 사용하기 위해 \<form>태그가 아닌 dataset.url을 가지는 a태그로 바꿔 봅시다.
```html
<!--
<div class="col d-flex justify-content-end">
    <a href="{% url 'forum:post_update' post.pk %}"
        class="btn btn-outline-success mt-2">수정</a>

    <form action="{% url 'forum:post_delete' post.pk %}" method="POST" class="delete-link ms-1">
        {% csrf_token %}
        <input type="submit" class="btn btn-danger mt-2 delete-link" value="삭제">
    </form>
</div>
-->

<!-- a태그 두개를 사용하기 때문에 플렉스 클래스를 사용하지 않음으로 text-end를 이용해 우측정렬 합니다. -->
<div class="col text-end">
    {% if post.author == request.user %}
    <a href="{% url 'forum:post_update' post.pk %}"
        class="btn btn-outline-success mt-2">수정</a>
        
    <a data-url="{% url 'forum:post_delete' post.pk %}"
        class="btn btn-danger mt-2 delete-link">삭제</a>
    {% endif %}
</div>
```

이제 게시물을 삭제해 봅시다.
![스크린샷](/statics/22/22_03.png)
![스크린샷](/statics/22/22_04.png)
개발자 도구의 콘솔창을 보면 에러가 뜨며 html 코드가 JSON 타입에 유효하지 않다고 전달해 줍니다.
오류가 나는 코드에 접속해 봅시다.

![스크린샷](/statics/22/22_05.png)
![스크린샷](/statics/22/22_06.png)
return redirect를 통해 서버에서 렌더링해 응답된 post_list html의 내용이 보입니다.

#### post_views.py
게시물 목록 페이지(/posts/)를 확인해 보면 게시물은 정상적으로 삭제된 것을 확인할 수 있습니다. 
다만 게시물이 삭제되고 나서 목록 페이지로의 리다이렉트가 되지 않고 있습니다. 우리의 게시물 뷰를 살펴봅시다.
```python
# request.method == 'POST' 조건문 대신 데코레이터을 활용했습니다.
@require_POST
@login_required
def post_delete(request, post_pk):
    post = get_object_or_404(Post, pk=post_pk)
    if request.user != post.author:
        return HttpResponseForbidden("다른 사용자의 게시물은 삭제하실 수 없습니다.")

    post.delete()
    return redirect('forum:post_list')
```
게시물 뷰의 삭제 함수는 post의 삭제 요청이 완료되면 사용자를 post_list로 리다이렉트 하도록 안내합니다.

Django의 return redirect(''), @login_required 등은 해당 메서드가 실행되면 302 리다이렉트 응답을 보냅니다.
일반적인 동기 요청이었다면 브라우저가 자동으로 응답을 처리해 리다이렉트 했겠지만 우리는 비동기 요청을 보냈습니다.
fetch는 redirect 응답을 처리하기 위해 response의 내용에 따른 분기코드를 작성해야 하는데 우리 프로젝트의 코드를 보면 비동기 요청의 반환 값을 JSON형식으로 변환 후 return 시켜주기만 했습니다.

그래서 우리는 fetch가 redirect 응답이 있을 때 처리할 수 있도록 코드를 작성해야 합니다.

#### post_detail.js
```javascript
async function apiFetch(url, method = 'get', body = null) {
    try {
        const options = {
            method: method,
        }
        if (body) options.body = body;

        const response = await fetch(url, options);
            

        // 먼저 리다이렉트를 처리 하기 전 response에 어떤 데이터가 있는지 확인해 봅시다.
        console.log(response);

        if (response.redirected) {
            console.log(response.redirected);
            console.log(response.url);
            // location.href = response.url;
        }


        if (!response.ok) throw new Error(response.statusText);

        return response.json();
    } catch (error) {
        console.error(`Fetch: Failed to fetch: ${error}`);
    }
}

```
##### 결과 확인
![스크린샷](/statics/22/22_07.png)
위의 콘솔창을 보면 response에 redirected와 이동할 url주소를 전달받은 것을 확인 할 수 있습니다.

##### status가 200인 이유 = html (post_list)가 렌더링 된 이유
서버가 302 응답을 보내면 fetch는 응답을 따라가 서버에서 렌더링 된 최종 응답을 받아옵니다. 
단지 비동기 요청이기 때문에 코드 내에 작성이 되어 있지 않다면 리다이렉트를 하지 않을 뿐 입니다.
- response.redirected : true (리다이렉트 응답이 있었다.)
response.url : 최종적으로 도착한 URL (리다이렉트 후 도착지 (서버가 렌더링 해준 페이지))
response.status : 최종 응답(리다이렉트)의 status (해당 페이지가 존재/정상 작동하기 때문에 200)

##### 그냥 리다이렉트 요청이 오면 fetch가 브라우저를 리다이렉트 시키면 안되나요?
1. 비동기 요청은 '데이터만' 다룹니다.
    - 비동기 요청은 데이터를 받아와 JS에서 처리하는 용도입니다.
    - 만약 리다이렉트를 시킨다면 비동기로 받아온 데이터를 활용하는 의미가 없어집니다.
2. 클라이언트의 화면 제어는 개발자가
    - fetch는 비동기로 서버에서 데이터를 전달 받을 뿐 클라이언트의 화면 제어는 JS의 몫입니다.
    - 서버에서 데이터를 받아왔는데 리다이렉트 응답 데이터가 있으면 개발자(JS)가 직접 이동시킵니다.
3. 다른 데이터가 포함되어 있다면?
    - 만약 자동 리다이렉트를 시켜주는데 여러개의 fetch요청에서 리다이렉트가 중복된다면
    - 같이 전달되는 데이터가 필요한 상황에서 자동으로 리다이렉트를 해버린다면

요청 후 "화면은 현 상태를 유지"하며 필요한 데이터에 따라 DOM 엘리먼트를 CRUD하는 비동기의 원칙을 지켜야 합니다.

위와 같은 이유로 fetch가 받아온 SSR 응답은 javascript 내에서만 사용이 가능합니다. 사용하고 싶다면 아래와 같은 형식으로 사용합니다.
```javascript
document.innerHTML = await response.text();
```
그러나 이렇게 사용하면 요청을 보낸 URL그대로이고 스크립트와 스타일 로드, 브라우저 히스토리 등에 문제가 생길 수 있기 때문에 location.href를 이용한 브라우저 단의 리다이렉트를 사용합니다.

이제 location.href = response.url의 주석 처리를 삭제해보면 게시물이 삭제되고 정상적으로 post_list에 리다이렉트 되는 것을 보실 수 있습니다.

#### API뷰의 JSON 응답 보내기
만약 여러 클라이언트 디바이스에 대응하기 위한 RESTful한 프로젝트라면 redirect가 아닌 JSON같은 텍스트의 형식으로 응답을 전달해야 합니다. 그럴 때는
```python
@require_POST
def post_delete(request, post_pk):
    post = get_object_or_404(Post, pk=post_pk)
    if request.user != post.author:
        return HttpResponseForbidden("다른 사용자의 게시물은 삭제하실 수 없습니다.")

    post.delete()
    contest = {
        ...
        'redirect': '/posts/',
    }
    return JsonResponse(context)
```
```javascript
if (context.redirect) {
    location.href = context.redirect;
}
```
위의 코드처럼 처리하면 됩니다.