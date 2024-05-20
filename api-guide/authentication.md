---
source:
    - authentication.py
---

# 인증

> 인증은 플러그 가능한 방식이어야 합니다.
>
> &mdash; Jacob Kaplan-Moss, ["REST 최악의 관행"][cite]

인증은 들어오는 요청을 사용자가 요청을 보낸 사용자나 서명된 토큰과 같은 식별 자격 증명 집합과 연관시키는 메커니즘입니다. [권한] 및 [스로틀링] 정책은 이러한 자격 증명을 사용하여 요청을 허용할지 여부를 결정할 수 있습니다.

REST 프레임워크는 기본적으로 여러 가지 인증 스키마를 제공하며, 사용자 정의 스키마를 구현할 수도 있습니다.

인증은 항상 뷰의 가장 시작 부분에서 실행되며, 권한 및 스로틀링 검사 전에 실행되고, 다른 코드가 진행되기 전에 실행됩니다.

`request.user` 속성은 일반적으로 `contrib.auth` 패키지의 `User` 클래스의 인스턴스로 설정됩니다.

`request.auth` 속성은 요청에 서명된 인증 토큰과 같은 추가 인증 정보를 나타내는 데 사용됩니다.

---

**참고:** 인증 자체만으로는 들어오는 요청을 허용하거나 거부하지 않습니다. 단순히 요청이 이루어진 자격 증명을 식별할 뿐입니다.

API의 권한 정책 설정에 대한 자세한 내용은 [권한 문서][permission]를 참조하십시오.

---

## 인증 결정 방법

인증 스키마는 항상 클래스 목록으로 정의됩니다. REST 프레임워크는 목록의 각 클래스를 사용하여 인증을 시도하며, 처음으로 인증에 성공한 클래스의 반환 값을 사용하여 `request.user`와 `request.auth`를 설정합니다.

어느 클래스도 인증하지 못하면, `request.user`는 `django.contrib.auth.models.AnonymousUser`의 인스턴스로 설정되고, `request.auth`는 `None`으로 설정됩니다.

인증되지 않은 요청에 대한 `request.user`와 `request.auth` 값은 `UNAUTHENTICATED_USER` 및 `UNAUTHENTICATED_TOKEN` 설정을 사용하여 수정할 수 있습니다.

## 인증 스키마 설정

기본 인증 스키마는 `DEFAULT_AUTHENTICATION_CLASSES` 설정을 사용하여 전역적으로 설정할 수 있습니다. 예를 들어:

    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': [
            'rest_framework.authentication.BasicAuthentication',
            'rest_framework.authentication.SessionAuthentication',
        ]
    }

또한 `APIView` 클래스 기반 뷰를 사용하여 뷰 또는 뷰셋별로 인증 스키마를 설정할 수도 있습니다.

    from rest_framework.authentication import SessionAuthentication, BasicAuthentication
    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ExampleView(APIView):
        authentication_classes = [SessionAuthentication, BasicAuthentication]
        permission_classes = [IsAuthenticated]

        def get(self, request, format=None):
            content = {
                'user': str(request.user),  # `django.contrib.auth.User` 인스턴스.
                'auth': str(request.auth),  # None
            }
            return Response(content)

또는 함수 기반 뷰에 `@api_view` 데코레이터를 사용하는 경우.

    @api_view(['GET'])
    @authentication_classes([SessionAuthentication, BasicAuthentication])
    @permission_classes([IsAuthenticated])
    def example_view(request, format=None):
        content = {
            'user': str(request.user),  # `django.contrib.auth.User` 인스턴스.
            'auth': str(request.auth),  # None
        }
        return Response(content)

## 인증되지 않음 및 금지 응답

인증되지 않은 요청이 권한을 거부당할 때 적절할 수 있는 두 가지 다른 오류 코드가 있습니다.

* [HTTP 401 인증되지 않음][http401]
* [HTTP 403 권한 거부][http403]

HTTP 401 응답에는 항상 클라이언트가 인증하는 방법을 지시하는 `WWW-Authenticate` 헤더가 포함되어야 합니다. HTTP 403 응답에는 `WWW-Authenticate` 헤더가 포함되지 않습니다.

