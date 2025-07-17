## 24. 댓글 페이징

### 서버 측 작업
#### comment_views.py
```python
def get_comments(post_pk, page):
    comments = Comment.objects.filter(post=post_pk).values(
        'pk', 'author__pk', 'author__username', 'author__nickname', 'content', 'created_date', 'updated_date')

    # 불필요한 반복문 연산을 막기 위해 get_comments에서 페이징 합니다.
    paginator = Paginator(comments, 10)
    # 요청한 페이지가 없다면:
    if not page:
        # 제일 마지막 페이지를 반환합니다.
        page = paginator.num_pages
    page_obj = paginator.get_page(page)

    custom_field = []
    for comment in page_obj:
        custom_field.append({
            'pk': comment['pk'],
            'author_pk': comment['author__pk'],
            'username': comment['author__username'],
            'nickname': comment['author__nickname'],
            'content': comment['content'],
            'created_date': comment['created_date'],
            'updated_date': comment['updated_date'],
        })

    context = {
        'comments': custom_field,
        # 클라이언트 측 페이징을 위해 페이지 숫자를 담아 전송해 줍니다.
        'last_page': paginator.num_pages,
    }
    return context


def comment_list(request, post_pk):
    comments = get_comments(post_pk, request.GET.get('page'))

    return JsonResponse(comments, safe=False)


@login_required
def comment_create(request, post_pk):
    if request.method == 'POST':
        post = get_object_or_404(Post, pk=post_pk)
        form = CommentForm(request.POST)
        if form.is_valid():
            comment = form.save(commit=False)
            comment.author = request.user
            comment.post = post
            comment.save()
            result_get_comments = get_comments(post_pk, None)
            context = {
                'status': 'success',
                'comments': result_get_comments['comments'],
                # 게시글을 작성했을 때도 last_page를 전송합니다.
                'last_page': result_get_comments['last_page']
                # 프론트엔드의 코드를 수정하지 않고, 형식을 맞추기 위해 comments와 last_page를
                # 각각의 key:value에 전송합니다.
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
댓글의 페이징을 구현하려고 합니다. 그런데 우리는 이미 게시물 리스트에서 페이징을 한번 만들어 보았는데요.
그 함수들을 재사용합니다.

#### /static/js/paging.js
먼저 페이징을 담당할 js모듈을 생성해 봅시다. 기존의 page-link a태그의 이벤트 리스너를 뺀 나머지 코드입니다.
```javascript
function makePageItem(innerText, page, isCurrentPage) {
    const pageItemLi = document.createElement('li');
    pageItemLi.classList.add('page-item');

    let pageLinkA;
    if (isCurrentPage) {
        pageLinkA = pageItemLi.appendChild(document.createElement('span'));
        pageLinkA.classList.add('page-link', 'active');
    }
    else {
        pageLinkA = pageItemLi.appendChild(document.createElement('a'));
        pageLinkA.classList.add('page-link');
        pageLinkA.href = "javascript:void(0)";
        pageLinkA.dataset.page = page;
    }
    pageLinkA.innerText = innerText;

    return pageItemLi;
}


// 위의 makePageItem함수와 다르게 이 함수는 post_list.js와 post_detail.js에서 사용할 것입니다.
// 그를 위해 export 키워드로 함수를 다른 js에서 사용할 수 있도록 합니다.
// 기존의 동기 코드를 함수화 하고 export 처리만 했습니다.
export function makePagination(currentPage, lastPage) {
    const paginationUl = document.querySelector('ul.pagination');
    paginationUl.innerHTML = '';

    const pageGroupSize = 10;
    const startOfGroup = Math.floor((currentPage - 1) / pageGroupSize) * pageGroupSize + 1;

    for(let i = startOfGroup; i <= startOfGroup + pageGroupSize - 1 && i <= lastPage; i++) {
        // post_list.html, post_detail.js 모두 currentPage를 문자열로 사용합니다.
            // const currentPage = "{{ posts.number }}";
            // requestPage = context['last_page'];
        // string으로 들어오는 currentPage를 비교식에 사용하기 위해 Number로 타입변환하여 사용합니다.
        const isCurrentPage = i === Number(currentPage);

        const returnedPageItemLi = makePageItem(i.toString(), i.toString(), isCurrentPage)
        paginationUl.appendChild(returnedPageItemLi);
    }

    if (startOfGroup > 1) {
        const returnedPageItemLi = makePageItem("<<", (startOfGroup - 1).toString(), false);
        paginationUl.prepend(returnedPageItemLi);
    }

    if (startOfGroup + pageGroupSize <= lastPage) {
        const returnedPageItemLi = makePageItem(">>", (startOfGroup + pageGroupSize).toString(), false);
        paginationUl.append(returnedPageItemLi);
    }
}
```

#### post_list.js
```javascript
//paging.js에서 export 했던 makePagination 함수를 불러와 줍니다.
import {makePagination} from '/static/js/paging.js'

