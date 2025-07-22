## 25. 추천 (비동기 요청) + N+1 쿼리 문제

[참고 문헌: 점프 투 장고 / 3-11 추천](https://wikidocs.net/71791)

### 1. 추천 수 응답 (서버 측 작업)

#### forum/models.py
```python
from django.contrib.auth.models import User
from django.db import models
from django.utils import timezone

from member.models import ForumUser

class Post(models.Model):
    author = models.ForeignKey(ForumUser, on_delete=models.CASCADE)
    title = models.CharField(max_length=100)
    content = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)
    updated_date = models.DateTimeField(null=True, blank=True)

    voted_users = models.ManyToManyField(ForumUser)

class Comment(models.Model):
    author = models.ForeignKey(ForumUser, on_delete=models.CASCADE)
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    content = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)
    updated_date = models.DateTimeField(null=True, blank=True)

    voted_users = models.ManyToManyField(ForumUser)

```
한 유저는 여러 게시물(댓글)에 추천할 수 있고 / 한 게시물(댓글)에 여러 유저가 추천할 수 있습니다. 
그렇기 때문에 ***다대다 관계 필드***를 사용하여 '추천인들' 필드를 Post, Comment 필드의 속성에 추가합니다.
그리고 마이그레이션을 진행해 봅시다.
```
(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py makemigrations
SystemCheckError: System check identified some issues:

ERRORS:
forum.Comment.author: (fields.E304) Reverse accessor 'ForumUser.comment_set' for 'forum.Comment.author' clashes with reverse accessor for 'forum.Comment.voted_users'.
        HINT: Add or change a related_name argument to the definition for 'forum.Comment.author' or 'forum.Comment.voted_users'.
forum.Comment.voted_users: (fields.E304) Reverse accessor 'ForumUser.comment_set' for 'forum.Comment.voted_users' clashes with reverse accessor for 'forum.Comment.author'.
        HINT: Add or change a related_name argument to the definition for 'forum.Comment.voted_users' or 'forum.Comment.author'.
forum.Post.author: (fields.E304) Reverse accessor 'ForumUser.post_set' for 'forum.Post.author' clashes with reverse accessor for 'forum.Post.voted_users'.
        HINT: Add or change a related_name argument to the definition for 'forum.Post.author' or 'forum.Post.voted_users'.
forum.Post.voted_users: (fields.E304) Reverse accessor 'ForumUser.post_set' for 'forum.Post.voted_users' clashes with reverse accessor for 'forum.Post.author'.
        HINT: Add or change a related_name argument to the definition for 'forum.Post.voted_users' or 'forum.Post.author'.
```
마이그레이션 생성 오류를 확인하실 수 있는데요.
02 차시에서 모델 생성 시 Post와 1:N관계를 가지는 author속성을 이용해 우리가 뷰나 템플릿 변수에서 'User.post_set.all()'메서드를 사용하여 해당 유저가 작성한 Posts를 불러오는 기능을 사용할 수 있음을 확인했습니다.
그러나 이제는 voted_users속성이 추가되면서 User의 post_set을 사용하면 장고가 ***유저가 작성한 Posts***를 보여줘야 하는지 ***유저가 추천한 Posts***를 보여줘야 하는지 알 수 없게 됩니다.
그래서 우리는 related_name 매개변수를 사용하여 서로를 구분할 수 있는 메서드명 스트링 값을 전달해 주어야 합니다.

```python
...
class Post(models.Model):
    author = models.ForeignKey(ForumUser, on_delete=models.CASCADE, related_name='posts')
    ...
    voted_users = models.ManyToManyField(ForumUser, related_name='voted_posts')
    def __str__(self):
        return self.title

class Comment(models.Model):
    author = models.ForeignKey(ForumUser, on_delete=models.CASCADE, related_name='comments')
    ...
    voted_users = models.ManyToManyField(ForumUser, related_name='voted_comments')
```
위처럼 related_name에 원하는 메서드 명을 전달할 수 있습니다. 기존과 같이 User.voted_comments.all() 메서드 형태로 접근 가능합니다.

위처럼 author 속성에도 related_name에 인자를 전달 해 주면 User.posts.all()을 이용해 접근할 수 있고, 
만약 전달하지 않았다면 기존처럼 User.post_set.all()을 이용해 작성한 게시물을 불러오고, User.voted_posts.all()을 이용해 추천한 게시물을 불러올 수 있습니다.

이제 마이그레이션을 진행해 줍시다.
```
(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py makemigrations
Migrations for 'forum':
  forum\migrations\0005_comment_voted_users_post_voted_users_and_more.py
    + Add field voted_users to comment
    + Add field voted_users to post
    ~ Alter field author on comment
    ~ Alter field author on post

(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, forum, member, sessions
Running migrations:
  Applying forum.0005_comment_voted_users_post_voted_users_and_more... OK
```

#### 결과 확인
장고 쉘에서 해당 메서드가 정상 작동하는지 확인해 봅시다.
```
(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py shell
8 objects imported automatically (use -v 2 for details).

Python 3.13.1 (tags/v3.13.1:0671451, Dec  3 2024, 19:06:28) [MSC v.1942 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>> from member.models import ForumUser
>>> user1 = ForumUser.objects.get(pk=1)

>>> user1.posts.all()
<QuerySet [<Post: 1234>, <Post: 2345>, <Post: 3456>]>

>>> user1.voted_posts.all()
<QuerySet []>
>>>
```

#### forum/comment_views.py
post_views는 템플릿 변수를 사용하기 때문에 템플릿 내에서 .count()메서드를 사용하여 추천 수를 출력할 수 있습니다.
그러나 comment_views는 JSON형식으로 응답해주기 때문에 응답 내용에 추천 수를 동봉해 봅시다.
- Post의 경우 템플릿 내부에서 {{ post.voted_users.count }}와 같이 접근 가능합니다.
```python
def get_comments(post_pk, page):
    comments = Comment.objects.filter(post=post_pk).values(
        'pk', 'author__pk', 'author__username', 'author__nickname', 'content', 'created_date', 'updated_date')

    paginator = Paginator(comments, 10)
    if not page:
        page = paginator.num_pages
    page_obj = paginator.get_page(page)

    custom_field = []
    for comment in page_obj: #page_obj의 크기 만큼 반복문이 돔
        # 쿼리셋 내부 comment 각각의 pk속성 값을 통해 DB에 해당 객체의 내용을 쿼리하고
        temp_cmt = get_object_or_404(Comment, pk=comment['pk'])
        # 다시 voted_users의 집계(count)연산을 DB에 요청 쿼리를 보냅니다.
        voted_users_count = temp_cmt.voted_users.count()
        custom_field.append({
            'pk': comment['pk'],
            'author_pk': comment['author__pk'],
            'username': comment['author__username'],
            'nickname': comment['author__nickname'],
            'content': comment['content'],
            'voted_users.count': voted_users_count,
            'created_date': comment['created_date'],
            'updated_date': comment['updated_date'],
        })

    context = {
        'comments': custom_field,
        'last_page': paginator.num_pages,
    }
    return context
```
위와 같은 방법을 사용하면 count같은 연산도 쿼리셋이나 리스트에 포함시킬 수 있습니다.
다만 위의 방법처럼 반복문 내에서 Comment 모델 객체 인스턴스를 생성하면 문제점이 몇가지 존재하는데요.
1. 불필요한 모델 인스턴스를 만들지 않고 DB만으로 count할 수 있습니다.
    - 집계(count, sum, avg ...)를 할때 모델 인스턴스 생성 없이 annotate()를 통해 DB에서 처리시킬 수 있습니다.
    - DB는 집계 연산에 특화되어 있습니다. 대량 데이터일 수록 반복문 내에서 인스턴스 생성은 속도차이가 커집니다.
    - 반복적인 모델 인스턴스 생성으로 서버의 리소스를 낭비합니다.
2. 2N+1 추가 쿼리 발생
    - 반복문 내에서 각각의 comment를 상대로 두번의 쿼리를 추가로 보내게 됩니다. (2N+1 추가 쿼리)
        - 첫 쿼리(values)[1] + 반복문 내 각각의 comment 쿼리[N] + 반복문 내 각각의 comment의 count연산[N]
    - 단일 쿼리로 요청할 수 있는 작업 2N+1 이라는 비효율적인 방법을 사용하게 됩니다.

성능적으로 보았을 때 데이터량이 많아질수록 value()와 annotate()를 이용하여 원하는 필드와 값을 DB선에서 처리해 가져오고
정말 필요한 경우에만 반복문 내에서 인스턴스를 생성하여 필요한 정보를 포함하는 것이 좋습니다.

#### annotate()를 이용한 뷰 함수
```python
from django.db.models import Count

def get_comments(post_pk, page):
    comments = (Comment.objects.filter(post=post_pk)
                # annotate를 사용하여 voted_users의 카운트를 DB에서 집계하고, 추가 필드를 생성합니다.
                .annotate(voted_users_count=Count('voted_users'))
                .values('pk', 'author__pk', 'author__username', 'author__nickname',
                        # 추가한 필드를 key값으로 하여 이 값도 values 결과에 포함시켜 쿼리셋에 저장합니다.
                        'content', 'voted_users_count', 'created_date', 'updated_date'))

    paginator = Paginator(comments, 10)
    if not page:
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
            'voted_users_count': comment['voted_users_count'],
            'created_date': comment['created_date'],
            'updated_date': comment['updated_date'],
        })

    context = {
        'comments': custom_field,
        'last_page': paginator.num_pages,
    }
    return context
```

##### + N+1 문제
특정 문제를 해결하기 위해 반복문을 사용 시 발생하는 문제 (특히 조회의 경우 추가적인 데이터 접근)
- 발생 예제
    ```python
    posts = Post.objects.all() # 초기 데이터 조회 (1 쿼리)

    for post in posts: # 반복문 내 추가 쿼리 (N 쿼리)
        print(post.author.username)
        # N개의 게시물에 대해 N번의 추가 데이터베이스 쿼리
        # 외래키인 author속성에 대해서 ORM은 외래키인 author의 pk값 만을 요청합니다.
        # 그래서 author 객체의 하위 속성인 username에 접근하기 위해서는 
        # 해당 pk값을 가지는 author객체를 쿼리하여 생성 후 해당 데이터에 접근하게 됩니다.
    ```
- 해결 방법
    ```python
    # 1. select_related 사용하기 (1:1, N:1)
        # ORM이 SQL쿼리 시 JOIN을 이용하여 연관 객체를 모두 불러옵니다.
        # Post의 author 처럼 단일 객체를 ForeignKey or OneToOne로 찾는 경우에 사용합니다.
    posts = Post.objects.select_related('author').all() # `SELECT...JOIN...`을 사용하여 한문장의 쿼리로 요청
    for post in posts:
        print(post.author.username) # 각각의 post의 author에 대한 정보를 같이 받아왔음으로 추가쿼리 발생 X

    # 2. prefetch_related 사용하기 (1:N, N:N)
        # ORM이 JOIN을 사용하여 한번의 쿼리 요청만 보내는 것이 아니라
        # 별도 쿼리로 연관 데이터들을 한번에 불러와 파이썬 내에서 매칭합니다. (2번 요청)
        # 접근하려는 하위 속성이 N인 경우에 사용합니다. (여러 객체를 찾는 경우)
            # ex. posts[N]:comments[N] SQL JOIN 시 row가 폭증하여 비효율 적입니다. 
            # 첫 번째 쿼리에서 Posts 객체 목록을 가져오고
            # 두 번째 쿼리에서 해당 Post들과 연결된 Comments 한번에 가져와 줍니다.
             # 그리고 파이썬에서 각 Post에 해당하는 Comments를 매핑해 줍니다.

        # 2.1 1:N 관계 (1+1 문제)
        author = ForumUser.objects.get(username='asdf1234') # 한개의 author에 대해 쿼리 (1)
        for post in author.posts.all(): # 그 author의 posts를 한번에 요청 쿼리 (1)
            print(post)

        author = ForumUser.objects.prefetch_related('posts').get(username='asdf1234')
        for post in author.posts.all():
            print(post)
        # 이런 1:N 관계의 경우 prefetch의 여부가 실질적 성능차이를 내진 않습니다.
        # 사용하지 않아도 .all()을 통해 한번의 추가 쿼리만 발생하고, prefetch를 사용해도 한번의 쿼리가 발생합니다.

        # 2.2 N:N 관계 (N+1 문제)
        authors = ForumUser.objects.all() # 여러 명의 authors에 대해 쿼리 (1)
        for author in authors: 
            for post in author.posts.all(): # 받아온 authors에서 각각의 author의 posts 요청 쿼리 (N)
                print(post)
            # or
            print(author.posts.all()) # 2중 반복문이 중요한게 아니고 각 author마다 posts를 쿼리한다는 것이 중요
                
        authors = ForumUser.objects.prefetch_related('posts').all()
        for author in authors:
            for post in author.posts.all():
                print(post)

    # 3. annotate, values
        # annotate: 반복문을 통한 집계가 필요한 경우 DB에서 집계 연산을 하도록 합니다.
        # values: 필요한 필드를 명시하여 요청합니다.
    ```

우리의 프로젝트에도 N+1 문제가 나타나는 곳이 존재합니다. 위의 예제처럼 post_list가 해당하는데요.
```python
def post_list(request):
    posts = Post.objects.order_by('-created_date')
    request_page = request.GET.get('page', 1)

    paginator = Paginator(posts, 10)
    page_obj = paginator.get_page(request_page)

    context = {'posts': page_obj}

    return render(request, 'forum/post_list.html', context)
```
```html
{% for post in posts %}
<tr>
    <td class="text-center">{{ post.pk }}</td>
    <td class="text-center">
        <a href="{% url 'forum:post_detail' post.pk %}">{{ post.title }}</a> [{{ post.comment_set.count }}]
    </td>
    <td class="text-center">{{ post.author.username }}</td>
    <td class="text-center">{{ post.created_date|date:"Y/m/d A h:i" }}</td>
</tr>
{% endfor %}
```
템플릿에서 반복문을 통해 posts를 탐색하고 그 내부에서 외래키 관계인 author의 username(혹은 nickname)하위속성에 접근하게 됩니다.
우리는 페이징을 통해 한페이지에 10개의 게시물만 전달하기 때문에 실제 체감되는 성능차는 미미할 수 있습니다.
다만 미래의 확장성(수가 늘거나, 다양한 정보를 참조)을 고려하거나 N+1 문제를 방지하기 위한 습관을 생각해서 post_list 뷰함수에 select_related를 사용해 봅시다.
```python
def post_list(request):
    posts = Post.objects.select_related('author').order_by('-created_date')
    ...
```

### 1. 추천 수 응답 (클라이언트 측 작업)

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
               class="btn btn-secondary mt-2">글 목록</a>
        </div>
        <!-- 글 목록 버튼과 수정,삭제 버튼 사이에 추천 버튼을 위치시킵니다. -->
        <div class="col text-center">
            <button class="btn btn-primary vote mt-2" data-url="">추천
                <span class="badge text-bg-secondary">{{ post.voted_users.count }}</span>
            </button>
        </div>
        <div class="col text-end">
            <!-- 수정과 삭제 버튼 -->
        </div>
    </div>
    ...
    <template id="comment-template">
        <div class="card mb-3">
            <div class="card-header">
                <!-- 카드 헤더를 row col 클래스를 사용하여 반으로 쪼개고 -->
                <div class="row">
                    <div class="col author"></div>
                    <div class="col vote text-end">
                        <!-- 카드 헤더의 오른쪽 상단에 추천 버튼을 위치시킵니다. -->
                        <button class="btn btn-primary btn-sm vote">추천
                            <span class="badge text-bg-secondary"></span>
                        </button>
                    </div>
                </div>
            </div>
            ...
        </div>
    </template>
...
</div>
...
```

#### post_detail.js
```js
function renderComments(comments, requestPage, lastPage) {
    const commentsDiv = document.querySelector('.comments');
    commentsDiv.innerHTML = '';
    comments.forEach(comment => {
        const cardDiv = document.querySelector('#comment-template').content.cloneNode(true);
        cardDiv.firstElementChild.id = `comment-${comment['pk']}`;

        cardDiv.querySelector('.card-header .author').append(`${comment['nickname']} (${comment['username']})`);
        cardDiv.querySelector('.card-header .vote button span').textContent = comment['voted_users_count']
    ...
```

#### 결과 확인
![스크린샷](/statics/25/25_01.png)

---

### 2. 추천 기능 (서버 측 작업)
```python

```

### 2. 추천 기능 (클라이언트 측 작업)
```js

```