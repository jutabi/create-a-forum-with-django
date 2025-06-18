## 09. 템플릿 상속

[참고 문헌: 점프 투 장고 / 2-09 템플릿 상속](https://wikidocs.net/70851)

우리가 지금까지 만든 템플릿은 list, detail, form 3가지 입니다. 그런데 여기에는 중복되는 내용이 존재하고 앞으로 기능이 추가되면 더욱더 많아질 것입니다. 그래서 우리는 Django의 템플릿 상속 기능을 적용합니다. 

먼저 'base.html', 기본 뼈대가 되는 템플릿을 작성합니다.
forum-with-django/templates/base.html
```html
<!-- 기존의 템플릿들에서 중복되는 내용들 (독타입, 헤드 태그)을 작성합니다. -->
<!DOCTYPE html>
<html lang="ko">
<head>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.6/dist/css/bootstrap.min.css" rel="stylesheet"
          integrity="sha384-4Q6Gf2aSP4eDXB8Miphtr37CMZZQ5oXLH2yaXMJ2w8e2ZtHTl7GptT4jmndRuHDT" crossorigin="anonymous">
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>
    <!-- 커스터마이징, 상속하여 수정할 block을 지정합니다. -->
    {% block title %}
    <!-- endblock 이후 블록 명은 적어주지 않아도 사용할 수 있지만 유지보수와 시인성을 위해 작성해 줍니다. -->
    {% endblock title %}
    </title>
</head>
<body>
{% block body %}
{% endblock body %}
</body>
</html>
```

post_list.html
```html
<!-- templates 최상위 위치의 base.html을 상속받습니다. -->
{% extends 'base.html' %}
<!-- base.html에서 지정했던 title 블록 내부의 내용을 작성합니다. -->
{% block title %}
Forum
{% endblock title %}

<!-- body 블록(body 태그 내부)의 내용을 작성합니다. -->
{% block body %}
<div class="container">
    <table class="table table-striped table-hover">
        <thead>
            <tr class="table-secondary">
                <th style="width: 10%">글번호</th>
                <th style="width: 60%">제목</th>
                <th style="width: 15%">작성일</th>
            </tr>
        </thead>
        <tbody class="table-group-divider">
            {% if posts %}
            {% for post in posts %}
            <tr>
                <td>{{ post.pk }}</td>
                <td>
                    <a href="{% url 'forum:post_detail' post.pk %}">{{ post.title }}</a>
                </td>
                <td>{{ post.created_date|date:"Y/m/d A h:i" }}</td>
            </tr>
            {% endfor %}
            {% endif %}
        </tbody>
    </table>
<a href="{% url 'forum:post_create' %}" class="btn btn-primary">게시물 작성</a>
</div>
{% endblock body %}
```
기존의 중복되던 구문이 사라지고 필요한 코드만 작성되어 있어 깔끔해진 모습을 볼 수 있습니다. 

나머지 post_detail.html, post_form.html 도 같은 방법으로 템플릿 상속을 적용합니다.