사용되는 응답 종류는 인증 스키마에 따라 다릅니다. 여러 인증 스키마가 사용될 수 있지만, 응답 유형을 결정하는 데는 하나의 스키마만 사용할 수 있습니다. **응답 유형을 결정할 때 뷰에 설정된 첫 번째 인증 클래스가 사용됩니다.**

요청이 성공적으로 인증되지만 여전히 요청을 수행할 권한이 없는 경우, 인증 스키마에 관계없이 항상 `403 권한 거부` 응답이 사용됩니다.

## Apache mod_wsgi 특정 구성

[mod_wsgi를 사용하는 Apache][mod_wsgi_official]에 배포하는 경우, 기본적으로 WSGI 애플리케이션에 권한 부여 헤더가 전달되지 않습니다. 이는 인증이 애플리케이션 수준이 아닌 Apache에서 처리될 것으로 가정하기 때문입니다.

Apache에 배포하고 세션 기반 인증을 사용하지 않는 경우, mod_wsgi를 명시적으로 구성하여 애플리케이션에 필요한 헤더를 전달해야 합니다. 이는 적절한 컨텍스트에서 `WSGIPassAuthorization` 지시어를 지정하고 `On`으로 설정하여 수행할 수 있습니다.

    # 서버 구성, 가상 호스트, 디렉토리 또는 .htaccess에 추가 가능
    WSGIPassAuthorization On

---

# API 참조

## BasicAuthentication

이 인증 스키마는 [HTTP 기본 인증][basicauth]을 사용하며, 사용자의 사용자 이름과 비밀번호에 대해 서명됩니다. 기본 인증은 일반적으로 테스트에만 적합합니다.

인증에 성공하면, `BasicAuthentication`은 다음 자격 증명을 제공합니다.

* `request.user`는 Django `User` 인스턴스가 됩니다.
* `request.auth`는 `None`이 됩니다.

권한이 거부된 인증되지 않은 응답은 적절한 WWW-Authenticate 헤더가 포함된 `HTTP 401 Unauthorized` 응답을 생성합니다. 예를 들어:

    WWW-Authenticate: Basic realm="api"

**참고:** 프로덕션 환경에서 `BasicAuthentication`을 사용하는 경우, API가 `https`를 통해서만 사용 가능하도록 해야 합니다. 또한, API 클라이언트가 로그인 시 사용자 이름과 비밀번호를 항상 다시 요청하고 해당 세부 정보를 영구 저장소에 저장하지 않도록 해야 합니다.

## TokenAuthentication

---

**참고:** Django REST 프레임워크에서 제공하는 토큰 인증은 비교적 단순한 구현입니다.

여러 사용자당 하나 이상의 토큰을 허용하고, 더 엄격한 보안 구현 세부 사항을 갖추며, 토큰 만료를 지원하는 구현을 원한다면 [Django REST Knox][django-rest-knox] 서드파티 패키지를 참조하십시오.

---

이 인증 스키마는 단순한 토큰 기반 HTTP 인증 스키마를 사용합니다. 토큰 인증은 네이티브 데스크탑 및 모바일 클라이언트와 같은 클라이언트-서버 설정에 적합합니다.

