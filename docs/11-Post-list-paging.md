## 11. Post list paging

[참고 문헌: 점프 투 장고 / 3-02 페이징](https://wikidocs.net/71240)

멤버 기능을 구현하기 전에 게시판 프로젝트의 제일 중요한 페이징을 진행합니다.
페이징 기능은 구현하려면 어려운점이 존재하지만 프레임워크를 쓰는 우리는 큰 어려움이 되지 않습니다.

### 테스트 데이터 생성
#### 장고 쉘에서 반복문을 이용하여 테스트 데이터를 생성, 삽입하자
```
(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py shell
8 objects imported automatically (use -v 2 for details).

Python 3.13.1 (tags/v3.13.1:0671451, Dec  3 2024, 19:06:28) [MSC v.1942 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>> from forum.models import Post
>>> for i in range(300): 
        // 들여쓰기 주의
...     post = Post(
...     title="Test title %03d" % i,
...     content="Test content %03d" % i)
...     post.save()
...
// 엔터 한번 더
>>> exit()
```

결과 확인:
![스크린샷](/statics/11/11_01.png)

---

### Django Paginator

#### views.py
page_list를 템플릿에게 전달해주는 forum/views.py 의 post_list()를 수정합시다.
```python
from django.core.paginator import Paginator
from django.shortcuts import render, get_object_or_404, redirect

from forum.forms import PostForm
from forum.models import *

def post_list(request):
    posts = Post.objects.order_by('-created_date')
    # 페이지는 GET방식 쿼리 파라미터(?page=12)로 전달하는 것이 권장되는 방식입니다.
    # 가끔 /page/12 와 같은 패스 파라미터를 사용하는 웹 어플리케이션을 볼 수 있는데 완전히 잘못된 것은 아니지만
    # REST 설계 원칙에는 맞지 않습니다. (다만 우리 프로젝트는 REST API를 기반한 프로젝트가 아닙니다.)

    # 패스 파라미터는 (/posts/12/)처럼 리소스의 고유 식별자를 지정할 때 사용합니다.
    # 우리 프로젝트에서도 (/posts/<int:post_pk>/)처럼 단 1개, 유일한 post를 가리킬 때 사용합니다.

    # 쿼리 파라미터는 (?search_keyword=django&page=12)처럼 필터링, 정렬, 페이징 등 옵션, 조건 지정에 사용합니다.

    # 리퀘스트 요청(주소)에서 사용자가 요청한 페이지 값을 불러옵니다.
    # 만일 페이지를 요청하지 않았을 경우(index 페이지) 1을 삽입합니다.
    request_page = request.GET.get('page', 1)

    # Django의 paginator 객체를 생성하는데, 위에서 생성일 역순으로 불러온 posts 객체를 기준으로
    # 페이지당 10개의 데이터를 할당하여 생성합니다.
    paginator = Paginator(posts, 10)
    # 생성한 paginator 객체에서 사용자가 요청한 페이지의 쿼리셋을 불러옵니다.
    page_obj = paginator.get_page(request_page)

    # 컨텍스트에 해당페이지의 내용들을 담아 사용자에게 렌더할 때 템플릿에 전달합니다.
    context = {'posts': page_obj}

    return render(request, 'forum/post_list.html', context)
```

#### post_list.html
이제 전달한 페이지 데이터를 템플릿에서 사용해 봅시다.

다만 우리는 javascript를 사용하여 부트스트랩 페이징 엘리먼트들을 생성, 삽입할 예정이기 떄문에 예제 코드를 확인하고 갑니다.
```html
{% extends 'base.html' %}
{% block title %}
Forum
{% endblock title %}

{% block body %}
<div class="container">
    <table class="table table-striped table-hover">
        ...
    </table>

    <div class="row">
        <div class="col">
            <a href="{% url 'forum:post_create' %}" class="btn btn-primary">게시물 작성</a>
        </div>
        <nav class="col">
            <ul class="pagination justify-content-center">
                <li class="page-item disabled">
                    <a class="page-link">Previous</a>
                </li>
                <li class="page-item"><a class="page-link" href="#">1</a></li>
                <li class="page-item active">
                    <a class="page-link" href="#" aria-current="page">2</a>
                </li>
                <li class="page-item"><a class="page-link" href="#">3</a></li>
                <li class="page-item">
                    <a class="page-link" href="#">Next</a>
                </li>
            </ul>
        </nav>
        <div class="col">
            
        </div>
    </div>
</div>
{% endblock body %}
```

##### 결과 확인
![스크린샷](/statics/11/11_02.png)
이미 views.py에서 render시 1페이지의 데이터만을 넘겨주기 때문에 paginator를 적용하기 전처럼 300개의 데이터가 한번에 표시되지는 않습니다.

우리가 유심히 봐야할 \<nav>태그를 확인해봅시다.
게시물 작성 버튼과 같은 라인에 페이징 버튼을 위치시키기 위해 row클래스 div태그 아래에 col클래스를 이용하여 하나의 row를 3등분 하여 엘리먼트들이 자리할 수 있도록 처리했습니다. (마지막, 3번째 col div태그는 검색기능이 추가될 예정입니다.)
![스크린샷](/statics/11/11_03.png)

다시 \<nav>로 시작하는 Bootstrap pagination 컴포넌트 코드를 확인해 봅시다.
```html
<!-- row 내에서 사용하기 위해 col 클래스명 삽입 -->
<nav class="col">
    <!-- 페이지네이션 컴포넌트 명시, 부모 엘리먼트에서 중앙정렬 되기 위하여 justify-content-center -->
    <ul class="pagination justify-content-center">
        <!-- 페이지 아이템이라고 명시하고, 클릭 불가능 표시(이전 페이지가 없다면)를 위해 disabled -->
        <li class="page-item disabled">
            <a class="page-link">Previous</a>
            <!-- 만약 클릭기능과 키보드 포커스를 해제하고 싶다면 자식 노드를 span 태그로 설정합니다. -->
            <!-- <span class="page-link">Previous</span> -->
        </li>
        <li>생략</li>
        <!-- 사용자에게 현재 페이지라는 것을 강조하고 싶을 때 active 클래스명 사용 -->
        <li class="page-item active">
            <!-- <a class="page-link active" 처럼 page-link에서 사용해도 상관 없습니다. -->
            <a class="page-link" href="#" aria-current="page">2</a>
        </li>
        <li>생략</li>
        <li>생략</li>
    </ul>
</nav>
```

위의 코드와 템플릿 태그를 이용하여 (if, for) 페이징을 구현하는 방법은 구현에 불편함이 있습니다. 그래서 우리는 javascript를 이용하여 구현합니다.

js의 document.createElement 메서드를 통해 html (태그) 엘리먼트를 생성하고 appendChild()를 통해 HTML DOM 내부에 생성한 엘리먼트들을 위치시킬 예정입니다.
다만 \<nav>, \<ul class=pagination>은 커스텀할 필요가 없고 HTML 내에 위치까지 알려줄 수 있는 태그입니다. 
우리가 원하는 작업은 그 내부의 page-item들만 입맛에 맞추어 생성하면 되기 때문에 \<li>들을 삭제하거나 참고하여 코드를 짜기 위해 주석처리 합시다.
```html
<div class="row">
    <div class="col">
        <a href="{% url 'forum:post_create' %}" class="btn btn-primary">게시물 작성</a>
    </div>
    <nav class="col">
        <ul class="pagination justify-content-center">
            <!--
            <li class="page-item disabled">
                <span class="page-link">Previous</span>
            </li>
            <li class="page-item"><a class="page-link" href="#">1</a></li>
            <li class="page-item active">
                <a class="page-link" href="#" aria-current="page">2</a>
            </li>
            <li class="page-item"><a class="page-link" href="#">3</a></li>
            <li class="page-item">
                <a class="page-link" href="#">Next</a>
            </li>-->
        </ul>
    </nav>
    <div class="col">

    </div>
</div>
```

---

### 자바스크립트로 페이징 엘리먼트 생성하기

#### post_list.html
```html
{% extends 'base.html' %}
{% block title %}
Forum
{% endblock title %}

{% block body %}
<div class="container">
    <table class="table table-striped table-hover">
        ...
    </table>

    <div class="row">
        <div class="col">
            <a href="{% url 'forum:post_create' %}" class="btn btn-primary">게시물 작성</a>
        </div>
        <nav class="col">
            <ul class="pagination justify-content-center">
                <!-- 이 위치에 js로 작성한 엘리먼트들이 자녀요소로 append된다. -->
            </ul>
        </nav>
        <div class="col">

        </div>
    </div>
    <script type="text/javascript">
        // Django paginator의 속성들
        // .number = 현재 페이지 값을 리턴
        const currentPage = {{ posts.number }}
        // .paginator.num_pages = 끝 페이지 (페이지의 갯수) 리턴
        // (장고의 페이지는 0이 아닌 1부터 시작하기 때문에 두 값이 같다.)
        const lastPage = {{ posts.paginator.num_pages }}
        // .paginator.page_range = return range(1, num_pages+1)) (파이썬에서 for i in range() 사용 시 사용)
        // .paginator.count = 전체 게시물 개수
        // .paginator.per_page = 페이장 보여줄 개수
        // .previous_page_number / .next_page_number = 현재 페이지의 이전, 이후 페이지 인덱스
        // .has_previous / .has_next = 현재 페이지가 이전, 이후 페이지가 있는지

        // 페이지 그룹 내에서 몇개의 페이지를 보여줄 것인가.
        // 10이라면 1~10, 5라면 1~5
        const pageGroupSize = 10;
        // 그룹의 시작 인덱스. 만약 pageGroupSize가 10일떄 10페이지 에서는 1, 11페이지 에서는 11
        const startOfGroup = Math.floor((currentPage - 1) / pageGroupSize) * pageGroupSize + 1;

        // ul 태그이며 pagination 클래스 명을 가지는 엘리먼트 검색
        const paginationUl = document.querySelector('ul.pagination');
        // 반복문 밖에서 선언을 하는게 메모리 누수에는 도움을 주지 않지만(메모리 누수가 원래 없음)
        // <<와 >> page-item 때도 편하게 사용 하기 위해 미리 선언 했습니다.
        let pageItemLi, pageLinkA;

        // 조건문: 그룹의 마지막(10번째) i에 도달할 때 까지 && 도달하지 않았더라도 마지막 페이지 까지만 출력
        for(let i = startOfGroup; i <= startOfGroup + pageGroupSize - 1 && i <= lastPage; i++) {
            pageItemLi = document.createElement('li');
            pageItemLi.classList.add('page-item');
            if(i === currentPage) {
                // 현재 페이지의 경우 클릭 기능을 비활성화 하기 위해 span 태그로 생성한다.
                pageLinkA = document.createElement('span');
                pageLinkA.classList.add('page-link', 'active');
            }
            else {
                pageLinkA = document.createElement('a');
                pageLinkA.classList.add('page-link');
                // 아래 코드처럼 하드코딩 하여도 상관은 없으나 우리는 page_detail.js 처럼 클릭 이벤트 리스너를 추가한다.
                //pageLinkA.href = '/?page=' + i;
                pageLinkA.href = "javascript:void(0)";
                pageLinkA.dataset.page = i.toString();
            }
            pageLinkA.innerText = i.toString();
            pageItemLi.appendChild(pageLinkA);
            paginationUl.appendChild(pageItemLi);
        }

        // 페이지 그룹의 시작 인덱스가 1보다 크다면. (-10 그룹이 존재 한다면)
        if (startOfGroup > 1) {
            pageItemLi = document.createElement('li');
            pageItemLi.classList.add('page-item');
            // pageItemLi에 자식으로 a 태그를 추가하고 그 엘리먼트를 리턴으로 받음
            pageLinkA = pageItemLi.appendChild(document.createElement('a'));
            pageLinkA.classList.add('page-link');
            pageLinkA.innerText = "<<";
            pageLinkA.href = "javascript:void(0)";
            // 11페이지의 경우 <<를 클릭하면 10페이지, 21페이지의 경우 20페이지로 이동하는 링크
            pageLinkA.dataset.page = (startOfGroup - 1).toString();

            paginationUl.prepend(pageItemLi);
        }

        // 페이지 그룹의 마지막 인덱스 뒤에 페이지가 더 있을 경우
        // (11~20 페이지 그룹에서 마지막 페이지가 21이상인 경우)
        if (startOfGroup + pageGroupSize <= lastPage) {
            pageItemLi = document.createElement('li');
            pageItemLi.classList.add('page-item');

            pageLinkA = pageItemLi.appendChild(document.createElement('a'));
            pageLinkA.classList.add('page-link');
            pageLinkA.innerText = ">>";
            pageLinkA.href = "javascript:void(0)";
            // 11~20 페이지의 경우 >>를 클릭하면 21페이지
            pageLinkA.dataset.page = (startOfGroup + pageGroupSize).toString();

            paginationUl.append(pageItemLi);
        }

        // span 태그(현재 페이지)를 제외하고 page-link 선택
        // href 대신 클릭 이벤트 리스너를 사용하는 이유는 나중에 검색기능을 추가 했을 때 (쿼리 파리미터 추가)
        // href 로는 구현이 귀찮아지기 때문에 미리 이벤트 리스너 방식을 사용합니다.
        const pageLinks = document.querySelectorAll('a.page-link');
        pageLinks.forEach((pageLink) => {
            pageLink.addEventListener('click', (event) => {
                event.preventDefault();
                location.href = "/?page=" + pageLink.dataset.page;
            })
        })
    </script>
</div>
{% endblock body %}
```

#### 결과 확인
![스크린샷](/statics/11/11_04.png)
![스크린샷](/statics/11/11_05.png)
![스크린샷](/statics/11/11_06.png)

---

### 스크립트를 스태틱 파일로 분리하기
post_detail.html과 .js 처럼 스크립트를 분리해봅시다.

post_list.js
```javascript
// 오류 발생
const currentPage = {{ posts.number }}
const lastPage = {{ posts.paginator.num_pages }}

const pageGroupSize = 10;
const startOfGroup = Math.floor((currentPage - 1) / pageGroupSize) * pageGroupSize + 1;
...
```
IDE에서 개발을 하고 계신다면 아마 {{ post.*** }} 두개의 템플릿 태그에서 오류가 뜰 것입니다.
그 이유는 템플릿 태그는 템플릿에서만 쓸 수 있기 때문입니다.

이 상황을 해결하는 방법 중 하나는 템플릿에서 변수를 선언하고 값을 할당하는 방법입니다.

post_list.html
```html
    <div class="row">
        <div class="col">
            <a href="{% url 'forum:post_create' %}" class="btn btn-primary">게시물 작성</a>
        </div>
        <nav class="col">
            <ul class="pagination justify-content-center">
                
            </ul>
        </nav>
        <div class="col">

        </div>
    </div>
    <script type="text/javascript">
        // 템플릿 변수가 필요한 변수의 선언과 할당을 템플릿의 <script> 태그에서 하고
        const currentPage = {{ posts.number }}
        const lastPage = {{ posts.paginator.num_pages }}
    </script>
    {% load static %}
    <!-- 스태틱 파일을 불러온다면 손쉽게 해결하실 수 있습니다. -->
    <script src="{% static 'js/post_list.js' %}"></script>
</div>
{% endblock body %}
```

---

### 함수화
post_list.js
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

const paginationUl = document.querySelector('ul.pagination');

const pageGroupSize = 10;
const startOfGroup = Math.floor((currentPage - 1) / pageGroupSize) * pageGroupSize + 1;

for(let i = startOfGroup; i <= startOfGroup + pageGroupSize - 1 && i <= lastPage; i++) {
    const isCurrentPage = i === currentPage;

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

const pageLinks = document.querySelectorAll('a.page-link');
pageLinks.forEach((pageLink) => {
    pageLink.addEventListener('click', (event) => {
        event.preventDefault();
        location.href = "/posts/?page=" + pageLink.dataset.page;
    })
})
```