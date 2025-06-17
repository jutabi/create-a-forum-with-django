## 01. 초기 설정

#### 1. 파이썬 설치

##### windows
[www.python.org](www.python.org)
`python -V`

##### linux
```
sudo apt update
sudo apt install python3
python3 -V
```

맥과 리눅스는 python2가 기본적으로 설치되어 있음으로 'python3'명시

---

#### 2. 파이썬 가상 환경 설정
##### 프로젝트의 루트 폴더에 '.venv'로 가상환경을 생성하는 것이 일반적인 방법이기에 프로젝트 디렉토리 생성
##### 깃허브로 프로젝트 버전관리를 하기 위해 폴더명에 '-'삽입
```
C:\Users\***>cd \

C:\>mkdir DjangoProjects

C:\>cd DjangoProjects

C:\DjangoProjects> mkdir forum-with-django

C:\DjangoProjects>cd forum-with-django
```
##### 프로젝트 디렉토리 내에서 (최상위) 가상환경 설치
```
C:\DjangoProjects\forum-with-django>python -m venv .venv

C:\DjangoProjects\forum-with-django>cd .venv\Scripts

C:\DjangoProjects\forum-with-django\.venv\Scripts>activate    
```
##### 가상환경 접속 성공시:
```
(.venv) C:\DjangoProjects\forum-with-django\.venv\Scripts>
```
##### 나가려면 환경 내에서 deactivate

> 가상 환경에 접속하면 linux / mac 에서 'python', 'pip' 명령어 사용 가능

---

#### 3. 장고 설치
```
(.venv) C:\DjangoProjects\forum-with-django\.venv\Scripts>pip install django
Collecting django
  Downloading django-5.2.2-py3-none-any.whl.metadata (4.1 kB)
Collecting asgiref>=3.8.1 (from django)
  Downloading asgiref-3.8.1-py3-none-any.whl.metadata (9.3 kB)
Collecting sqlparse>=0.3.1 (from django)
  Downloading sqlparse-0.5.3-py3-none-any.whl.metadata (3.9 kB)
Collecting tzdata (from django)
  Downloading tzdata-2025.2-py2.py3-none-any.whl.metadata (1.4 kB)
Downloading django-5.2.2-py3-none-any.whl (8.3 MB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 8.3/8.3 MB 11.6 MB/s eta 0:00:00
Downloading asgiref-3.8.1-py3-none-any.whl (23 kB)
Downloading sqlparse-0.5.3-py3-none-any.whl (44 kB)
Downloading tzdata-2025.2-py2.py3-none-any.whl (347 kB)
Installing collected packages: tzdata, sqlparse, asgiref, django
Successfully installed asgiref-3.8.1 django-5.2.2 sqlparse-0.5.3 tzdata-2025.2

[notice] A new release of pip is available: 24.3.1 -> 25.1.1
[notice] To update, run: python.exe -m pip install --upgrade pip

(.venv) C:\DjangoProjects\forum-with-django\.venv\Scripts>  
```

##### 마지막 줄 경고 내용에 따라 pip 업데이트
```
(.venv) C:\DjangoProjects\forum-with-django\.venv\Scripts>django-admin --version
5.2.2

(.venv) C:\DjangoProjects\forum-with-django\.venv\Scripts>python -m pip install --upgrade pip
Requirement already satisfied: pip in c:\djangoprojects\forum-with-django\.venv\lib\site-packages (24.3.1)
Collecting pip
  Downloading pip-25.1.1-py3-none-any.whl.metadata (3.6 kB)
Downloading pip-25.1.1-py3-none-any.whl (1.8 MB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 1.8/1.8 MB 11.0 MB/s eta 0:00:00
Installing collected packages: pip
  Attempting uninstall: pip
    Found existing installation: pip 24.3.1
    Uninstalling pip-24.3.1:
      Successfully uninstalled pip-24.3.1
Successfully installed pip-25.1.1

(.venv) C:\DjangoProjects\forum-with-django\.venv\Scripts>
```

---

#### 4. 장고 프로젝트 생성
```
(.venv) C:\DjangoProjects\forum-with-django\.venv\Scripts>cd ..\..

(.venv) C:\DjangoProjects\forum-with-django>django-admin startproject forumwithdjango .

(.venv) C:\DjangoProjects\forum-with-django>
```
##### 우리는 이미 프로젝트 폴더를 생성했기 때문에 프로젝트 명 뒤에 '.'을 입력해 현재 디렉토리에 프로젝트를 생성하도록 함.
##### 그렇지 않으면 C:\DangoProjects\forum-with-django\forumwithdjango\ 가 프로젝트의 루트 디렉토리가 된다.

