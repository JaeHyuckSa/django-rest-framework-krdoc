![Logo by Jake 'Sid' Smith](https://github.com/encode/django-rest-framework/raw/master/docs/img/logo.png)

Django REST 프레임워크는 웹 API를 구축하기 위한 강력하고 유연한 도구입니다.

REST 프레임워크를 사용해야 할 이유는 다음과 같습니다:

* 웹 브라우저 API는 개발자에게 큰 편리함을 제공합니다.
* [인증 정책][authentication]에는 [OAuth1a][oauth1-section] 및 [OAuth2][oauth2-section]용 패키지가 포함됩니다.
* [직렬화][serializers]는 [ORM][modelserializer-section] 및 [비 ORM][serializer-section] 데이터 소스를 모두 지원합니다.
* 모든 것을 맞춤 설정할 수 있습니다 - [기능 기반 뷰][functionview-section]만 사용해도 [더 많은][generic-views] [강력한][viewsets] [기능][routers]을 사용할 필요가 없습니다.
* 광범위한 문서와 [훌륭한 커뮤니티 지원][group]이 제공됩니다.
* [Mozilla][mozilla], [Red Hat][redhat], [Heroku][heroku], [Eventbrite][eventbrite]를 포함한 국제적으로 인정받는 회사에서 사용하고 신뢰받고 있습니다.

---

## 요구 사항

REST 프레임워크는 다음을 필요로 합니다:

* Python (3.6, 3.7, 3.8, 3.9, 3.10, 3.11)
* Django (3.0, 3.1, 3.2, 4.0, 4.1, 4.2, 5.0)

각 Python 및 Django 시리즈의 최신 패치 릴리스를 **강력히 권장**하며 공식적으로 지원합니다.

다음 패키지는 선택 사항입니다:

* [PyYAML][pyyaml], [uritemplate][uriteemplate] (5.1+, 3.0.0+) - 스키마 생성 지원.
* [Markdown][markdown] (3.0.0+) - 브라우저 API를 위한 마크다운 지원.
* [Pygments][pygments] (2.4.0+) - 마크다운 처리에 구문 강조를 추가.
* [django-filter][django-filter] (1.0.1+) - 필터링 지원.
* [django-guardian][django-guardian] (1.1.1+) - 객체 수준 권한 지원.

## 설치

`pip`을 사용하여 설치하고 필요한 선택적 패키지를 포함하세요...

    pip install djangorestframework
    pip install markdown       # 브라우저 API를 위한 마크다운 지원.
    pip install django-filter  # 필터링 지원

...또는 github에서 프로젝트를 클론하세요.

    git clone https://github.com/encode/django-rest-framework

`INSTALLED_APPS` 설정에 `'rest_framework'`를 추가하세요.

    INSTALLED_APPS = [
        ...
        'rest_framework',
    ]

브라우저 API를 사용하려는 경우 REST 프레임워크의 로그인 및 로그아웃 뷰도 추가하는 것이 좋습니다. 다음을 루트 `urls.py` 파일에 추가하세요.

    urlpatterns = [
        ...
        path('api-auth/', include('rest_framework.urls'))
    ]

URL 경로는 원하는 대로 설정할 수 있습니다.

## 예제

REST 프레임워크를 사용하여 간단한 모델 기반 API를 만드는 예제를 살펴보겠습니다.

프로젝트의 사용자 정보를 접근하는 읽기/쓰기 API를 만들 것입니다.

REST 프레임워크 API의 모든 글로벌 설정은 `REST_FRAMEWORK`라는 단일 구성 딕셔너리에 저장됩니다. `settings.py` 모듈에 다음을 추가하세요.

    REST_FRAMEWORK = {
        # Django의 표준 `django.contrib.auth` 권한을 사용하거나,
        # 인증되지 않은 사용자에게 읽기 전용 액세스를 허용합니다.
        'DEFAULT_PERMISSION_CLASSES': [
            'rest_framework.permissions.DjangoModelPermissionsOrAnonReadOnly'
        ]
    }

`INSTALLED_APPS`에 `rest_framework`도 추가했는지 확인하세요.

이제 API를 만들 준비가 되었습니다.
다음은 프로젝트의 루트 `urls.py` 모듈입니다:

    from django.urls import path, include
    from django.contrib.auth.models import User
    from rest_framework import routers, serializers, viewsets

    # 직렬화는 API 표현을 정의합니다.
    class UserSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = User
            fields = ['url', 'username', 'email', 'is_staff']

    # ViewSets는 뷰 동작을 정의합니다.
    class UserViewSet(viewsets.ModelViewSet):
        queryset = User.objects.all()
        serializer_class = UserSerializer

    # 라우터는 URL 구성을 자동으로 결정하는 쉬운 방법을 제공합니다.
    router = routers.DefaultRouter()
    router.register(r'users', UserViewSet)

    # 자동 URL 라우팅을 사용하여 API를 연결합니다.
    # 또한, 브라우저 API를 위한 로그인 URL을 포함합니다.
    urlpatterns = [
        path('', include(router.urls)),
        path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))
    ]

