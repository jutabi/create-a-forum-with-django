## 26. 게시물 검색

### 서버 측 작업
#### base_views.py
```python
def post_list(request):
    posts = (
        Post.objects
        .select_related('author')
        .annotate(voted_users_count=Count('voted_users'))
        .order_by('-created_date')
    )
    request_page = request.GET.get('page', 1)
    # 검색, 페이징의 경우 GET 요청의 쿼리 파라미터에 정보를 담아 요청합니다.
    search_keyword = request.GET.get('search_keyword', '')

    # 만약 사용자가 검색을 요청했다면:
    if search_keyword:
        # posts 쿼리셋에서 title속성의 값이 search_keyword 값이 포함되어 있는지를 필터를 겁니다.
        # contains => 대소문자를 구분하여 포함 여부를 검사
        # icontains => 대소문자에 상관 없이 포함 여부를 검사
        posts = posts.filter(title__icontains=search_keyword)

    paginator = Paginator(posts, 10)
    page_obj = paginator.get_page(request_page)

    context = {'posts': page_obj}
    return render(request, 'forum/post_list.html', context)
```

### 클라이언트 측 작업
#### post_list.html
```html
{% extends 'base.html' %}
{% block title %}
Forum
{% endblock title %}

{% block body %}
<div class="container">
    ...
    <div class="row">
        <div class="col">
            <!-- 검색 -->
            <a href="{% url 'forum:post_create' %}" class="btn btn-primary">게시물 작성</a>
        </div>
        <nav class="col">
            <!-- 페이징 -->
            <ul class="pagination justify-content-center">
            </ul>
        </nav>
        <div class="col text-end">
            <!-- 검색 영역 / 내부 요소들을 한줄로 배치하기 위해 inline-flex display를 사용했습니다. -->
            <form class="d-inline-flex">
                <label class="me-1">
                    <input id="search-keyword-input" class="form-control" name="search_keyword">
                </label>
                <button type="submit" class="btn btn-outline-success">Search</button>
            </form>
        </div>
    </div>
</div>
...
{% endblock body %}
```
이렇게만 작성된 상태로 검색을 진행 해 봅시다.
![스크린샷](/statics/26/26_01.png)
폼 태그 내에 액션 속성이 없기 때문에 현재 url에 그대로 \<form>요청을 보내게 되어 정상작동 하는 것을 확인하실 수 있습니다.
다만 페이징이 제대로 동작하는 것 같지만 실제로 눌러보면:

![스크린샷](/statics/26/26_02.png)
쿼리 파라미터에 검색어 적용이 없이 페이징 요청이 되어있는 것을 보실 수 있습니다.

![스크린샷](/statics/26/26_03.png)
그래도 직접 url에 쿼리파라미터를 전송해 주면 정상 작동을 확인할 수 있는데요. 이제 JS내에서 페이징 관련 코드를 수정해 봅시다.

#### post_list.js
```js
const pageLinks = document.querySelectorAll('a.page-link');
pageLinks.forEach((pageLink) => {
    pageLink.addEventListener('click', (event) => {
        event.preventDefault();
        // location.href = `/posts/?page=${pageLink.dataset.page}`;

        // 댓글 페이징 시 사용했던 URLSearchParams를 다시 사용해 봅시다.
        const queryParams = new URLSearchParams();
        queryParams.append('page', pageLink.dataset.page)

        location.href = `/posts/?${queryParams}`;
    })
})
```
먼저 기존 코드를 URLSearchParams를 사용하는 방법으로 변경하고 정상 작동을 확인합니다.
```js
const pageLinks = document.querySelectorAll('a.page-link');
pageLinks.forEach((pageLink) => {
    pageLink.addEventListener('click', (event) => {
        event.preventDefault();

        // window.location.search는 현재 쿼리 파라미터를 스트링값으로 반환해 주는데요.
        // 이를 URLSearchParams에 인자로 넣어주면 그 key:value값을 삽입한 객체를 반환해 줍니다.
        // 만약 검색을 하고 페이지 버튼을 클릭하면 search_keyword[key]가 queryParams에 생성됩니다.
        const queryParams = new URLSearchParams(window.location.search);
        // append를 사용하면 기존의 loc.search내에 있던 데이터와 겹쳐 중복 쿼리스트링이 생성됩니다.
        // set을 사용하여 존재하지 않는다면 생성 후 set, 존재한다면 내용 변경을 해 줍니다.
        queryParams.set('page', pageLink.dataset.page)

        location.href = `/posts/?${queryParams}`;
    })
})
```
그리고 위의 코드로 변경하여 페이징과 검색기능이 동시에 작동하는지 확인해 봅시다.
#### 결과 확인
![스크린샷](/statics/26/26_04.png)

