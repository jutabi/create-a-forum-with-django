## 18. 기능별 views 분리하기

[참고 문헌: 점프 투 장고 / 3-10 views.py 파일 분리](https://wikidocs.net/71657)

우리 프로젝트 forum앱의 views를 보면 함수가 벌써 6개나 되는데 앞으로 댓글의 CRUD와 추천기능까지 추가된다면 더 늘어나게 됩니다. 
게다가 Post, Comment, Vote의 기능들을 하나의 .py에서 관리하게 됩니다. 이것은 유지보수와 기능 별 관리 측면에서 불리함을 가집니다. 그렇기 때문에 이제 기능 별로 views를 나누어 봅시다.

### \_\_init__.py 이용하기
#### 선행지식
Django는 기본적으로 각 앱의 views\.py 파일을 뷰 모듈로 인식합니다.
하지만 views가 디렉토리여도 그 안에 \_\_init__.py파일이 있고 분리된 뷰 파일을 import 해준다면 단일 views.py를 사용할 때와 똑같은 사용이 가능합니다.

#### views 디렉토리 생성
/forum/views/디렉토리를 생성합니다. 
(파이참 사용시 '경로'대신 'Python 패키지'를 선택하여 '\_\_init__.py'를 미리 자동으로 생성해도 됩니다.)
![스크린샷](/statics/18/18_01.png)

#### 기능별 views 파일 생성
![스크린샷](/statics/18/18_02.png)

##### 각 파일의 내용을 작성합니다.
base_views.py
```python
from django.contrib.auth.decorators import login_required
from django.core.paginator import Paginator
from django.http import HttpResponseForbidden, HttpResponseNotAllowed
from django.shortcuts import render, get_object_or_404, redirect

from forum.forms import PostForm, CommentForm
from forum.models import *

def post_list(request):
    posts = Post.objects.order_by('-created_date')
    request_page = request.GET.get('page', 1)

    paginator = Paginator(posts, 10)
    page_obj = paginator.get_page(request_page)

    context = {'posts': page_obj}

    return render(request, 'forum/post_list.html', context)
```
기존의 views.py에서 post_list(index page)의 내용을 복사해주고 사용하지 않는 import 문을 제거해 줍니다. 
IDE 사용 시 회색처리된 (사용되지 않은) import 문을 삭제합니다. pycharm 사용시 Ctrl+Alt+O 를 사용해 쉽게 최적화 할 수 있습니다.

##### 주의
기존에 from forum.models 와 forum.forms 가 아닌 '.forms', '.models'처럼 상위 디렉토리 참조를 사용하여 모듈을 임포트 했다면 '..views', '..forms'처럼 두번 상위 참조 해야 합니다. (디렉토리가 하나 추가되었음으로)

post_views.py
```python
from django.contrib.auth.decorators import login_required
from django.http import HttpResponseForbidden
from django.shortcuts import render, get_object_or_404, redirect

from forum.forms import PostForm
from forum.models import *


def post_detail(request, post_pk):
    ...

@login_required
def post_create(request):
    ...

def post_update(request, post_pk):
    ...

def post_delete(request, post_pk):
    ...
```

comment_views.py
```python
from django.http import HttpResponseNotAllowed
from django.shortcuts import render, get_object_or_404, redirect

from forum.forms import CommentForm
from forum.models import *


def comment_create(request, post_pk):
    ...
```

#### views\.py 삭제 후 서버 가동
이제 내용이 빈 views.py를 삭제하고 서버를 가동해 봅시다.
```
(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py runserver
Watching for file changes with StatReloader
Performing system checks...

Exception in thread django-main-thread:
Traceback (most recent call last):
  ...
  File "<frozen importlib._bootstrap>", line 1387, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1360, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1331, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 935, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 1026, in exec_module
  File "<frozen importlib._bootstrap>", line 488, in _call_with_frames_removed
  File "C:\Users\***\PycharmProjects\forum-with-django\forumwithdjango\urls.py", line 22, in <module>
    path('', forum_views.post_list),
             ^^^^^^^^^^^^^^^^^^^^^
AttributeError: module 'forum.views' has no attribute 'post_list'
```
여러 줄의 오류가 뜨지만 맨 마지막 urls.py를 봅시다. forum앱의 views에서 post_list 함수를 찾을 수 없다고 하는데 urls.py를 한번 봅시다.
![스크린샷](/statics/18/18_03.png)
모든 뷰함수에서 찾을 수 없다는 표시가 뜹니다. 부제를 보고 당연히 '\_\_init__.py가 없어서'라는 이유는 알겠지만 왜? 라는 생각이 듭니다.

##### \_\_init__.py의 의미
\_\_init__.py은 파이썬에서 해당 디렉토리가 '패키지'임을 선언하는 역할을 합니다.
즉, 디렉토리에 \_\_init__.py가 존재한다면 파이썬은 그 디렉토리를 파이썬 패키지로 인식하여 import 문으로 하위 모듈이나 함수를 불러올 수 있도록 합니다.
- 주요 역할
    - 패키지 선언
        - 하위 모듈이나 클래스를 import 해두어 패키지 외부에서 간단하게 접근이 가능하도록 합니다.
    - 초기화 코드 실행
        - 패키지를 import 할 시 실행할 초기화 코드를 넣을 수 있습니다.

#### \_\_init__.py 작성
이제 \_\_init__.py을 작성해 봅시다.
```python
from forum.views.base_views import *
# 위처럼 최상위 앱에서 views/base_views.py를 불러와도 되고
from .post_views import *
# 이처럼 현 디렉토리 기준 형제 파일들을 불러와도 됩니다.
from .comment_views import *
```
\_\_init__.py에 하위 모듈들을 import 합니다.

이제 서버를 실행해 보면 정상작동 하는 것을 보실 수 있습니다.

---

### \_\_init__.py 삭제

그런데 다시 한번 urls.py를 살펴봅시다.
![스크린샷](/statics/18/18_04.png)

정상 작동하는 것은 알겠으나 우리는 기능별로 뷰 파일을 분류한 이유는 파일 하나에서 모든 기능을 구현하다 보니 코드가 길어진다는 단점 때문도 있었지만 유지보수에도 이득을 가져가기 위해서 분리를 진행했습니다.

그러나 우리의 urls\.py를 보면 views.post_*** / views.comment_*** 처럼 어느 뷰 파일에 존재하는지는 명확하게 구분이 불가능합니다. 그저 하나의 views\.py 파일에 작성된 것 처럼 보입니다.

물론 우리가 FBV 메서드 이름을 post_ comment_ 식으로 명시하긴 했지만 이러한 메서드들만 존재하는 것이 아닙니다.
그래서 우리는 \_\_init__.py를 삭제하고 urls\.py 에서 모듈의 이름을 명시적으로 임포트 할 것 입니다.

#### forumwithdjango/urls.py
```python
from django.contrib import admin
from django.urls import path, include
# from forum import views as forum_views
from forum.views import base_views
# 기존의 views.py 임포트 대신 base_views를 임포트 합니다.

urlpatterns = [
    path('', base_views.post_list),
    path('admin/', admin.site.urls),
    path('posts/', include('forum.urls')),
    path('member/', include('member.urls')),
]
```

#### forum/urls.py
```python
from django.urls import path
from forum.views import base_views, post_views, comment_views

app_name = 'forum'

urlpatterns = [
    path('', base_views.post_list, name='post_list'),

    path('new/',                  post_views.post_create, name='post_create'),
    path('<int:post_pk>/',        post_views.post_detail, name='post_detail'),
    path('<int:post_pk>/update/', post_views.post_update, name='post_update'),
    path('<int:post_pk>/delete/', post_views.post_delete, name='post_delete'),
    # 파이썬은 코드 내 공백이 문법적으로 문제되지 않습니다. 가독성을 위해 정렬하는 것은 좋은 방법입니다.
    # 들여쓰기는 당연히 문법적으로 중요하니 맞춰야 합니다.

    path('<int:post_pk>/comments/new/', comment_views.comment_create, name='comment_create'),
]
```

이제 urls.py에서 어떤 뷰파일이 메서드를 담당하고 있는지 명시적으로 확인이 가능해졌습니다.