이제 [http://127.0.0.1:8000/](http://127.0.0.1:8000/)에서 브라우저에서 API를 열고 새로운 'users' API를 볼 수 있습니다. 오른쪽 상단의 로그인 컨트롤을 사용하면 시스템에서 사용자를 추가, 생성 및 삭제할 수도 있습니다.

## 빠른 시작

빨리 시작하고 싶으신가요? [빠른 시작 가이드][quickstart]는 REST 프레임워크로 API를 구축하는 가장 빠른 방법입니다.

## 개발

레포지토리를 클론하고, 테스트 스위트를 실행하며, REST 프레임워크에 변경 사항을 기여하는 방법에 대한 정보는 [기여 가이드라인][contributing]을 참조하세요.

## 지원

지원이 필요하시면 [REST 프레임워크 토론 그룹][group]을 참조하거나, `irc.libera.chat`의 `#restframework` 채널을 이용하거나, ['django-rest-framework'][django-rest-framework-tag] 태그를 포함하여 [Stack Overflow][stack-overflow]에 질문을 올려주세요.

우선 지원이 필요하시면 [전문 또는 프리미엄 후원 계획](https://fund.django-rest-framework.org/topics/funding/)에 가입하세요.

## 보안

보안 문제는 [Django 보안 팀](https://www.djangoproject.com/foundation/teams/#security-team)의 감독하에 처리됩니다.

**보안 문제는 security@djangoproject.com으로 이메일을 보내주세요.**

프로젝트 관리자는 필요한 경우 공공 공개 전에 문제를 해결하기 위해 협력할 것입니다.


[mozilla]: https://www.mozilla.org/en-US/about/
[redhat]: https://www.redhat.com/
[heroku]: https://www.heroku.com/
[eventbrite]: https://www.eventbrite.co.uk/about/
[pyyaml]: https://pypi.org/project/PyYAML/
[uriteemplate]: https://pypi.org/project/uritemplate/
[markdown]: https://pypi.org/project/Markdown/
[pygments]: https://pypi.org/project/Pygments/
[django-filter]: https://pypi.org/project/django-filter/
[django-guardian]: https://github.com/django-guardian/django-guardian
[index]: .
[oauth1-section]: api-guide/authentication/#django-rest-framework-oauth
[oauth2-section]: api-guide/authentication/#django-oauth-toolkit
[serializer-section]: api-guide/serializers#serializers
[modelserializer-section]: api-guide/serializers#modelserializer
[functionview-section]: api-guide/views#function-based-views
[sandbox]: https://restframework.herokuapp.com/
[sponsors]: https://fund.django-rest-framework.org/topics/funding/#our-sponsors

[quickstart]: tutorial/quickstart.md

[generic-views]: api-guide/generic-views.md
[viewsets]: api-guide/viewsets.md
[routers]: api-guide/routers.md
[serializers]: api-guide/serializers.md
[authentication]: api-guide/authentication.md

[contributing]: community/contributing.md
[funding]: community/funding.md

[group]: https://groups.google.com/forum/?fromgroups#!forum/django-rest-framework
[stack-overflow]: https://stackoverflow.com/
[django-rest-framework-tag]: https://stackoverflow.com/questions/tagged/django-rest-framework
[security-mail]: mailto:rest-framework-security@googlegroups.com
[twitter]: https://twitter.com/_tomchristie
