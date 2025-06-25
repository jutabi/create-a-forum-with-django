## 15. 작성자가 아닌 사용자의 수정과 삭제 제한

### 뷰에서 제한하기

#### forum/views.py
```python
...
from django.http import HttpResponseForbidden

def post_update(request, pk):
    post = get_object_or_404(Post, pk=pk)

    if request.user != post.author:
        # HttpResponseForbidden (HTTP 403 Forbidden)
        # 주로 사용자가 서버에 접근할 권한이 없을 때 (접근은 가능하나 허가되지 않은 리소스에 접근)
        # 사용하는 오류 HTTP리스폰스 입니다.

        # Django의 대표적인 오류 응답 메서드
        # HTTPResponseBadRequest  400  잘못된 데이터 전송(파라미터 누락, 타입 오류)
        # HTTPResponseNotFound    404  존재하지 않는 페이지/리소스 요청 시
        # HTTPResponseNotAllowed  405  지원하지 않는 HTTP 메서드 요청 시
        # HTTPResponseGone        410  이전에는 존재했지만 리소스가 영구적으로 삭제됐을 떄
        # HTTPResponseServerError 500  서버 내부 오류 발생 시
        return HttpResponseForbidden("다른 사용자의 게시물은 수정하실 수 없습니다.")

    if request.method == 'GET':
        ...
    elif request.method == 'POST':
        ...

def post_delete(request, pk):
    post = get_object_or_404(Post, pk=pk)
    if request.user != post.author:
        return HttpResponseForbidden("다른 사용자의 게시물은 삭제하실 수 없습니다.")

    post.delete()

    return redirect('forum:post_list')
```
로그인 한 사용자가 작성하지 않은 게시물의 수정 또는 삭제 버튼을 클릭해 봅시다.
##### 결과 확인
![스크린샷](/statics/15/15_01.png)
![스크린샷](/statics/15/15_02.png)

---

### 템플릿에서 제한하기
현재 프로젝트는 사용자가 수정이나 삭제에 대한 요청을 보낼 시 서버에서 403 에러를 리스폰스를 하니 틀린 것은 아닙니다.
다만 UX의 관점에서 보았을 때 사용자에게 권한이 없는 기능(다른 사용자의 게시물 수정, 삭제)을 보여주고 실제 동작하는 버튼으로 생성했다면(결과가 어떻더라도) 이것은 잘못된 UX 디자인 입니다.

이러한 이유로 로그인 한 사용자의 게시물이 아니라면 수정과 삭제 버튼을 표시하지 않도록 합시다.

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
               class="btn btn-secondary mt-2 justify-content-around">글 목록</a>
        </div>
        <div class="col text-end">
            <!--뷰와 똑같이 request.user를 이용해 세션의 로그인된 사용자 객체를 불러와 비교합니다.-->
            <!--글 목록 버튼이 왼쪽 정렬이기 때문에 col div까지 if문 안에 포함시켜도 괜찮습니다.-->
            {% if post.author == request.user %}
            <a href="{% url 'forum:post_update' post.pk %}"
               class="btn btn-outline-success mt-2">수정</a>

            <a href="javascript:void(0)"
               class="btn btn-danger mt-2 delete-link"
               data-url="{% url 'forum:post_delete' post.pk %}">삭제</a>
            {% endif %}
        </div>
    </div>
</div>
```

##### 결과 확인
![스크린샷](/statics/15/15_03.png)
![스크린샷](/statics/15/15_04.png)