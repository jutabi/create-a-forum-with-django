## 31. 회원 탈퇴
### 서버 측 작업
회원탈퇴 뷰는 구현이 간단하기 때문에 auth_view에 작성되어 있지 않습니다. 직접 구현해 봅시다.

#### member/models.py
먼저 회원 탈퇴 시 이유를 묻고 그 데이터를 저장하는 모델을 생성해 봅시다.
```python
from django.contrib.auth.models import AbstractUser
from django.db import models


class ForumUser(AbstractUser):
    nickname = models.CharField(max_length=10)


# 아래처럼 각 필드를 생성하거나 하나의 charField에 ','join 연산을 통해 통째로 집어 넣어도 됩니다.
# 다만 DB 쿼리 요청 만으로 간편하게 데이터를 확인할 수 있도록 각자의 필드를 생성하였습니다.
class DeleteReason(models.Model):
    not_use = models.BooleanField(default=False)
    lack_of_features = models.BooleanField(default=False)
    poor_management = models.BooleanField(default=False)
    other_detail = models.CharField(max_length=100, blank=True, null=True)
```
마이그레이션을 진행해 줍니다.
```
(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py makemigrations
Migrations for 'member':
  member\migrations\0002_deletereason.py
    + Create model DeleteReason

(.venv) C:\Users\***\PycharmProjects\forum-with-django>python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, forum, member, sessions
Running migrations:
  Applying member.0002_deletereason... OK
```

#### member/forms.py
뷰와 템플릿에서 모델을 편하게 사용하기 위해 장고 폼을 생성합니다.
```python
from django import forms
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth.hashers import check_password

from member.models import ForumUser, DeleteReason


class RegisterForm(UserCreationForm):
    ...


class DeleteReasonForm(forms.ModelForm):
    password = forms.CharField(label="비밀번호 확인", widget=forms.PasswordInput)
    class Meta:
        model = DeleteReason
        fields = ['not_use', 'lack_of_features', 'poor_management', 'other_detail']
        labels = {
            'not_use': '더이상 사용하지 않음',
            'lack_of_features': '기능의 부재',
            'poor_management': '운영상 미숙',
            'other_detail': '기타(입력해 주세요.)'
        }

    # 비밀번호 확인을 위해 __init__과 clean메서드를 생성합니다.
    # 만약 비밀번호 확인 기능이 없다면 추가하지 않아도 됩니다.
    # __init__과 super()에 대한 설명은 다음차시에서 진행합니다.
    def __init__(self, user, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.user = user

    def clean(self):
        cleaned_data = super().clean()
        password = cleaned_data.get('password')
        if password and not check_password(password, self.user.password):
            self.add_error('password', '비밀번호가 일치하지 않습니다.')
```

#### member/urls.py
```python
urlpatterns = [
    ...
    # 탈퇴 후 사용자에게 확인 메세지를 보여주기 위해 /done/ 엔드포인트를 작성하였습니다.
    path('delete/',                views.delete,                    name='delete'),
    path('delete/done/',           views.delete_done,               name='delete_done'),
    ...
```

#### member/views.py
```python
@login_required
def delete(request):
    if request.method == 'GET':
        form = DeleteReasonForm(request.user)
        return render(request, 'registration/membership_delete_form.html', {'form': form})
    elif request.method == 'POST':
        # 매개변수 순서에 주의
        form = DeleteReasonForm(request.user, request.POST)
        if form.is_valid():
            form.save()
            request.user.delete()
            logout(request)
            return redirect('member:delete_done')
        else:
            return render(request, 'registration/membership_delete_form.html', {'form': form})


def delete_done(request):
    return render(request, 'registration/membership_delete_done.html')
```

---

### 클라이언트 측 작업

#### profile.html
로그인한 사용자의 프로필 페이지라면 회원탈퇴 버튼을 출력합니다.
```html
{% extends 'base.html' %}
{% block title %}
{{ profile_user.nickname }}'s Profile
{% endblock title %}

{% block body %}
<div class="container mt-3">
    <div class="card">
        ...
        {% if request.user == profile_user %}
        <div class="card-footer">
            <a href="{% url 'member:delete' %}" class="btn btn-danger float-right" type="button">회원 탈퇴</a>
        </div>
        {% endif %}
    </div>
</div>
{% endblock body %}
```

#### membership_delete_form.html
```html
{% extends 'base.html' %}
{% block title %}
Delete Membership
{% endblock %}
{% block style %}
{% endblock %}

{% block body %}
    <div class="container">
        <form method="POST">
            {% csrf_token %}
            <h2>회원 탈퇴</h2>
            <!-- 템플릿에서 장고 폼을 사용하여 p태그로 감싸 checkbox 생성 -->
            {{ form.as_p }}
            <button type="submit" class="btn btn-danger">Delete</button>
        </form>
    </div>
{% endblock %}
```

#### membership_delete_done.html
```html
{% extends 'base.html' %}
{% block title %}
Delete Membership Done
{% endblock %}

{% block body %}
    <div class="container">
        <h4>회원 탈퇴 완료</h4>
        <p>이용해 주셔서 감사합니다.</p>
    </div>
{% endblock %}
```

---

### 결과 확인
![스크린샷](/statics/31/31_01.png)
![스크린샷](/statics/31/31_02.png)
![스크린샷](/statics/31/31_03.png)
![스크린샷](/statics/31/31_04.png)
![스크린샷](/statics/31/31_05.png)