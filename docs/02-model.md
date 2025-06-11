## 02. 모델 생성성

[참고 문헌: 점프 투 장고 / 2-02 모델](https://wikidocs.net/70650)

#### 1. timezone 설정
##### forumwithdjango/setting.py
```
# 템플릿과 폼에서만 자동 변환, REST API JSON 응답 시에는 return 전에 views.py 에서 변환 필요 (timezone.localtime())
LANGUAGE_CODE = 'ko-kr'

TIME_ZONE = 'Asia/Seoul'
```
Django에서는 모든 datetime을 UTC 로 저장하는 것이 원칙이다.
모델 생성시 models.DateTimeField 속성에 default=timezone.localtime() 을 설정 하더라도 Django는 내부적으로 UTC 로 저장하기 때문에 오류 발생 가능.
그래서 데이터베이스에 저장시에는 UTC(기본값)로 저장하고 사용자에게 response 할 때 local_time으로 변환하여 반환한다.

---

#### 2. 마이그레이션
'01. 초기설정'에서 작성한 urls.py 를 보면 'admin/'주소로 라우팅이 되어 있는 것을 확인할 수 있다.
그런데 막상 /admin/에 접속해보면 아래의 오류가 보여진다.
![스크린샷](/statics/02_01.png)
우리가 runserver 명령을 통해 서버를 가동했을 떄 표시되었던 오류를 보면
```
(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>python manage.py runserver
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.
June 11, 2025 - 15:07:21
Django version 5.2.2, using settings 'forumwithdjango.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CTRL-BREAK.
```
admin, auth, contenttypes, sessions 에 대해 마이그레이션이 필요하다는 안내를 한다. 이 앱들은 Django 프로젝트에서 기본으로 제공되는 어플리케이션인데 이들 중 'admin'어플리케이션이 '/admin/'경로에 해당하는 관리자 기능을 담당한다.
이 항목들은 Django 프로젝트에 포함된 앱(생성했던 'forum'앱 처럼)들인데 'forum/settings.py'에서 확인할 수 있다.
```
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```
안내된 앱들은 데이터베이스와 관련이 있는 앱들인데 해당 앱들의 테이블을 데이터베이스에 만들고 접근할 수 있도록 migrate가 필요한 것이다.

##### migrate
서버 실행시 안내된 마이그레이션 명령어를 실행한다.
```
(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying sessions.0001_initial... OK

(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>
```
Django 마이그레이션 시스템에 의해 필요한 테이블이 생성된다.

##### 결과 확인
![스크린샷](/statics/02_05.png)
Django가 기본적으로 제공하는 admin 관리자 페이지가 정상적으로 작동하는 것을 볼 수 있다.


##### 생성된 테이블을 보고싶다면:
db.sqlite3 파일을 더블클릭 하고 확인
![스크린샷](/statics/02_02.png)
##### 1. sql 쿼리문 입력 후 실행
![스크린샷](/statics/02_03.png)
##### 2. 파이참 데이터베이스 도구 사용
![스크린샷](/statics/02_04.png)

---

#### 3. forum의 모델 생성
##### 서비스 할 'forum'의 모델 구조
Post
| Post |
|------|
|title |
|content|
|created_date|

Comment
|Comment|
|-------|
| Post (foreign_key) |
|content|
|created_date|

> author, modified_date, voted_user, views 속성 추후 추가 예정

#### 3. /forum/models.py
```
from django.db import models
from django.utils import timezone

# Create your models here.

class Post(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)

class Comment(models.Model):
    // Post 모델에 대하여 외래키 생성, on_delete=models.CASCADE => 특정 Post가 삭제될 때 해당하는 댓글'들'도 같이 삭제됨
    // 1:N 관계
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    content = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)
```

##### /forumwithdjango/settings.py INSTALLED_APPS
01강 에서 생성한 앱을 settings 전역 설정에 추가한다.
```
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    'forum',
    # or
    # 'forum.apps.ForumConfig',
]
```
앱 이름인 'forum' 혹은 'forum'앱의 apps.py 에 서술된 'ForumConfig' 둘 중 아무 것이나 등록해도 상관 없다.


---

#### 4. makemigrations
##### migrate 명령으로 서술한 모델들 (Post, Comment)의 테이블을 만들기 전에 해주어야 하는 makemigrations
터미널에 'python manage.py makemigrations'를 입력하자.
```
(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>python manage.py makemigrations
Migrations for 'forum':
  forum\migrations\0001_initial.py
    + Create model Post
    + Create model Comment

(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, forum, sessions
Running migrations:
  Applying forum.0001_initial... OK

(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>
```
모델을 생성하거나 모델에 변경사항이 있다면 migrate를 하기 전 ***migration***을 만들어 주어야 한다.
makemigrations 명령은 models.py에 정의된 모델의 변경사항을 Django ORM이 migration파일을 생성한다.(/앱 이름/migrations/000X_XXXXX.py))
그리고 migrate 명령어를 실행하면 Django 마이그레이션 시스템이 생성된 migration파일에 작성된 operation 목록을 순서대로 실행하여 각각의 model에 해당하는 데이터베이스 테이블을 생성한다.

##### 생성된 마이그레이션 파일 (ex. 0001_initial.py)
```
class Migration(migrations.Migration):

    initial = True

    dependencies = [
    ]

    operations = [
        migrations.CreateModel(
            name='Post',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')),
                ('title', models.CharField(max_length=100)),
                ('content', models.TextField()),
                ('created_date', models.DateTimeField(default=django.utils.timezone.now)),
            ],
        ),
        migrations.CreateModel(
            name='Comment',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')),
                ('content', models.TextField()),
                ('created_date', models.DateTimeField(default=django.utils.timezone.now)),
                ('post', models.ForeignKey(on_delete=django.db.models.deletion.CASCADE, to='forum.post')),
            ],
        ),
    ]
```
마이그레이션 파일을 생성하고 migrate를 진행한다.

