## 23. 비동기 로그인 & 게시물 임시저장

### @login_required의 next값 문제
우리는 이전 15차시에서 @login_required 데코레이터를 사용하여 두가지의 문제점이 생겼습니다.

1. 수정 페이지에서 제출을 클릭하고 로그인을 하면 기존에 수정했던 내용이 사라져 있는 것을 확인하실 수 있습니다.
물론 뒤로가기를 두번 클릭하면 수정했던 내용과 함께 렌더되었던 페이지가 보이지만 CSRF토큰이 만료되어 제출할 수 없습니다.
(일단 제출하여 CSRF 오류를 보고 뒤로가기를 다시해 방금 로그인 하며 받은 CSRF토큰으로 교체된 후 제출은 가능합니다..)
2. 비로그인 상태에서 수정 페이지에 접근 후 로그인을 하면 수정 페이지로 이동하게 됩니다. 
그러나 삭제 요청의 경우 POST 방식인데 next가 삭제 URL을 GET 방식으로 요청하게 되어 405에러를 반환합니다. 

---

### 1. next에 POST 요청 주소 해결하기
먼저 2번 @login_required의 next값에 POST요청의 주소가 넘어가서 발생하는 문제를 먼저 해결해 봅시다.
JS에서 비동기로 로그인을 한 뒤 이전 페이지(뒤로가기)로 리다이렉트를 할 것입니다.

#### 커스텀 로그인 뷰
Django 기본 auth_views.LoginView는 비동기 요청에 대한 JSON응답을 하지 않습니다. 
로그인을 비동기 처리하기 위해 커스텀 뷰를 작성합니다.
##### member/views.py
```python
def custom_login(request):
    if request.user.is_authenticated:
        return redirect('forum:post_list')
    if request.method == 'POST':
        form = AuthenticationForm(data=request.POST)
        user = authenticate(
            request,
            username=request.POST['username'],
            password=request.POST['password']
        )
        if user is not None:
            login(request, user)
            return JsonResponse({'status': "success"})
        else:
            context = {
                'status': 'error',
                # 넌필드 에러를 전달해 줍니다. 
                'error': form.non_field_errors()
            }
            return JsonResponse(context)
    return render(request, 'registration/login.html')
```

#### registration/login.html
```html
{% extends 'base.html' %}
{% block title %}
Login
{% endblock title %}

{% block body %}
<!-- JS에서 해당 DOM 엘리먼트에 접근하기 위해 ID값을 부여해 줍니다. -->
<div id="error-div" class="container">

</div>
<div ...>
    <form id="login-form" method="POST">
        ...
    </form>
</div>

<script type="text/javascript">
const loginForm = document.querySelector("#login-form");
loginForm.addEventListener("submit", async function (event) {
    event.preventDefault();
    // 이전처럼 body.append가 아닌 <form> DOM 엘리먼트를 인자로 넣어 바로 생성할 수 있습니다.
    const formData = new FormData(loginForm);
    try {
        // 이전 넌필드 에러 메세지 삭제
        const $errorDiv = document.querySelector("#error-div");
        $errorDiv.innerHTML = "";

        const options = {
            method: "POST",
            body: formData,
        }
        const response = await fetch(document.location.pathname, options)
        //console.log(response);
        if (!response.ok) throw new Error(response.statusText);
        const data = await response.json();
        console.log(data);

        if (data.status === "success") {
            if (document.referrer) {
                // location.href 대신 replace()를 사용하여 브라우저의 히스토리에 로그인 페이지를 남기지 않게 합니다.
                location.replace(document.referrer);
            } else {
                // url 입력, 즐겨찾기, 새탭 열기 같은 '이전 페이지'가 없는 경우
                location.replace('/');
            }
        }
        else {
            const div = document.createElement("div");
            div.classList.add("alert", "alert-danger");
            const strong = div.appendChild(document.createElement("strong"));
            strong.textContent = data.error;

            $errorDiv.append(div);
        }
    } catch (error) {
        console.log(error);
    }
})
</script>
{% endblock body %}
```

---

### 2. 게시물 임시 저장하기

