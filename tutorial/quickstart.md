# 빠른 시작

관리자가 시스템의 사용자 및 그룹을 조회하고 편집할 수 있는 간단한 API를 만들겠습니다.

## 프로젝트 설정

`tutorial`이라는 새 Django 프로젝트를 만들고, `quickstart`라는 새 앱을 시작합니다.

    # 프로젝트 디렉토리 생성
    mkdir tutorial
    cd tutorial

    # 패키지 종속성을 로컬에서 격리하기 위한 가상 환경 생성
    python3 -m venv env
    source env/bin/activate  # Windows에서는 `env\Scripts\activate` 사용

    # 가상 환경에 Django와 Django REST framework 설치
    pip install django
    pip install djangorestframework

    # 단일 애플리케이션으로 새 프로젝트 설정
    django-admin startproject tutorial .  # 마지막의 '.' 문자에 주의
    cd tutorial
    django-admin startapp quickstart
    cd ..

프로젝트 레이아웃은 다음과 같이 되어야 합니다:

    $ pwd
    <some path>/tutorial
    $ find .
    .
    ./tutorial
    ./tutorial/asgi.py
    ./tutorial/__init__.py
    ./tutorial/quickstart
    ./tutorial/quickstart/migrations
    ./tutorial/quickstart/migrations/__init__.py
    ./tutorial/quickstart/models.py
    ./tutorial/quickstart/__init__.py
    ./tutorial/quickstart/apps.py
    ./tutorial/quickstart/admin.py
    ./tutorial/quickstart/tests.py
    ./tutorial/quickstart/views.py
    ./tutorial/settings.py
    ./tutorial/urls.py
    ./tutorial/wsgi.py
    ./env
    ./env/...
    ./manage.py

애플리케이션이 프로젝트 디렉토리 내에 생성된 것이 다소 특이할 수 있습니다. 프로젝트의 네임스페이스를 사용하면 외부 모듈과의 이름 충돌을 방지할 수 있습니다(빠른 시작의 범위를 벗어나는 주제입니다).

이제 데이터베이스를 처음으로 동기화합니다:

    python manage.py migrate

이후 예제에서 인증할 `admin`이라는 초기 사용자와 비밀번호를 생성합니다.

    python manage.py createsuperuser --username admin --email admin@example.com

데이터베이스 설정이 완료되고 초기 사용자가 생성되면, 앱 디렉토리를 열고 코딩을 시작합니다...

## 직렬화기

먼저 몇 가지 직렬화기를 정의합니다. 데이터 표현을 위해 사용할 `tutorial/quickstart/serializers.py`라는 새 모듈을 만듭니다.

```python
from django.contrib.auth.models import Group, User
from rest_framework import serializers


class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ['url', 'username', 'email', 'groups']


class GroupSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Group
        fields = ['url', 'name']
```

이 경우 `HyperlinkedModelSerializer`를 사용하여 하이퍼링크 관계를 사용하고 있음을 주목하십시오. 기본 키 및 기타 여러 관계를 사용할 수도 있지만, 하이퍼링크는 RESTful 디자인에 적합합니다.

## 뷰

이제 뷰를 작성하겠습니다. `tutorial/quickstart/views.py`를 열고 다음을 작성합니다.

```python
from django.contrib.auth.models import Group, User
from rest_framework import permissions, viewsets

from tutorial.quickstart.serializers import GroupSerializer, UserSerializer


class UserViewSet(viewsets.ModelViewSet):
    """
    사용자를 조회하거나 편집할 수 있는 API 엔드포인트입니다.
    """
    queryset = User.objects.all().order_by('-date_joined')
    serializer_class = UserSerializer
    permission_classes = [permissions.IsAuthenticated]


class GroupViewSet(viewsets.ModelViewSet):
    """
    그룹을 조회하거나 편집할 수 있는 API 엔드포인트입니다.
    """
    queryset = Group.objects.all().order_by('name')
    serializer_class = GroupSerializer
    permission_classes = [permissions.IsAuthenticated]
```

