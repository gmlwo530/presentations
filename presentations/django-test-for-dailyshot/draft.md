# 장고 테스트 삽질기

## Mock

mock을 왜 쓰나요
- 특정 메소드 실제로 실행 안시키기
- 메소드 실행 여부 파악
- DB 동작 없이 모델 객체 생성 

Mock 기본 사용법

```python
class ProductionClass:
    def method(self):
        self.something(1, 2, 3)

    def something(self, a, b, c):
        pass

real = ProductionClass()
real.something = MagicMock()
real.method()
real.something.assert_called_once_with(1, 2, 3)
```
출처: https://docs.python.org/ko/3.8/library/unittest.mock-examples.html

```python

```

Mock(spec=Something)

Mock은 spec 키워드 인자를 사용하여, 모의 객체를 위한 사양으로 객체를 제공할 수 있도록 합니다. 사양 객체에 존재하지 않는 모의 객체의 메서드/어트리뷰트에 액세스하면 어트리뷰트 에러가 즉시 발생합니다. 사양의 구현을 변경하면, 해당 클래스를 사용하는 테스트는 해당 테스트에서 클래스를 인스턴스 화하지 않고도 즉시 실패하기 시작합니다.

>>>
mock = Mock(spec=SomeClass)
mock.old_method()
Traceback (most recent call last):
   ...
AttributeError: object has no attribute 'old_method'

- https://docs.python.org/ko/3.8/library/unittest.mock-examples.html#creating-a-mock-from-an-existing-object

### patch

모듈과 클래스는 사실상 전역이라서, 테스트 후에 패치를 실행 취소해야 합니다, 그렇지 않으면 패치가 다른 테스트로 지속하여, 진단하기 어려운 문제를 일으킵니다.

### patch의 사용 방법 2가지 - decorator와 context manager

@patch("some_module.foo_func")
def test_foo(self, mock_foo_func):
    ...

with patch("some_module.foo_func") as mock_foo_func:
    ...

### context manager patch 여러 개 사용하기
`*_`

### mocking 할 모듈 위치 잡기

payments.core.execute_approval # bad - mocking 안 됨

payments.pay.execute_approva # 사용하는 곳의 경로로 모킹 해야 됨

### model 클래스 모킹하기 

with patch("some.models.Payment") as mock:
    mock.objects.get.side_effect = Payment.DoesNotExists() # Payment.DoesNotExists는 mock 땜에 exceptino이 아님

with patch("some.models.Payment.objects") as mock:
    mock.get.side_effect = Payment.DoesNotExists() # Payment.DoesNotExists는 mock 땜에 exceptino이 아님

## setUpTestData와 setUp의 차이 @

- setUpTestData : TestCase가 실행 될 때 한 번 실행 됨
- setUp : test 함수 별로 실행 됨

예시 들기

## 상속 받는 TEST CASE @

- 이미지 첨부

https://docs.djangoproject.com/en/4.0/topics/testing/tools/#provided-test-case-classes

### SimpleTestCase

- 데이터베이스 쿼리 사용을 막음
- 한 함수의 플로우와 예외를 테스트 할 때 유용하게 씀
- 당연히 데이터베이스 쿼리를 허용하지 않아서 빠른 테스트가 가능
- Mock을 이용하면 모델을 사용하는 함수 또는 클래스도 테스트 가능

### TransactionTestCase

- 각 테스트가 끝날 때 마다 모든 테이블을 truncate(테이블은 두고 테이블이 가지고 있는 데이터 날림)함으로써 각 테스트 별로 데이터의 겹침이 없이 테스트 가능
- 실제로 데이터베이스에서 commit, rollback 함으로써 데이터베이스와 어떻게 상호작용하는지 알 수 있음 


```python
class ProductTransactionTestCase(TransactionTestCase):
    def test_1(self):
        ...  # TRUNCATE TABLE Product; (그 외 사용 된 테이블 개수만큼 더 실행)
    
    def test_2(self):
        ...  # TRUNCATE TABLE Product; (그 외 사용 된 테이블 개수만큼 더 실행)
```

### TestCase

- Django에서 흔히 쓰이는 클래스
- 테스트들을 atomic 블락으로 감싸게 됨
- 각 테스트가 끝날 때마다 atomic으로 감싼 transaction이 롤백 됨
- 그러므로 특정 database transaction 테스트를 하는것에 적정되지 않음
- setUpTestData는 클래스 레벨에서 atomic 블락으로 감싸게 됨

```python
class ProductTestCase(TestCase): # BEGIN TRANSACTION;
    def test_1(self): # SAVEPOINT S1;
        ...           # ROLLBACK TO SAVEPOINT S1;
    
    def test_2(self): # SAVEPOINT S2;
        ...           # ROLLBACK TO SAVEPOINT S2;

    # ROLLBACK;
```

### BaseTestCase - for dailyshot

https://speakerdeck.com/youngminkoo/pycon-kr-2019-teseuteue-geolrineun-siganeul-star-92-percent-star-juligi


[ppt 테스트 시간 비교 이미지 첨부]


-> (과장해서) TransactionTestCase 사용하지 마라

하지만, select_for_update, on_commit 등 특정 transaction 동작에 따른 테스트를 해야 될 때 있음. 이런 동작을 테스트 하기 위해서는 TransactionTestCase를 사용하라고 Django 문서에 나와있음.

https://medium.com/@juan.madurga/speed-up-django-transaction-hooks-tests-6de4a558ef96

-> 이거 잘못 만듬 안 써도 됨. on_commit에 대한 해결책이라서 깊게 공부하지 않고 만듬

대신 testcase와 transcationtestcase는 잘 구분해서 쓰길

## Serializer 테스트, context 와 image field 다루기 @

request가 들어갔을 때 request mock을 만들어주는 이유

```python
mock_request = Mock()
mock_request.user = receiver_user
mock_request.build_absolute_uri.return_value = None => image field 용

self.assertDictEqual(
    dict(
        id=payment.id,
        buyer_name=self.test_user.profile.get_realname_in_review()[:-1],
        receiver_name=None,
        created_at=datetime_to_drf_datetime(payment.created_at),
        orders=order_dict_list,
    ),
    PaymentListSerializerVersion1(payment, context={"request": mock_request}).data,
)
```


## 동시성 테스트 @

https://github.com/dailyshot/earlybird/blob/master/payments/tests/test_base.py


## 그 외 잡기술@ 
 
### 더미 데이터 만들 때 ImageField 용 데이터 만들기

```python
import tempfile

event = Event.objects.create(
    name="이벤트",
    image=tempfile.NamedTemporaryFile(suffix=".jpg").name,
)
```

### 환경 변수에 따른 테스트

https://docs.djangoproject.com/en/4.0/topics/testing/tools/#overriding-settings

```python
with self.settings(SERVER_ENV="production"):
    ...

@override_settings(SERVER_ENV="test", TEST_SERVER_DOMAIN="test.domain")
    def test_base_pay_library_test_env(self):
        ...
```