---

#### 5. django shell을 이용한 views.py 맛보기

##### 모델 데이터 생성
터미널에 'python manage.py shell'을 입력해보자.
```
(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>python manage.py shell
8 objects imported automatically (use -v 2 for details).

Python 3.13.1 (tags/v3.13.1:0671451, Dec  3 2024, 19:06:28) [MSC v.1942 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>>
```
여기서는 작업중인 프로젝트에 대한 파이썬 명령어를 사용할 수 있다.
테스트 디버그를 하는 것 처럼 파이썬 코드를 작성하여 Post 객체를 생성하고 데이터베이스에 저장해보자.
```
(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>python manage.py shell
8 objects imported automatically (use -v 2 for details).

Python 3.13.1 (tags/v3.13.1:0671451, Dec  3 2024, 19:06:28) [MSC v.1942 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)

>>> from forum.models import *
>>> post = Post(      
... title="test title 01",
... content="text content 01")
>>> post.save() 
>>>
```
> 'created_date'속성은 default 속성을 이용하여 기본값을 정해주었기 때문에 모델 객체 생성시 속성값을 명시하지 않더라도 자동으로 'timezone.now'에 해당하는 현재 시간값이 자동으로 저장된다.

##### 모델 객체 조회
생성한 데이터를 출력해보자.
```
>>> post.id
1
>>> post.content
'text content 01'
>>> post.title
'test title 01'

>>> Post.objects.all()
<QuerySet [<Post: Post object (1)>]>
>>>
```
Post.objects.all()메서드를 이용해 데이터베이스에 저장된 모든 데이터를 불러왔다.
'Post object (1)'처럼 id값이 personal key로 출력되는 것을 볼 수 있다.
이것을 'title'속성으로 변경하고 싶다면 models.py에서 모델에 "\__str\__"메서드를 오버라이딩 하면 된다.
```
class Post(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)
    
    def __str__(self):
        return self.title
```
shell 를 종료 후 재실행 하여 쿼리셋을 다시 불러와본다. (ctrl+z or quit() 입력으로 종료.)
```
(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>python manage.py shell
8 objects imported automatically (use -v 2 for details).

Python 3.13.1 (tags/v3.13.1:0671451, Dec  3 2024, 19:06:28) [MSC v.1942 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)

>>> from forum.models import *
>>> Post.objects.all()
<QuerySet [<Post: test title 01>]>
>>>
```
> 모델에 속성값이 아닌 메서드가 추가된 경우 마이그레이션 작업이 필요하지 않다. 

filter method를 이용하여 원하는 값 출력하기:
```
>>> from forum.models import *

>>> post = Post(      
... title="test title 02",
... content="test content 02")
>>> post.save()

>>> Post.objects.filter(id=2)
<QuerySet [<Post: test title 02>]>

>>> Post.objects.get(id=2)
<Post: test title 02>

>>> Post.objects.filter(title__contains="02")
<QuerySet [<Post: test title 02>]>
>>>
```
filter => 조건에 해당하는 데이터를 모두 리턴 (<QuerySet>)
get => 조건에 해당하는 단일 데이터 리턴 (<models.Model: >) (pk 값을 이용한 특정 객체 선택)
filter(***__contains="") => 속성값에 특정 값을 포함하고 있는 데이터들 모두 리턴

##### 모델 객체 속성값 변경
```
>>> post = Post.objects.get(id=1)
>>> post.title = "modified test title 01"
>>> post.save() 

>>> Post.objects.all() 
<QuerySet [<Post: modified test title 01>, <Post: test title 02>]>
>>>
```

#### 모델 객체 삭제
```
>>> post = Post.objects.get(id=1)
>>> post.delete() 
(1, {'forum.Post': 1})

>>> Post.objects.all()
<QuerySet [<Post: test title 02>]>
>>>
```

#### ForeignKey 속성값을 이용하기
```
>>> comment = Comment(
... post = Post.objects.get(id=2),
... content = "post02's comment")

>>> comment2 = Comment(
... post = Post.objects.get(id=2),
... content = "post02's comment2")

>>> comment.post
<Post: test title 02>

>>> comment.save()
>>> comment2.save()

// Post모델이 Comment모델의 속성에 외래키로 등록되었기 때문에 모델객체.모델_set 메서드가 사용 가능하다. (1:N 관계에서만 사용 가능.)
>>> post = Post.objects.get(id=2)
>>> post.comment_set.all() 
<QuerySet [<Comment: Comment object (1)>, <Comment: Comment object (2)>]>
>>>
```
> Post 객체의 comment_set 메소드는 모델 생성시 'related_name'속성 값으로 메소드명을 변경할 수 있다. (같은 모델을 참조하는 외래키가 여러개인 경우 필수

ex) Post의 속성에 외래키를 통한 작성자와 추천인들 속성이 있을 경우:
```
class Post(models.Model):
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='created_post')
    voted_users = models.ForeignKey(User, related_name='voted_post')
    ...
```
```
// 특정 user가 작성한 게시물들, 추천한 게시물들 중 어떠한 속성을 가리키는지 알 수 없다.
user.post_set.all() (X)

user.created_post.all() (O)
user.voted_post.all() (O)
```