// 이전의 동기 코드처럼 makePagination함수를 실행하고 (makePageItem은 직접 참조가 아니기에 자동으로 처리합니다.)
makePagination(currentPage, lastPage);

// post_detail의 댓글 페이징은 location.href를 이용한 SSR요청이 아니기 때문에 따로따로 작성해 줍니다.
const pageLinks = document.querySelectorAll('a.page-link');
pageLinks.forEach((pageLink) => {
    pageLink.addEventListener('click', (event) => {
        event.preventDefault();
        location.href = `/posts/?page=${pageLink.dataset.page}`;
    })
})
```

#### post_list.html
```html
{% extends 'base.html' %}
{% block title %}
Forum
{% endblock title %}

{% block body %}
<div class="container">
    ...
</div>
<script type="text/javascript">
    const currentPage = "{{ posts.number }}";
    const lastPage = "{{ posts.paginator.num_pages }}";
</script>
{% load static %}
<!-- 기존의 코드와 다르게 type을 module로 설정해 주었습니다. -->
<!-- 모듈함수가 있는 js를 불러오려면 모듈로 설정해야 하지만 그 모듈함수를 import하는 js또한 모듈로 불러와야 합니다. -->
    <!-- 모듈 타입으로 불러와야 import/export를 사용할 수 있습니다. -->
<!-- post_list.js에서 import한 paging.js는 불러오지 않아도 됩니다. -->
    <!-- 다만 export/import한 함수만 post_list.js에서 사용 가능힙니다. -->
<script src="{% static 'js/forum/post_list.js' %}?{% now 'U' %}" type="module"></script>
{% endblock body %}
```
#### 결과 확인
댓글의 페이징을 구현하기 전에 모듈화 한 게시물의 페이징이 정상 작동하는지 확인해봅니다.
![스크린샷](/statics/24/24_01.png)

---

### 이제 paging.js를 재사용하여 댓글의 페이징을 구현해 봅시다.
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
    <!-- 페이징 엘리먼트 구역 -->
    <nav class="col">
        <ul class="pagination justify-content-center">

        </ul>
    </nav>

    <div class="comments">

    </div>
    ...
</div>

<script type="text/javascript">
    const loginUrl = "{% url 'member:login' %}";
    const loggedInUserPk = "{{ request.user.pk }}";
    const postPk = "{{ post.pk }}"
</script>
{% load static %}
<!-- post_list.html과 같이 모듈로 js 로드합니다. -->
<script src="{% static 'js/forum/post_detail.js' %}?{% now 'U' %}" type="module"></script>
{% endblock body %}
```