여러 뷰를 작성하는 대신, 공통 동작을 `ViewSets`라는 클래스에 그룹화하고 있습니다.

필요한 경우 이를 개별 뷰로 분할할 수 있지만, 뷰셋을 사용하면 뷰 로직을 깔끔하게 구성할 수 있으며 매우 간결합니다.

## URL

이제 API URL을 연결하겠습니다. `tutorial/urls.py`로 이동합니다...

```python
from django.urls import include, path
from rest_framework import routers

from tutorial.quickstart import views

router = routers.DefaultRouter()
router.register(r'users', views.UserViewSet)
router.register(r'groups', views.GroupViewSet)

# 자동 URL 라우팅을 사용하여 API 연결.
# 또한, 브라우저 API를 위한 로그인 URL을 포함합니다.
urlpatterns = [
    path('', include(router.urls)),
    path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

뷰 대신 뷰셋을 사용하기 때문에, 뷰셋을 라우터 클래스에 등록하여 API URL 구성을 자동으로 생성할 수 있습니다.

API URL에 대한 더 많은 제어가 필요한 경우, 일반 클래스 기반 뷰를 사용하고 URL 구성을 명시적으로 작성할 수 있습니다.

마지막으로, 브라우저 API에서 사용할 로그인 및 로그아웃 뷰를 포함하고 있습니다. 이는 선택 사항이지만, API가 인증을 요구하고 브라우저 API를 사용하고자 하는 경우 유용합니다.

## 페이지네이션

페이지네이션을 통해 페이지당 반환되는 객체 수를 제어할 수 있습니다. 이를 활성화하려면 `tutorial/settings.py`에 다음 줄을 추가합니다.

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

## 설정

`'rest_framework'`를 `INSTALLED_APPS`에 추가합니다. 설정 모듈은 `tutorial/settings.py`에 있습니다.

```python
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

이제 완료되었습니다.

---

## API 테스트

이제 구축한 API를 테스트할 준비가 되었습니다. 커맨드 라인에서 서버를 실행합니다.

    python manage.py runserver

이제 `curl`과 같은 도구를 사용하여 커맨드 라인에서 API에 액세스할 수 있습니다...

    bash: curl -u admin -H 'Accept: application/json; indent=4' http://127.0.0.1:8000/users/
    Enter host password for user 'admin':
    {
        "count": 1,
        "next": null,
        "previous": null,
        "results": [
            {
                "url": "http://127.0.0.1:8000/users/1/",
                "username": "admin",
                "email": "admin@example.com",
                "groups": []
            }
        ]
    }

또는 [httpie][httpie] 커맨드 라인 도구를 사용합니다...

    bash: http -a admin http://127.0.0.1:8000/users/
    http: password for admin@127.0.0.1:8000:: 
    $HTTP/1.1 200 OK
    ...
    {
        "count": 1,
        "next": null,
        "previous": null,
        "results": [
            {
                "email": "admin@example.com",
                "groups": [],
                "url": "http://127.0.0.1:8000/users/1/",
                "username": "admin"
            }
        ]
    }

또는 브라우저에서 `http://127.0.0.1:8000/users/` URL로 직접 접근합니다...

![빠른 시작 이미지](https://github.com/saJaeHyukc/django-rest-framework/blob/master/docs/img/quickstart.png)

브라우저를 통해 작업하는 경우, 오른쪽 상단의 컨트롤을 사용하여 로그인해야 합니다.

좋습니다. 매우 간단했죠!

REST 프레임워크의 구성 요소에 대해 더 깊이 이해하고 싶다면 [튜토리얼][tutorial]을 참조하거나 [API 가이드][guide]를 살펴보세요.

[tutorial]: 1-serialization.md
[guide]: ../api-guide/requests.md
[httpie]: https://httpie.io/docs#installation
