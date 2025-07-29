## 28. SMTP, django-environ

### SMTP
Simple Mail Transfer Protocol은 인터넷에서 이메일을 전송하기 위한 표준 프로토콜 입니다.
이메일을 보내는 쪽(클라이언트 or 메일 서버)과 받는 쪽(상대 메일 서버)사이에서 메세지를 전송하는 규칙을 정의하며, 외부 메일 시스템 간 교환이 가능하게 해줍니다.
#### 기본 역할
이메일을 전송하는 기능을 담당합니다. 송신에만 집중하며 수신과 저장에는 POP3/IMAP 등 추가 프로토콜이 담당합니다.
#### 동작 방식
사용자가 이메일 클라이언트에서 메일을 작성하면 메일 서버로 전달되고, 메일 서버는 SMTP를 통해 목적지 서버로 메일을 전송합니다.
최종적으로 POP3/IMAP 프로토콜을 통해 메일을 읽을 수 있습니다.

### Gmail SMTP 설정하기
여러 웹메일 서비스들이 있지만 Gmail을 이용하여 SMTP 비밀번호를 발급받아 사용해 봅시다.

![스크린샷](/statics/28/28_01.png)
![스크린샷](/statics/28/28_02.png)
![스크린샷](/statics/28/28_03.png)
![스크린샷](/statics/28/28_04.png)
![스크린샷](/statics/28/28_05.png)
![스크린샷](/statics/28/28_06.png)
위의 사항들을 따라 16자리 비밀번호를 발급받습니다.

### settings\.py 설정하기
```python
# 어떠한 방법을 사용해서 Email을 보낼 것인가: SMTP 사용하여 실제로 전송
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'

# 메일을 보낼 서버 주소: gmail 메일 서버 사용
EMAIL_HOST = 'smtp.gmail.com'

# 메일 서버와 장고의 통신 시 사용할 포트번호
    # 기본 컨벤션
    # TLS 방식: 587
    # SSL 방식: 465
    # SMTP 기본: 25 (보안에 취약하여 권장하지 않음)
EMAIL_PORT = 587

# 보안 연결 사용 여부
    # 장고(서버)에서 메일 서버로 정보를 보낼 때 암호화 하여 보내줍니다.
EMAIL_USE_TLS = True

EMAIL_HOST_USER = 'Your Gmail id@gmail.com'
EMAIL_HOST_PASSWORD = '발급받은 비밀번호'

# 전송되는 메일의 발신자
DEFAULT_FROM_EMAIL = 'ForumWithDjango <admin@FWD.com>'
```

#### 우리 프로젝트의 전달 과정
장고 프로젝트 서버 -**SMTP 사용**-> Gmail SMTP 서버 -**SMTP 사용**-> 수신사 메일 서버 -**POP3/IMAP**-> 수신자 읽음

#### 결과 확인
실제 자신의 이메일을 가지는 아이디를 생성하고 비밀번호 찾기를 실시해 봅시다.
![스크린샷](/statics/28/28_07.png)
![스크린샷](/statics/28/28_08.png)
![스크린샷](/statics/28/28_09.png)
![스크린샷](/statics/28/28_10.png)
비밀번호 변경이 정상 처리된 것을 확인하실 수 있습니다.

---

### django-environ
이제 django-environ을 이용해 SMTP 비밀번호를 별도의 파일로 분리해, 버전 관리에 포함되지 않도록 구성해 봅시다.

#### 설치
```bash
(.venv) C:\Users\***\PycharmProjects\forum-with-django>pip install django-environ 
Collecting django-environ
  Using cached django_environ-0.12.0-py2.py3-none-any.whl.metadata (12 kB)
Using cached django_environ-0.12.0-py2.py3-none-any.whl (19 kB)
Installing collected packages: django-environ
Successfully installed django-environ-0.12.0

(.venv) C:\Users\***\PycharmProjects\forum-with-django>
```

#### .env 파일 생성
프로젝트 루트 최상단(manage\.py의 위치)에 '.env'파일을 생성합니다.
![스크린샷](/statics/28/28_11.png)
.gitignore파일에 .env를 작성합니다.
```
.idea
db.sqlite3
*.pyc
__pycache__
.env
```

#### .env 작성과 활용
##### .env
```
EMAIL_HOST_USER=***@gmail.com
EMAIL_HOST_PASSWORD='**** **** **** ****'
```
env 파일을 작성할 때에는 공백 없이 작성해야 합니다.
만약 value에 공백이 필요할 경우 quotes를 감싸줍니다.

##### settings\.py
```python
from pathlib import Path

from django.template.context_processors import static

# .env파일의 패스 설정을 위한 os와 environ을 임포트합니다.
import os, environ
env = environ.Env()

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent

# 기존 settings의 BASE_DIR을 이용하여 .env파일의 패스를 설정합니다.
environ.Env.read_env(os.path.join(BASE_DIR, '.env'))

...

EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_USE_TLS = True
# 아래와 같이 env에 key를 전달해 value를 얻어올 수 있습니다.
EMAIL_HOST_USER = env('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD')
DEFAULT_FROM_EMAIL = 'ForumWithDjango <admin@FWD.com>'
```

##### 결과 확인
서버를 재가동 하고 env파일에서 정상적으로 값을 가져오는지 메일을 보내 봅시다.
![스크린샷](/statics/28/28_12.png)

#### 그 외 별도로 관리해야 하는 키들
- SECRET_KEY: 장고 보안 핵심. 유출 시 세션/암호화가 뚫릴 수 있습니다.
    - 모든 프로젝트에 존재합니다. 우리의 프로젝트는 이미 버전관리에 올라갔지만 실습을 위해 .env에 작성해 봅시다.
- 데이터베이스 정보(DB_NAME, DB_USER, DB_PASSWORD, DB_HOST, DB_PORT)
- 외부 서비스 토큰(API_KEY, ACCESS_TOKEN, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY 등)
- 기타 OAuth, Stripe, 결제 플랫폼 등 민감 정보.