#### post_detail.js
```javascript
// export 한 함수를 사용하기 위해 import
import {makePagination} from '/static/js/paging.js';

...

async function getComments(requestPage) {
    // url 문자열을 변수로 생성합니다.
    let url = `/posts/${postPk}/comments/`
    // 인자로 받아온 requestPage가 있다면 (특정 페이지 요청이 들어왔다면)
    if (requestPage) {
        // fetch는 GET요청에 대해 ajax처럼 body(data)에 담아 제출할 수 없습니다. 
        // 대신 url에 직접 삽입 해야 하기 때문에 URLSearchParams 객체를 생성해 쿼리 파라미터를 입력 해 줍니다.
        const queryParams = new URLSearchParams({ page: requestPage });
        url = url.concat(`?${queryParams}`);
        // 위의 코드가 실행되면 '/posts/${postPk}/comments/?page=***'의 문자열이 반환됩니다.
    }
    const context = await apiFetch(url);

    // 만약 요청 페이지가 없거나 마지막 페이지보다 큰 페이지를 요청한다면
    if (requestPage === undefined || requestPage > context['last_page']) {
        // 마지막(현재) 페이지의 버튼을 span, highlight처리 하기 위해 requestPage를 마지막 페이지로 설정하고
        requestPage = context['last_page'];
    }
    // renderComments에 요청페이지를 인자로 전달 해 줍니다.
    renderComments(context['comments'], requestPage, context['last_page']);
}

...

// 페이징 DOM 엘리먼트 렌더를 위해 요청페이지(현재페이지)와 마지막페이지를 인자로 받습니다.
function renderComments(comments, requestPage, lastPage) {
    ...
    bindDeleteListeners();
    bindUpdateListeners();

    // export 처리했던 makePagination 함수를 불러옵니다.
    makePagination(requestPage, lastPage);

    // page-link 클래스의 a태그들에 클릭 이벤트를 추가합니다.
    document.querySelectorAll('a.page-link').forEach((link) => {
        link.addEventListener('click', async function () {
            // 데이터셋에 저장된 페이지를 요청합니다.
            await getComments(link.dataset.page);
        })
    })
}


document.querySelector('#comment-form')
    .addEventListener('submit', async function (event) {
        ...
        if (context['status'] === 'success') {
            // 서버에서 전달받은 context의 comments와 last_page를 이용하여 렌더함수를 불러옵니다.
            // 신규 댓글의 경우 본인 댓글을 보여주기 위해 last_page를 요청합니다.
            renderComments(context['comments'], context['last_page'], context['last_page']);
            this.querySelector('textarea').value = '';
        } else if (context['status'] === 'error') {
            ...
        }
    })


// 기존에는 post_detail.html내에서 getComments()를 통해 초기 로드를 진행했습니다.
// 그러나 type=module로 js를 불러오면 html내에서 함수를 로드할 수 없습니다.
// a.js를 모듈로 불러오면 그 안에 정의된 함수들은 전역(window)에 등록되지 않고 오직 a.js 내부에서만 사용 가능합니다.
// 모듈에서 정의된 변수, 함수, 클래스는 export한 것들만 import해서 사용할 수 있습니다.
// 정말 필요한 경우 a.js내에서 window.aaa = aaa;처럼 강제로 전역화 할 수 있습니다.
// 위와 같은 이유로 우리는 html내에서는 모듈만 불러오고 모듈 내에서 초기 실행 함수를 마지막줄에서 불러옵니다.

// 1. DOM요소에 의존하지 않을 때
// await getComments();
// 2. 혹시라도 DOM요소가 렌더되지 않았을 때 동작하면 문제가 될 때
window.addEventListener('DOMContentLoaded', () => getComments());
// 2의 이벤트리스너는 실행 전의 함수 자체를 넘겨줘야 합니다. => 함수 참조 ('DOM..', getComments)
// 만약 getComments()를 넣어주면 함수가 아닌 실행 결과(return)를 넘겨주게 됩니다. 
// (이벤트 리스너는 async 함수도 자연스럽게 실행해 줍니다.)

// 화살표 함수를 사용한 이유
// 함수 참조 방법을 사용하면 이벤트가 발생할 때 함수에 자동으로 이벤트 객체(event)를 첫번째 인자로 넣어줍니다.
// 이벤트 리스너에서 인자가 있는 함수를 사용하려면 화살표 함수를 사용해야 합니다.
// 그렇지 않으면 getComments의 requestPage인자에 이벤트 객체가 들어가 정상작동 하지 않습니다.
```

#### 결과 확인
![스크린샷](/statics/24/24_02.png)
![스크린샷](/statics/24/24_03.png)