##### Django 템플릿 디렉토리 생성. (추후 설명)
```
(.venv) C:\DjangoProjects\forum-with-django>mkdir templates

(.venv) C:\DjangoProjects\forum-with-django>
```

---

#### 5. 장고 실행하기
```
(.venv) C:\DjangoProjects\forum-with-django>python manage.py runserver
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.
June 10, 2025 - 17:02:57
Django version 5.2.2, using settings 'forumwithdjango.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CTRL-BREAK.

WARNING: This is a development server. Do not use it in a production setting. Use a production WSGI or ASGI server instead.
For more information on production servers see: https://docs.djangoproject.com/en/5.2/howto/deployment/
```
##### 정상 작동 확인
![스크린샷](/statics/01/01_01.jpeg)

##### 안내된 주소(http://127.0.0.1:8000/)에 접속시 정상적으로 GET 요청과 반환이 이루어짐
```
[10/Jun/2025 17:03:16] "GET / HTTP/1.1" 200 12068
Not Found: /favicon.ico
[10/Jun/2025 17:03:16] "GET /favicon.ico HTTP/1.1" 404 2217
```

---

#### 6. Hello World
##### ctrl+c 를 입력하여 서버를 닫은 후 Django 앱 생성
```
(.venv) C:\DjangoProjects\forum-with-django>python manage.py startapp forum

(.venv) C:\DjangoProjects\forum-with-django>
```
##### cmd 내에서 에디터를 사용하기 보단 VSCode를 사용해보자. (뒤의 내용들은 파이참 사용 예정)
[https://code.visualstudio.com/download](https://code.visualstudio.com/download)

##### 설치 후 프로젝트 디렉토리 열기
![스크린샷](/statics/01/01_02.png)
![스크린샷](/statics/01/01_03.png)
##### 인터프리터 선택 (만들었던 .venv 가상 환경 선택)
![스크린샷](/statics/01/01_04.png)
##### 터미널 재시동까지 대기
![스크린샷](/statics/01/01_05.png)
##### 생성했던 앱 (forum) 폴더 내부의 'views.py' 수정
```python
def hello(request):
  return HttpResponse("Hello World!")
```
![스크린샷](/statics/01/01_06.png)
##### 프로젝트 설정 앱 (forumwithdjango) 폴더 내부의 'urls.py' 수정
```python
from forum import views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('forum/', views.hello)
]
```
![스크린샷](/statics/01/01_07.png)
##### 파일 저장 후 서버 재가동
##### Command Prompt 선택 후 서버 가동 (python manage.py runserver)
![스크린샷](/statics/01/01_08.png)
##### 정상 작동 확인
![스크린샷](/statics/01/01_09.png)

---

#### 7. Git
[https://git-scm.com](https://git-scm.com)

##### 프로젝트 최상위 폴더에 '.gitignore'생성 후 내용 작성:
```
db.sqlite3
*.pyc
__pycache__
```

##### 커밋하기

```
C:\DjangoProjects\forum-with-django>git init
Initialized empty Git repository in C:/DjangoProjects/forum-with-django/.git/

C:\DjangoProjects\forum-with-django>git status 
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   .gitignore
        new file:   forum/__init__.py
        new file:   forum/admin.py
        new file:   forum/apps.py
        new file:   forum/migrations/__init__.py
        new file:   forum/models.py
        new file:   forum/tests.py
        new file:   forum/views.py
        new file:   forumwithdjango/__init__.py
        new file:   forumwithdjango/asgi.py
        new file:   forumwithdjango/settings.py
        new file:   forumwithdjango/urls.py
        new file:   forumwithdjango/wsgi.py
        new file:   manage.py

C:\DjangoProjects\forum-with-django>git add *

C:\DjangoProjects\forum-with-django>git commit -m "Initial commit"
[master (root-commit) 117eb0e] Initial commit
 14 files changed, 225 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 forum/__init__.py
 create mode 100644 forum/admin.py
 create mode 100644 forum/apps.py
 create mode 100644 forum/migrations/__init__.py
 create mode 100644 forum/models.py
 create mode 100644 forum/tests.py
 create mode 100644 forum/views.py
 create mode 100644 forumwithdjango/__init__.py
 create mode 100644 forumwithdjango/asgi.py
 create mode 100644 forumwithdjango/settings.py
 create mode 100644 forumwithdjango/urls.py
 create mode 100644 forumwithdjango/wsgi.py
 create mode 100644 manage.py
```

---

#### 8. PyCharm

##### 지금 까지 했던 프로젝트 생성 작업 원클릭 세팅
![스크린샷](/statics/01/01_10.png)
![스크린샷](/statics/01/01_11.png)
> 로컬과 Command Prompt 상관 없이 사용