위의 자바스크립트에서 location.replace() 대신 history.back()을 사용하면 로그인 후 브라우저에서 캐싱한 기존 \<form>과 사용자가 입력했던 데이터가 살아 있는것을 확인할 수 있습니다. 하지만 제출을 해보면 CSRF토큰 검증 에러가 응답되는데요.
사용자가 로그아웃을 하면 기존에 발급되었던 토큰이 만료되기 때문입니다. 이 상황에서 한번 더 뒤로가기를 하면 기존의 \<form>에 새로운 토큰값이 발급되어 탑재되는 상황이 발생하긴 하지만 정상적인 방법은 아닙니다.
그래서 우리는 브라우저의 localStorage에 폼의 내용을 몇초마다 저장하는 방식을 구현할 것입니다.

#### post_form.html
```html
{% extends 'base.html' %}
{% block title %}
Forum: 게시물 작성
{% endblock title %}

{% block body %}
<div class="container">
    ...
</div>
<script type="text/javascript">
// form = PostForm(instance=post)로 인스턴스를 전달 했을 경우 pk값 접근
const postId = "{{ form.instance.pk }}";
const saveDraft = () => {
    /**
     * 브라우저의 localStorage에 입력 내용을 저장하는 함수
     * 화살표 함수로 선언한 것에 큰 의의는 없습니다. function으로 선언 하여도 됩니다.
     * 해당하는 게시물에서만 내용을 불러오기 위해 key에 postId값을 작성했습니다.
     */
    localStorage.setItem(`post_draft_${postId}`, document.querySelector('#content').value);
};
document.querySelector('#content').addEventListener('input', () => {
    /**
     * 입력이 들어오면 이전에 걸어 두었던 타임아웃을 해제하고 새로운 타임아웃을 작동하는 함수
     * setTimeout은 특정 시간이 지난 후 지정한 함수를 실행하고 예약한 타이머의 식별자를 반환합니다.
     * 그 식별자를 통해 clearTimeout 메서드에 인자로 담아 해당 타이머를 취소할 수 있습니다.
     * 다만 일반적으로 const timeout = setTimeout()처럼 지역변수로 선언하면 해당 블록에서만 사용 가능함으로
     * 다시 이벤트 리스너에 진입했을때도 사용할 수 있도록 브라우저 전역 객체(window)에 할당하여 전역변수로 사용합니다.
     */
    clearTimeout(window.timeout);
    // <textarea>에 input이 발생하고 2초동안 아무 동작이 없으면(clearTimeout이 동작하지 않음):
    // 위에서 선언한 saveDraft함수를 실행합니다.
    window.timeout = setTimeout(saveDraft, 2000);
});
window.addEventListener('DOMContentLoaded', () => {
    /**
     * 페이지가 새로 열리거나 새로고침 되면 발생 (DOMContentLoaded)
     * localStorage에 저장된 임시 저장값(draft)를 불러옵니다.
     * 만약 데이터가 존재하고 사용자가 confirm에 yes를 선택하면 해당 데이터를 넣어 주고
     * No를 선택했다면 localStorage에서 해당 임시 저장 값을 삭제합니다.
     */
    const draft = localStorage.getItem(`post_draft_${postId}`);
    if (draft) {
        if (confirm("임시저장 된 데이터가 존재합니다. 불러올까요?")) {
            document.querySelector("#content").value = draft;
        } else {
            // No를 선택했다면 localStorage에서 해당 임시 데이터를 삭제합니다.
            localStorage.removeItem(`post_draft_${postId}`);
        }
    }
})
</script>
{% endblock body %}
```
이제 해당 게시물을 수정하고 제출하지 않은 채 나갔다가 다시 수정 폼에 접근해 봅시다.
#### 결과 확인
![스크린샷](/statics/23/23_01.png)
![스크린샷](/statics/23/23_02.png)
브라우저 개발자도구의 어플리케이션 탭에서 LocalStorage탭에서 저장된 데이터를 확인하실 수 있습니다.

