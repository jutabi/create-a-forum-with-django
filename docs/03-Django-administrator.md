## 03. 장고 관리자

[참고 문헌: 점프 투 장고 / 2-03 장고 관리자](https://wikidocs.net/70718)

#### 1. 슈퍼 유저 생성
```
(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>python manage.py createsuperuser
사용자 이름 (leave blank to use 'mckay'): admin
이메일 주소: ***@google.com   
Password: 
Password (again): 
This password is too short. It must contain at least 8 characters.
비밀번호가 너무 일상적인 단어입니다.
비밀번호가 전부 숫자로 되어 있습니다.
Bypass password validation and create user anyway? [y/N]: y
Superuser created successfully.

(.venv) C:\Users\simi7\PycharmProjects\forum-with-django>

```
비밀번호에 대한 경고가 출력되지만 ***y***를 입력하여 강인할 수 있다.

---

#### 2. 관리자 페이지 접속
이전에 migrate를 통해 활성화한 '/admin/'주소에 접속한다.
![스크린샷](/statics/03_01.png)

설정했던 관리자 아이디와 비밀번호를 입력하고 접속.
![스크린샷](/statics/03_02.png)

---

#### 3. 관리자 페이지에 모델 등록
##### forum/admin.py에 아래의 내용을 작성한다.
```python
from django.contrib import admin
from forum.models import *

# Register your models here.
admin.site.register(Post)
```
![스크린샷](/statics/03_03.png)
forum 앱 내의 admin.py 에서 PostAdmin 클래스를 선언했기 때문에 FORUM 어플리케이션에 대한 테이블과 Post 모델이 보인다. 
이후 Member 기능 추가시 Member 앱의 admin.py 를 수정하면 MEMBER 테이블이 보여질 것이다.

우리는 Post 모델을 등록했지만 Django에서 자동으로 Post's'복수형으로 표시한다.

![스크린샷](/statics/03_04.png)
![스크린샷](/statics/03_05.png)
이전에 'views.py 맛보기'에서 테스트로 삽입했던 게시물 데이터의 조회와 수정이 가능한 것을 볼 수 있다.

![스크린샷](/statics/03_06.png)
물론 새로운 post 데이터 추가도 가능하다.

---

#### 4. admin.py의 유용한 기능들
admin.py에서 사용할 수 있는 기능들 중 필자가 유용하다고 생각하는 기능 몇가지를 소개합니다.

#### search_fields
```python
from django.contrib import admin
from forum.models import *

# Register your models here.
class PostAdmin(admin.ModelAdmin):
    # Post 모델의 title 속성에 대하여 검색기능 생성
    search_fields = ['title']

# admin.site에 등록할 때 Post객채와 같이 위에서 선언한 클래스도 같이 인자로 전달한다.
admin.site.register(Post, PostAdmin)
```
결과:
![스크린샷](/statics/03_07.png)

#### list_display
```python
class PostAdmin(admin.ModelAdmin):
    ...
    # Posts list를 보여줄 때 열 속성
    list_display = ['title', 'content', 'created_date']
```
결과:
![스크린샷](/statics/03_08.png)

#### list_display_links
```python
class PostAdmin(admin.ModelAdmin):
    ...
    # Posts list에서 어떠한 속성에 수정기능 하이퍼 링크를 추가할 것인지
    list_display_links = ['content']
    # 기본 값은 title에 하이퍼링크가 추가되어 있지만 제거
    # list_display_links = None
```
결과:
![스크린샷](/statics/03_09.png)

#### list_editable
```python
class PostAdmin(admin.ModelAdmin):
    ...
    # Posts list에서 상세(수정) 페이지에 접근하지 않고도 수정할 수 있는 필드 지정
    list_editable = ['title', 'content']

    # 만약 list_display_links 와 중복되는 속성이 있다면 오류를 반환 함으로 주의
    list_display_links = ['content']
    list_editable = ['title', 'content'] (X)
```
결과:
![스크린샷](/statics/03_10.png)

#### list_per_page
```python
class PostAdmin(admin.ModelAdmin):
    ...
    # Posts list 페이징 최대 값, 기본 값은 100
    list_per_page = 5
```
결과:
![스크린샷](/statics/03_11.png)

#### ordering
```python
class PostAdmin(admin.ModelAdmin):
    ...
    # list의 정렬 방법, '-'는 Desc
    ordering = ['-created_date']
```
결과:
![스크린샷](/statics/03_12.png)

#### fields, exclude
```python
class PostAdmin(admin.ModelAdmin):
    ...
    # Post 'add', 'change'에서 접근할 수 있는 정보 설정 

    # (admin이라 하더라도 생성날짜는 수정하지 못하도록 설정)
    # fields = ['title', 'content']
    # OR
    exclude = ['created_date']
```
결과:
![스크린샷](/statics/03_13.png)

#### fields 중첩
```python
class PostAdmin(admin.ModelAdmin):
    ...
    # 한 라인에 여러 필드를 표시한다.
    fields = [('title', 'content'), 'created_date']
```
결과:
![스크린샷](/statics/03_14.png)

#### fieldsets
```python
class PostAdmin(admin.ModelAdmin):
    ...
    # fields = ['title', 'content']
    # OR
    # exclude = ['created_date']

    # Post 의 'add' 와 'change' 에서 collapse 를 사용 하여 Advanced option 강조
    # fields, exclude 와 중복시 충돌이 날 수 있기에 주의하여 사용 (ex. exclude로 제외한 속성을 사용한다.)
    fieldsets = [
        (None, {"fields": ["title", "content"],},),
        ("Advanced options", {
            "classes": ["collapse"],
            "fields": ["created_date"],},),
    ]
```
결과:
![스크린샷](/statics/03_15.png)
![스크린샷](/statics/03_16.png)