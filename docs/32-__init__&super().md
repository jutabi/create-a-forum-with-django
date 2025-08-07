## 32. \_\_init__과 super()
이전 차시에서 우리는 회원탈퇴 시 사용자에게 비밀번호를 입력하게 하는 안전장치를 만들었습니다.
그때 사용했던 \_\_init__, super메서드에 대해 알아봅시다.

### 사용했던 예제 코드
```python
class DeleteReasonForm(forms.ModelForm):
    password = forms.CharField(label="비밀번호 확인", widget=forms.PasswordInput)
    class Meta:
        ...

    def __init__(self, user, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.user = user

    def clean(self):
        cleaned_data = super().clean()
        password = cleaned_data.get('password')
        if password and not check_password(password, self.user.password):
            self.add_error('password', '비밀번호가 일치하지 않습니다.')
```

### 1. \_\_init__()메서드
- 인스턴스 초기화(생성자) 메서드
    - 객체가 만들어 질 때(인스턴스) 자동으로 호출됩니다.
- 위의 예제에서는 폼을 만들때 해당하는 user객체를 받아 self.user에 저장해 두는 용도로 활용합니다.

#### 매개변수
- self: 파이썬의 클래스에서 꼭 들어가는 자신(인스턴스) 참조
- user: 사용하는 뷰에서 request.user를 전달
- *args: 추가 위치 인자들(튜플 형태)
- **kwargs: 추가 키워드 인자들(딕셔너리 형태)

super()를 사용하여, 쉽게 설명하면 부모 클래스(ModelForm)에게 user를 제외한 전달인자들을 넘겨줍니다.
(폼을 사용하기 위해 필요한 기본 인자들(부모 클래스가 필요한)을 전달해 줍니다.) (아래에서 자세히 설명합니다.)

---

### 2. clean()메서드
- 장고 폼의 유효성 검사 로직을 커스텀할 때 사용하는 메서드 입니다.
    - 뷰에서 is_valid()메서드를 통해 유효성 검사를 진행할 때 정의된 clean()메서드가 실행됩니다.
- 위의 예제에서는 폼 데이터 내의 입력받은 비밀번호가 사용자의 비밀번호와 일치하는 지를 판단하는 역할을 수행합니다.

부모 클래스(super() = ModelForm)의 clean()메서드로 부터 폼 입력값 데이터를 딕셔너리 형태로 받아옵니다.
auth.hashers의 check_password메서드를 사용하여 기존의 비밀번호와 입력된 비밀번호의 값을 비교합니다.

---

### 3. *args, **kwargs 매개변수
#### args (가변 위치 인자)
- 여러개의 추가 위치 인자를 순서대로 튜플에 저장해 전달받습니다.
- 메서드에서 여러개의 '이름이 없는' 인자를 받을 때 사용합니다.
```python
def argsFunc(*args):
    print(args)

argsFunc(2, 1, 4)

=> (2, 1, 4)
```

#### kwargs (가변 키워드 인자)
- 여러개의 추가 키워드 인자를 딕셔너리 형태로 전달받습니다.
- 메서드에서 여러개의 'key=value'형태의 인자를 받을 때 사용합니다.
```python
def kwargsFunc(**kwargs):
    print(kwargs)

kwargs(a=2, b=1, c=4)

=> { 'a': 2, 'b': 1, 'c': 4 }
```

두 방식 모두 메서드에 넘길 파라미터의 개수가 정해지지 않았거나, 다양한 인자를 받아야 할 때 사용합니다.
특히 프레임워크 코드에서 인자를 전부 받아, 상속받는 부모 클래스에 그대로 전달하는 용도로 사용됩니다.

두 매개변수를 동시에 사용할 때는 항상 (*args, **kwargs)의 순서로 써야 합니다.

#### example
```python
def func(a, b, *args, **kwargs):
    print(a, b)
    print(args)
    print(kwargs)

func(2, 1, 4, 3, e=5, f=6)

=> 2 1
=> (4, 3)
=> { 'e': 5, 'f': 6}
```