#### 검색창에 기존 내용 적용하기
사용자 경험(UX) 향상을 위해, 검색 시 사용자가 입력한 검색어를 검색창에 유지하여, 사용자가 현재의 검색 컨텍스트를 직관적으로 파악할 수 있도록 하는 것이 좋습니다.
```js
import {makePagination} from '/static/js/paging.js'
// 공통으로 사용하기 때문에 전역변수
const queryParams = new URLSearchParams(window.location.search);

makePagination(currentPage, lastPage);

const pageLinks = document.querySelectorAll('a.page-link');
pageLinks.forEach((pageLink) => {
    pageLink.addEventListener('click', (event) => {
        event.preventDefault();

        queryParams.set('page', pageLink.dataset.page)

        location.href = `/posts/?${queryParams}`;
    })
})

if (queryParams.has('search_keyword')) {
    document.querySelector('#search-keyword-input').value = queryParams.get('search_keyword');
}
```

---

## search_target 적용하기
많은 웹 커뮤니티는 검색기능에 제목만 검색되지 않습니다. 
이번엔 제목, 글쓴이, 제목+작성내용의 검색 타겟을 선택할 수 있도록 해봅시다.

### 서버 측 작업
#### base_views\.py
```python
def post_list(request):
    posts = (
        Post.objects
        .select_related('author')
        .annotate(voted_users_count=Count('voted_users'))
        .order_by('-created_date')
    )
    request_page = request.GET.get('page', 1)
    search_keyword = request.GET.get('search_keyword', '')
    search_target = request.GET.get('search_target', '')

    if search_keyword:
        if search_target == 'title':
            posts = posts.filter(title__icontains=search_keyword)
        elif search_target == 'nickname':
            posts = posts.filter(author__nickname__icontains=search_keyword)
        elif search_target == 'title_content':
            # 장고에서 filter()에 OR조건을 걸고 싶다면 Q객체를 사용해야 합니다.
            # filter(A | B)의 형식은 불가능합니다. filter(A, B)를 사용하여 AND조건을 걸수는 있습니다.
            posts = posts.filter(Q(title__icontains=search_keyword) | Q(content__icontains=search_keyword))

    paginator = Paginator(posts, 10)
    page_obj = paginator.get_page(request_page)

    context = {'posts': page_obj}
    return render(request, 'forum/post_list.html', context)
```

### 클라이언트 측 작업
#### post_list.html
```html
<div class="col text-end">
    <form class="d-inline-flex">
        <label>
            <input id="search-keyword-input" class="form-control" name="search_keyword">
        </label>
        <label class="me-1">
            <!-- 제목+내용 옵션이 잘려보이는 현상이 있을 수 있어 w-auto 클래스를 추가하였습니다. -->
            <select id="search-target-select" class="form-select w-auto" name="search_target">
                <option id="title-option" value="title">제목</option>
                <option id="nickname-option" value="nickname">닉네임</option>
                <option id="title_content-option" value="title_content">제목+내용</option>
            </select>
        </label>
        <button type="submit" class="btn btn-outline-success">Search</button>
    </form>
</div>
```

#### post_list.js
```js
if (queryParams.has('search_keyword')) {
    document.querySelector('#search-keyword-input').value = queryParams.get('search_keyword');
    // 검색어 유지와 같이 search_target select태그 또한 기존의 선택값을 유지시킵니다.
    // select태그의 value에 값을 입력하면 하위 option태그의 value속성 값과 매칭됩니다.
    document.querySelector('#search-target-select').value = queryParams.get('search_target');
    //or
    document.querySelector('#search-target-select').selectedIndex =
        document.querySelector(`#${queryParams.get('search_target')}-option`).index;
    // 두번째 방법을 사용하는 것이 아니라면 option태그의 id속성을 제거해도 됩니다.
}
```

#### 결과 확인
![스크린샷](/statics/26/26_05.png)

---

### + 최소 글자수 제한 기능

#### Model 단
```python
class Search(models.Model):
    keyword = models.CharField(validators=[MinLengthValidator(1)], max_length=100)
```

#### Form 단
```python
class SearchForm(forms.Form):
    search_keyword = forms.CharField(
        min_length=2,
        max_length=100,
    )
```

#### views 단
```python
def post_list(request):
    ...

    if search_keyword:
        if len(search_keyword) < 2:
            return HttpResponseForbidden('Too short')
            
        if search_target == 'title':
            posts = posts.filter(title__icontains=search_keyword)
        elif search_target == 'nickname':
            posts = posts.filter(author__nickname__icontains=search_keyword)
        elif search_target == 'title_content':
            posts = posts.filter(Q(title__icontains=search_keyword) | Q(content__icontains=search_keyword))

```