`TokenAuthentication` 스키마를 사용하려면, `TokenAuthentication`을 포함하도록 [인증 클래스를 구성](#setting-the-authentication-scheme)해야 하며, 추가로 `rest_framework.authtoken`을 `INSTALLED_APPS` 설정에 포함해야 합니다:

    INSTALLED_APPS = [
        ...
        'rest_framework.authtoken'
    ]

설정을 변경한 후 `manage.py migrate`를 실행하십시오.

`rest_framework.authtoken` 앱은 Django 데이터베이스 마이그레이션을 제공합니다.

사용자에 대한 토큰도 생성해야 합니다.

    from rest_framework.authtoken.models import Token

    token = Token.objects.create(user=...)
    print(token.key)

클라이언트가 인증하려면, 토큰 키를 `Authorization` HTTP 헤더에 포함해야 합니다. 키는 "Token" 문자열 리터럴로 접두사로 붙여야 하며, 두 문자열 사이에 공백을 넣어야 합니다. 예를 들어:

    Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b

*헤더에서 `Bearer`와 같은 다른 키워드를 사용하려면, `TokenAuthentication`을 서브클래싱하고 `keyword` 클래스 변수를 설정하십시오.*

인증에 성공하면, `TokenAuthentication`은 다음 자격 증명을 제공합니다.

* `request.user`는 Django `User` 인스턴스가 됩니다.
* `request.auth`는 `rest_framework.authtoken.models.Token` 인스턴스가 됩니다.

권한이 거부된 인증되지 않은 응답은 적절한 WWW-Authenticate 헤더가 포함된 `HTTP 401 Unauthorized` 응답을 생성합니다. 예를 들어:

    WWW-Authenticate: Token

`curl` 명령줄 도구는 토큰 인증된 API를 테스트하는 데 유용할 수 있습니다. 예를 들어:

    curl -X GET http://127.0.0.1:8000/api/example/ -H 'Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'

---

**참고:** 프로덕션 환경에서 `TokenAuthentication`을 사용하는 경우, API가 `https`를 통해서만 사용 가능하도록 해야 합니다.

---

### 토큰 생성

#### 시그널 사용

모든 사용자에게 자동으로 생성된 토큰을 부여하려면, User의 `post_save` 시그널을 캐치하면 됩니다.

    from django.conf import settings
    from django.db.models.signals import post_save
    from django.dispatch import receiver
    from rest_framework.authtoken.models import Token

    @receiver(post_save, sender=settings.AUTH_USER_MODEL)
    def create_auth_token(sender, instance=None, created=False, **kwargs):
        if created:
            Token.objects.create(user=instance)

이 코드 스니펫을 설치된 `models.py` 모듈 또는 Django가 시작 시 가져올 위치에 배치해야 합니다.

이미 사용자를 생성한 경우, 기존 사용자에게 토큰을 생성하려면 다음과 같이 할 수 있습니다:

    from django.contrib.auth.models import User
    from rest_framework.authtoken.models import Token

    for user in User.objects.all():
        Token.objects.get_or_create(user=user)

#### API 엔드포인트 노출

`TokenAuthentication`을 사용할 때, 클라이언트가 사용자 이름과 비밀번호를 제공하여 토큰을 얻을 수 있는 메커니즘을 제공하고 싶을 수 있습니다. REST 프레임워크는 이 동작을 제공하는 내장 뷰를 제공합니다. 이를 사용하려면 `obtain_auth_token` 뷰를 URLconf에 추가하십시오:

    from rest_framework.authtoken import views
    urlpatterns += [
        path('api-token-auth/', views.obtain_auth_token)
    ]

패턴의 URL 부분은 원하는 대로 사용할 수 있습니다.

`obtain_auth_token` 뷰는 유효한 `username` 및 `password` 필드가 폼 데이터나 JSON으로 뷰에 POST될 때 JSON 응답을 반환합니다:

    { 'token' : '9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b' }

기본적으로 `obtain_auth_token` 뷰는 설정에서 기본 렌더러 및 파서 클래스를 사용하지 않고 명시적으로 JSON 요청 및 응답을 사용합니다.

기본적으로 `obtain_auth_token` 뷰에는 권한 또는 스로틀링이 적용되지 않습니다. 스로틀링을 적용하려면 뷰 클래스를 재정의하고 `throttle_classes` 속성을 포함시켜야 합니다.

`obtain_auth_token` 뷰의 사용자 정의 버전이 필요한 경우, `ObtainAuthToken` 뷰 클래스를 서브클래싱하고 이를 URLconf에 사용하여 수행할 수 있습니다.

예를 들어, `token` 값 외에 추가 사용자 정보를 반환할 수 있습니다:

    from rest_framework.authtoken.views import ObtainAuthToken
    from rest_framework.authtoken.models import Token
    from rest_framework.response import Response

    class CustomAuthToken(ObtainAuthToken):

        def post(self, request, *args, **kwargs):
            serializer = self.serializer_class(data=request.data,
                                               context={'request': request})
            serializer.is_valid(raise_exception=True)
            user = serializer.validated_data['user']
            token, created = Token.objects.get_or_create(user=user)
            return Response({
                'token': token.key,
                'user_id': user.pk,
                'email': user.email
            })

`urls.py`에서:

    urlpatterns += [
        path('api-token-auth/', CustomAuthToken.as_view())
    ]


#### Django 관리자와 함께 사용

토큰을 수동으로 생성할 수도 있습니다. 많은 사용자 기반을 사용하는 경우, `TokenAdmin` 클래스를 원숭이 패치하여 사용자 지정하는 것이 좋습니다. 특히 `user` 필드를 `raw_field`로 선언하여 사용자 지정할 수 있습니다.

`your_app/admin.py`:

    from rest_framework.authtoken.admin import TokenAdmin

    TokenAdmin.raw_id_fields = ['user']


#### Django manage.py 명령 사용

버전 3.6.4부터 다음 명령을 사용하여 사용자 토큰을 생성할 수 있습니다:

    ./manage.py drf_create_token <username>

이 명령은 주어진 사용자의 API 토큰을 반환하며, 존재하지 않는 경우 생성합니다:

    Generated token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b for user user1

토큰이 손상되거나 유출된 경우 다시 생성하려면 추가 매개변수를 전달할 수 있습니다:

    ./manage.py drf_create_token -r <username>


## SessionAuthentication

이 인증 스키마는 Django의 기본 세션 백엔드를 사용하여 인증합니다. 세션 인증은 웹사이트와 동일한 세션 컨텍스트에서 실행되는 AJAX 클라이언트에 적합합니다.

인증에 성공하면, `SessionAuthentication`은 다음 자격 증명을 제공합니다.

* `request.user`는 Django `User` 인스턴스가 됩니다.
* `request.auth`는 `None`이 됩니다.

권한이 거부된 인증되지 않은 응답은 `HTTP 403 Forbidden` 응답을 생성합니다.

세션 인증을 사용하는 AJAX 스타일 API를 사용하는 경우, `PUT`, `PATCH`, `POST` 또는 `DELETE` 요청과 같은 "안전하지 않은" HTTP 메서드 호출에 대해 유효한 CSRF 토큰을 포함해야 합니다. 자세한 내용은 [Django CSRF 문서][csrf-ajax]를 참조하십시오.

**경고**: 로그인 페이지를 생성할 때 항상 Django의 표준 로그인 뷰를 사용하십시오. 이를 통해 로그인 뷰가 적절히 보호되도록 합니다.

REST 프레임워크의 CSRF 검증은 동일한 뷰에 세션 기반 및 비세션 기반 인증을 지원해야 하기 때문에 표준 Django와 약간 다르게 작동합니다. 이는 인증된 요청만 CSRF 토큰이 필요하고 익명 요청은 CSRF 토큰 없이 전송될 수 있음을 의미합니다. 이 동작은 항상 CSRF 검증이 적용되어야 하는 로그인 뷰에는 적합하지 않습니다.

## RemoteUserAuthentication

이 인증 스키마를 사용하면 웹 서버에 인증을 위임하여 `REMOTE_USER` 환경 변수를 설정할 수 있습니다.

이를 사용하려면 `AUTHENTICATION_BACKENDS` 설정에 `django.contrib.auth.backends.RemoteUserBackend` (또는 서브클래스)이 있어야 합니다. 기본적으로, `RemoteUserBackend`는 이미 존재하지 않는 사용자 이름에 대해 `User` 객체를 생성합니다. 이를 변경하려면 [Django 문서](https://docs.djangoproject.com/en/stable/howto/auth-remote-user/)를 참조하십시오.

인증에 성공하면, `RemoteUserAuthentication`은 다음 자격 증명을 제공합니다:

* `request.user`는 Django `User` 인스턴스가 됩니다.
* `request.auth`는 `None`이 됩니다.

인증 방법 구성에 대한 자세한 내용은 웹 서버의 문서를 참조하십시오. 예를 들어:

* [Apache Authentication How-To](https://httpd.apache.org/docs/2.4/howto/auth.html)
* [NGINX (Restricting Access)](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/)


# 사용자 정의 인증

사용자 정의 인증 스키마를 구현하려면, `BaseAuthentication`을 서브클래싱하고 `.authenticate(self, request)` 메서드를 재정의하십시오. 이 메서드는 인증이 성공하면 `(user, auth)`의 두 요소 튜플을 반환하거나, 그렇지 않으면 `None`을 반환해야 합니다.

일부 상황에서는 `None`을 반환하는 대신 `.authenticate()` 메서드에서 `AuthenticationFailed` 예외를 발생시키고 싶을 수 있습니다.

일반적으로 따라야 할 접근 방식은 다음과 같습니다:

* 인증이 시도되지 않으면 `None`을 반환하십시오. 사용 중인 다른 인증 스키마도 여전히 확인됩니다.
* 인증을 시도했지만 실패하면 `AuthenticationFailed` 예외를 발생시키십시오. 권한 검사에 관계없이 즉시 오류 응답이 반환되며, 다른 인증 스키마는 확인되지 않습니다.

또한 `.authenticate_header(self, request)` 메서드를 재정의할 수 있습니다. 구현된 경우, `HTTP 401 Unauthorized` 응답의 `WWW-Authenticate` 헤더 값으로 사용할 문자열을 반환해야 합니다.

`.authenticate_header()` 메서드가 재정의되지 않으면, 인증 스키마는 인증되지 않은 요청에 대해 접근이 거부될 때 `HTTP 403 Forbidden` 응답을 반환합니다.

---

**참고:** 요청 객체의 `.user` 또는 `.auth` 속성에 의해 사용자 정의 인증기가 호출될 때, `AttributeError`가 `WrappedAttributeError`로 다시 발생할 수 있습니다. 이는 원래 예외가 외부 속성 접근에 의해 억제되지 않도록 하기 위해 필요합니다. Python은 `AttributeError`가 사용자 정의 인증기에서 발생한 것임을 인식하지 못하고 대신 요청 객체에 `.user` 또는 `.auth` 속성이 없다고 가정합니다. 이러한 오류는 사용자 정의 인증기에서 수정하거나 처리해야 합니다.

---

## 예시

다음 예시는 들어오는 모든 요청을 'X-USERNAME'이라는 사용자 정의 요청 헤더에 있는 사용자 이름으로 인증합니다.

    from django.contrib.auth.models import User
    from rest_framework import authentication
    from rest_framework import exceptions

    class ExampleAuthentication(authentication.BaseAuthentication):
        def authenticate(self, request):
            username = request.META.get('HTTP_X_USERNAME')
            if not username:
                return None

            try:
                user = User.objects.get(username=username)
            except User.DoesNotExist:
                raise exceptions.AuthenticationFailed('No such user')

            return (user, None)

---

# 서드파티 패키지

다음 서드파티 패키지도 사용할 수 있습니다.

## django-rest-knox

[Django-rest-knox][django-rest-knox] 라이브러리는 내장된 TokenAuthentication 스키마보다 더 안전하고 확장 가능한 방식으로 토큰 기반 인증을 처리하기 위한 모델 및 뷰를 제공합니다. 단일 페이지 애플리케이션 및 모바일 클라이언트를 염두에 두고 만들어졌습니다. 클라이언트당 토큰을 제공하며, 일부 다른 인증(일반적으로 기본 인증)을 제공할 때 토큰을 생성하고, 토큰을 삭제하며, 사용자가 로그인한 모든 클라이언트를 로그아웃시키는 뷰를 제공합니다.

## Django OAuth Toolkit

[Django OAuth Toolkit][django-oauth-toolkit] 패키지는 OAuth 2.0을 지원하며, Python 3.4+와 함께 작동합니다. 이 패키지는 [jazzband][jazzband]에 의해 유지 관리되며, 훌륭한 [OAuthLib][oauthlib]를 사용합니다. 이 패키지는 잘 문서화되어 있으며, 현재 **OAuth 2.0 지원을 위한 권장 패키지**입니다.

### 설치 및 구성

`pip`을 사용하여 설치합니다.

    pip install django-oauth-toolkit

패키지를 `INSTALLED_APPS`에 추가하고, REST 프레임워크 설정을 수정합니다.

    INSTALLED_APPS = [
        ...
        'oauth2_provider',
    ]

    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': [
            'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
        ]
    }

자세한 내용은 [Django REST framework - 시작하기][django-oauth-toolkit-getting-started] 문서를 참조하십시오.

## Django REST framework OAuth

[Django REST framework OAuth][django-rest-framework-oauth] 패키지는 REST 프레임워크를 위한 OAuth1 및 OAuth2 지원을 제공합니다.

이 패키지는 이전에 REST 프레임워크에 직접 포함되었으나 현재는 서드파티 패키지로 지원 및 유지 관리되고 있습니다.

### 설치 및 구성

`pip`을 사용하여 패키지를 설치합니다.

    pip install djangorestframework-oauth

구성 및 사용에 대한 자세한 내용은 [인증][django-rest-framework-oauth-authentication] 및 [권한][django-rest-framework-oauth-permissions]에 대한 Django REST framework OAuth 문서를 참조하십시오.

## JSON Web Token Authentication

JSON Web Token은 토큰 기반 인증에 사용할 수 있는 비교적 새로운 표준입니다. 내장된 TokenAuthentication 스키마와 달리, JWT 인증은 토큰을 검증하기 위해 데이터베이스를 사용할 필요가 없습니다. JWT 인증을 위한 패키지는 [djangorestframework-simplejwt][djangorestframework-simplejwt]이며, 플러그 가능한 토큰 블랙리스트 앱과 같은 몇 가지 기능을 제공합니다.

## Hawk HTTP Authentication

[HawkREST][hawkrest] 라이브러리는 [Mohawk][mohawk] 라이브러리를 기반으로 하여 API에서 [Hawk][hawk] 서명 요청 및 응답을 사용할 수 있게 합니다. [Hawk][hawk]는 공유 키로 서명된 메시지를 사용하여 두 당사자가 안전하게 통신할 수 있게 합니다. 이는 [HTTP MAC 액세스 인증][mac] (OAuth 1.0의 일부를 기반으로 함)을 기반으로 합니다.

## HTTP Signature Authentication

HTTP Signature (현재 [IETF 초안][http-signature-ietf-draft])는 HTTP 메시지에 대한 출처 인증 및 메시지 무결성을 달성하는 방법을 제공합니다. [Amazon의 HTTP Signature 스키마][amazon-http-signature]와 유사하게, 많은 서비스에서 사용되며, 상태 비저장, 요청별 인증을 허용합니다. [Elvio Toccalino][etoccalino]는 HTTP Signature 인증 메커니즘을 쉽게 사용할 수 있는 [djangorestframework-httpsignature][djangorestframework-httpsignature] (구식) 패키지를 유지 관리합니다. 업데이트된 포크 버전은 [drf-httpsig][drf-httpsig]입니다.

## Djoser

[Djoser][djoser] 라이브러리는 회원가입, 로그인, 로그아웃, 비밀번호 재설정 및 계정 활성화와 같은 기본 작업을 처리하기 위한 일련의 뷰를 제공합니다. 이 패키지는 사용자 정의 사용자 모델과 함께 작동하며, 토큰 기반 인증을 사용합니다. Django 인증 시스템의 사용 가능한 REST 구현입니다.

## django-rest-auth / dj-rest-auth

이 라이브러리는 등록, 인증(소셜 미디어 인증 포함), 비밀번호 재설정, 사용자 세부 정보 검색 및 업데이트 등을 위한 일련의 REST API 엔드포인트를 제공합니다. 이러한 API 엔드포인트를 사용하면 AngularJS, iOS, Android 등의 클라이언트 앱이 Django 백엔드 사이트와 독립적으로 통신할 수 있습니다.

현재 이 프로젝트에는 두 개의 포크가 있습니다.

* [Django-rest-auth][django-rest-auth]는 원래 프로젝트였으나, [현재 업데이트를 받지 않고 있습니다](https://github.com/Tivix/django-rest-auth/issues/568).
* [Dj-rest-auth][dj-rest-auth]는 이 프로젝트의 새로운 포크입니다.

## drf-social-oauth2

[Drf-social-oauth2][drf-social-oauth2]는 Facebook, Google, Twitter, Orcid 등 주요 소셜 oauth2 벤더로 인증할 수 있도록 도와주는 프레임워크입니다. 간편한 설정으로 JWT 방식으로 토큰을 생성합니다.

## drfpasswordless

[drfpasswordless][drfpasswordless]는 Django REST Framework의 TokenAuthentication 스키마에 비밀번호 없는 지원을 추가합니다. 사용자는 이메일 주소나 휴대폰 번호와 같은 연락처로 전송된 토큰으로 로그인하고 가입합니다.

## django-rest-authemail

[django-rest-authemail][django-rest-authemail]은 사용자 가입 및 인증을 위한 RESTful API 인터페이스를 제공합니다. 사용자 이름 대신 이메일 주소를 사용하여 인증합니다. API 엔드포인트는 가입, 가입 이메일 확인, 로그인, 로그아웃, 비밀번호 재설정, 비밀번호 재설정 확인, 이메일 변경, 이메일 변경 확인, 비밀번호 변경, 사용자 세부 정보 검색을 제공합니다. 완전히 기능적인 예제 프로젝트와 자세한 지침이 포함되어 있습니다.

## Django-Rest-Durin

[Django-Rest-Durin][django-rest-durin]은 여러 웹/CLI/모바일 API 클라이언트에 대해 하나의 인터페이스로 토큰 인증을 수행하는 라이브러리를 제공합니다. 각 API 클라이언트가 API를 사용할 때마다 다른 토큰 구성을 허용합니다. 이 라이브러리는 사용자당 여러 토큰을 지원하며, Django-Rest-Framework와 함께 작동하는 사용자 정의 모델, 뷰, 권한을 제공합니다. 토큰 만료 시간은 API 클라이언트마다 다를 수 있으며, Django 관리 인터페이스를 통해 사용자 정의할 수 있습니다.

자세한 내용은 [문서](https://django-rest-durin.readthedocs.io/en/latest/index.html)를 참조하십시오.

[cite]: https://jacobian.org/writing/rest-worst-practices/
[http401]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.4.2
[http403]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.4.4
[basicauth]: https://tools.ietf.org/html/rfc2617
[permission]: permissions.md
[throttling]: throttling.md
[csrf-ajax]: https://docs.djangoproject.com/en/stable/howto/csrf/#using-csrf-protection-with-ajax
[mod_wsgi_official]: https://modwsgi.readthedocs.io/en/develop/configuration-directives/WSGIPassAuthorization.html
[django-oauth-toolkit-getting-started]: https://django-oauth-toolkit.readthedocs.io/en/latest/rest-framework/getting_started.html
[django-rest-framework-oauth]: https://jpadilla.github.io/django-rest-framework-oauth/
[django-rest-framework-oauth-authentication]: https://jpadilla.github.io/django-rest-framework-oauth/authentication/
[django-rest-framework-oauth-permissions]: https://jpadilla.github.io/django-rest-framework-oauth/permissions/
[juanriaza]: https://github.com/juanriaza
[djangorestframework-digestauth]: https://github.com/juanriaza/django-rest-framework-digestauth
[oauth-1.0a]: https://oauth.net/core/1.0a/
[django-oauth-toolkit]: https://github.com/evonove/django-oauth-toolkit
[jazzband]: https://github.com/jazzband/
[oauthlib]: https://github.com/idan/oauthlib
[djangorestframework-simplejwt]: https://github.com/davesque/django-rest-framework-simplejwt
[etoccalino]: https://github.com/etoccalino/
[djangorestframework-httpsignature]: https://github.com/etoccalino/django-rest-framework-httpsignature
[drf-httpsig]: https://github.com/ahknight/drf-httpsig
[amazon-http-signature]: https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html
[http-signature-ietf-draft]: https://datatracker.ietf.org/doc/draft-cavage-http-signatures/
[hawkrest]: https://hawkrest.readthedocs.io/en/latest/
[hawk]: https://github.com/hueniverse/hawk
[mohawk]: https://mohawk.readthedocs.io/en/latest/
[mac]: https://tools.ietf.org/html/draft-hammer-oauth-v2-mac-token-05
[djoser]: https://github.com/sunscrapers/djoser
[django-rest-auth]: https://github.com/Tivix/django-rest-auth
[dj-rest-auth]: https://github.com/jazzband/dj-rest-auth
[drf-social-oauth2]: https://github.com/wagnerdelima/drf-social-oauth2
[django-rest-knox]: https://github.com/James1345/django-rest-knox
[drfpasswordless]: https://github.com/aaronn/django-rest-framework-passwordless
[django-rest-authemail]: https://github.com/celiao/django-rest-authemail
[django-rest-durin]: https://github.com/eshaan7/django-rest-durin