#### 클래스 내에서 활용
생성자(\_\_init__) or 메서드에서 args, kwargs를 받으면 인자의 개수나 종류에 상관없이 유연하게 처리할 수 있습니다.
자식 클래스에서 **부모 클래스의 기능/로직을 이용하기 위해**, 부모 클래스가 정의한 인자를 대신 받아 그대로 super()를 통해 전달해 줍니다.
```python
class Parent:
    def __init__(self, a, b, **kwargs):
        print(kwargs)
        ...

# 자식 클래스가 Parent를 상속받는다면, 자식 클래스에서도 __init__에 같은 인자를 받을 수 있는데,
# 이때 부모 클래스가 정의한(필요한) 인자를 부모 클래스에 넘겨주어 부모 클래스의 __init__이 정상 작동할 수 있도록 합니다.
class Child(Parent):
    # 커스텀 인자를 앞에 적고, 순서에 유의하여 *args와 **kwargs를 인자로 받습니다.
    def __init__(self, user, *args, **kwargs):
        self.user = user
        super().__init__(*args, **kwargs)
```
```python
#forms.py
class DeleteReasonForm(forms.ModelForm):
    def __init__(self, user, *args, **kwargs)

#views.py
def function(request):
    ...
    form = DeleteReasonForm(request.user, request.POST)

# 위의 예제에서는 첫 번쨰 인자(request.user)는 자식클래스 생성자의 user매개변수 값으로 전달됩니다.
# 그 이후에 전달되는 인자(들)은(request.POST)부모 클래스(ModelForm)가 필요로 하는 데이터입니다.
# 그 데이터들은 부모 클래스의 기능을 사용하기 위해 *args 튜플에 담아 부모 클래스에게 그대로 전달해 줍니다.

# 부모클래스에 정의되어 있는 인자들 예시: request.POST, request.FILES, initial ...
```

---

### 4. super()메서드
super()메소드는 부모 클래스를 가리키는, 부모 클래스의 메서드나 생성자를 사용(호출)할 수 있게 해주는 메서드입니다.
다만 틀린 부분이 존재하는데요. 바로 다중 상속일 떄 입니다.

#### MRO(Method Resolution Order)
super()는 단순히 직계 부모 클래스만을 가리키지 않습니다. 'MRO'라는 파이선의 상속 탐색 순서를 따라 바로 다음 탐색 순서에 해당하는 클래스를 참조하게 됩니다.
```python
class A:
    def func(self):
        print("A Class")

class B(A):
    def func(self):
        print("B Class")
        super().func()

class C(A):
    def func(self):
        print("C Class")
        super().func()

class D(B, C):
    def func(self):
        print("D Class")
        super().func()

d = D()
d.func()

=> D Class
=> B Class
=> C Class
=> A Class

# Python의 MRO를 설명하는 예제입니다.
# 클래스를 상속받을 때 super()를 사용하게 되면, MRO 순서에 따라 부모 클래스를 탐색하게 됩니다.
# 이때 깊은 상속관계에서 여러 조상 클래스들이 같은 메서드(func())를 가지고 있다면 이 MRO순서에 따라 중복이 없이 클래스를 탐색하고,
# 해당 메서드들을 실행하게 됩니다.

print(D.mro())
=> [D, B, C, A, object]
# 위 mro의 결과에 따라 D에서 super()를 사용하면 B클래스, C클래스를 참조하게 되어 결국 부모 클래스를 참조하게 되는 것입니다.
```

#### super()를 사용하는 이유
- 상위 클래스의 명확한 이름을 작성하지 않아도 됩니다.
    - MRO상 호출한 클래스의 바로 다음 순서는 부모 클래스를 가리키고 있습니다.
    - 이를 이용하여 부모 클래스명을 직접 작성하지 않아 유지보수에 이점, 상속 구조의 확장/변경 시 코드 변경 최소화
- 생성자/메서드 중복 호출 방지
    - 위의 예제에서도 볼 수 있듯이 D는 B, C클래스를 상속받고, B와 C는 A클래스를 상속받습니다.
    - 이때 여러 부모 클래스에 같은 메서드가 존재한다면 A클래스는 B에서 올라갈 때 한번, C에서 올라갈 때 한번, 총 두번의 중복 실행이 될 우려가 있습니다.
        - 만약 B, C클래스에서 super()가 아닌 A.func()으로 호출했다면 A클래스의 func()이 두번 실행되게 됩니다.
    - 이런 상황을 방지하기 위하여 super()메서드는 다중 상속 시 트리 탐색 순서를 C3 선형화 알고리즘을 사용하여 중복없이 탐색할 수 있도록 도와줍니다.

- 결론적으로 'super()는 단순히 직계 상위 클래스를 호출'하는 메서드가 아니라, 상속 구조에서 메서드 탐색 순서(MRO)에 따라 중복 없이 상위 클래스들을 탐색할 수 있도록 도와주는 파이썬 객체 모델의 기능입니다.

#### 조금 더 자세히
- MRO는 **현재 클래스를 기준**으로 해당 클래스와 그 모든 부모(조상)들 까지 탐색하는 상속구조를 파악하여 생성됩니다.
    - 즉, 내가 상속받은 클래스의 체인 전체를 계산하여 중복이 없는 하나의 리스트로 정렬합니다.
- MRO는 클래스가 정의될 때 생성되며, 클래스명.mro() 또는  클래스명.__mro__로 조회하실 수 있습니다.