#### LocalStorage 와 SessionStorage
어플리케이션 탭을 확인해보면 LocalStorage외에도 SessionStorage라는 것을 확인할 수 있는데요. 둘의 차이점을 알아봅시다.
- 로컬 스토리지
    - 브라우저/컴퓨터를 꺼도 데이터가 남아있습니다.
    - 일반적으로 5~10MB의 저장 용량을 가집니다.
    - 같은 도메인 전체에서 공유합니다. (탭, 새 창에 관게되지 않습니다.)
    - 오랫동안 저장되어 있어야 하는 데이터에 사용됩니다.
- 세션 스토리지
    - 탭(세션)을 닫으면 내용들이 모두 삭제됩니다.
    - 로컬스토리지와 비슷한 저장 용량을 가집니다.
    - 같은 탭 && 같은 도메인 내에서만 사용됩니다.
    - 가입 폼 일시저장, 탭 단위의 데이터에 사용됩니다.

데이터를 오래 남겨두고 싶다면 로컬 스토리지를 사용하고, 한 세션 내에서 필요하면 세션 스토리지를 사용합니다.

#### 로컬 스토리지의 만료
로컬 스토리지는 만료 시간이 없습니다. 브라우저와 컴퓨터가 꺼지더라도 직접 삭제하지 않으면 계속 저장되어 있습니다.
사용자가 직접 개발자도구 내에서 삭제하거나 캐시 비우기, 개발자(프론트엔드)가 직접 주기적, 조건부로 삭제하는 로직을 작성하지 않으면 반영구적으로 저장되어 있습니다. 이제 그 로직을 작성해 봅시다.

##### post_form.html
```html
{% extends 'base.html' %}
{% block title %}
Forum: 게시물 작성
{% endblock title %}

{% block body %}
<div class="container">
    <form id="post-form" method="POST" class="my-5">
        ...
    </form>
</div>
<script type="text/javascript">
const postId = "{{ form.instance.pk }}";

const saveDraft = () => {
    const expDate = new Date();
    // Date 객체를 생성하고 현재 날짜에 3일을 더한 시간을 expire date로 정합니다.
    //expDate.setDate(expDate.getDate() + 3);

    // 테스트를 위해 만료 시간을 10초로 변경합니다.
    expDate.setSeconds(expDate.getSeconds() + 10);

    // 로컬 스토리지는 key:value의 형식을 가지고 있습니다.
    // value에 여러 값을 할당하기 위해 JSON을 사용합니다.
    const json = {
        'content': document.querySelector('#content').value,
        'expire': expDate.toISOString(),
    }
    // JSON의 내용을 문자열로 전달하기 위해 stringify
    localStorage.setItem(`post_draft_${postId}`, JSON.stringify(json));
};

document.querySelector('#content').addEventListener('input', () => {
    clearTimeout(window.timeout);
    window.timeout = setTimeout(saveDraft, 2000);
});

window.addEventListener('DOMContentLoaded', () => {
    // JSON 데이터이기 떄문에 .parse로 파싱해 데이터를 가져옵니다.
    const draft = JSON.parse(localStorage.getItem(`post_draft_${postId}`));
    
    if (draft) {
        if (confirm("임시저장 된 데이터가 존재합니다. 불러올까요?")) {
            document.querySelector("#content").value = draft.content;
        } else {
            // No를 선택했다면 localStorage에서 해당 임시 데이터를 삭제합니다.
            localStorage.removeItem(`post_draft_${postId}`);
        }
    }

    // localStorage의 길이만큼 반복문
    for (let i = 0; i < localStorage.length; i++) {
        const draftKey = localStorage.key(i);
        // key값이 'post_draft_'로 시작한다면:
        if (draftKey.startsWith('post_draft_')) {
            const draftValue = JSON.parse(localStorage.getItem(draftKey));
            
            const nowDate = new Date().toISOString();
            const expDate = draftValue['expire'];
            
            if (nowDate >= expDate) {
                console.log('만료! 만료!');
                localStorage.removeItem(draftKey);
            }
        }
    }
})

// 폼이 제출되면 저장했던 임시 데이터를 삭제합니다.
document.querySelector('#post-form').addEventListener('submit', function() {
    localStorage.removeItem(`post_draft_${postId}`);
})
</script>
{% endblock body %}
```

정상 작동이 확인되면 이제 static/js/에 스크립트 파일을 분리해 줍시다.