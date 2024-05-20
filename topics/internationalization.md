# 국제화

> 국제화 지원은 선택 사항이 아닙니다. 핵심 기능이어야 합니다.
>
> &mdash; [Jannis Leidel, Django Under the Hood에서 연설, 2015][cite].

REST 프레임워크는 번역 가능한 오류 메시지를 제공합니다. [Django의 표준 번역 메커니즘][django-translation]을 활성화하여 이러한 메시지를 원하는 언어로 표시할 수 있습니다.

이렇게 하면 다음을 수행할 수 있습니다:

* 표준 `LANGUAGE_CODE` Django 설정을 사용하여 영어 외의 다른 언어를 기본 언어로 선택합니다.
* Django에 포함된 `LocaleMiddleware`를 사용하여 클라이언트가 스스로 언어를 선택할 수 있도록 허용합니다. API 클라이언트의 일반적인 사용법은 `Accept-Language` 요청 헤더를 포함하는 것입니다.

## 국제화된 API 활성화

표준 Django `LANGUAGE_CODE` 설정을 사용하여 기본 언어를 변경할 수 있습니다:

    LANGUAGE_CODE = "es-es"

`MIDDLEWARE` 설정에 `LocaleMiddleware`를 추가하여 요청별 언어 요청을 활성화할 수 있습니다:

    MIDDLEWARE = [
        ...
        'django.middleware.locale.LocaleMiddleware'
    ]

요청별 국제화가 활성화되면 클라이언트 요청은 가능한 경우 `Accept-Language` 헤더를 존중합니다. 예를 들어, 지원되지 않는 미디어 유형에 대한 요청을 만들어 보겠습니다:

**요청**

    GET /api/users HTTP/1.1
    Accept: application/xml
    Accept-Language: es-es
    Host: example.org

**응답**

    HTTP/1.0 406 NOT ACCEPTABLE

    {"detail": "No se ha podido satisfacer la solicitud de cabecera de Accept."}

REST 프레임워크에는 표준 예외 사례와 시리얼라이저 유효성 검사 오류 모두에 대한 내장 번역이 포함되어 있습니다.

번역은 오류 문자열 자체에만 적용됩니다. 오류 메시지의 형식과 필드 이름의 키는 동일하게 유지됩니다. 예를 들어 `400 Bad Request` 응답 본문은 다음과 같이 보일 수 있습니다:

    {"detail": {"username": ["Esse campo deve ser único."]}}

응답의 `detail` 및 `non_field_errors`와 같은 부분에 대해 다른 문자열을 사용하려면 [사용자 정의 예외 처리기][custom-exception-handler]를 사용하여 이 동작을 수정할 수 있습니다.

#### 지원되는 언어 세트 지정

기본적으로 모든 사용 가능한 언어가 지원됩니다.

사용 가능한 언어 중 일부만 지원하려면 Django의 표준 `LANGUAGES` 설정을 사용하십시오:

    LANGUAGES = [
        ('de', _('German')),
        ('en', _('English')),
    ]

## 새로운 번역 추가

REST 프레임워크 번역은 [Transifex][transifex-project]를 사용하여 온라인으로 관리됩니다. Transifex 서비스를 사용하여 새로운 번역 언어를 추가할 수 있습니다. 유지 관리 팀은 이러한 번역 문자열이 REST 프레임워크 패키지에 포함되도록 할 것입니다.

프로젝트에 로컬로 번역 문자열을 추가해야 하는 경우가 있습니다. 이는 다음과 같은 경우 필요할 수 있습니다:

* Transifex에서 아직 번역되지 않은 언어로 REST 프레임워크를 사용하려는 경우.
* 프로젝트에 REST 프레임워크의 기본 번역 문자열에 포함되지 않은 사용자 정의 오류 메시지가 포함된 경우.

#### 새로운 언어를 로컬로 번역

이 가이드는 Django 앱을 번역하는 방법에 이미 익숙하다는 가정하에 작성되었습니다. 익숙하지 않은 경우, 먼저 [Django의 번역 문서][django-translation]를 읽으십시오.

새 언어를 번역하는 경우 기존 REST 프레임워크 오류 메시지를 번역해야 합니다:

1. 국제화 리소스를 저장할 새 폴더를 만듭니다. 이 경로를 [`LOCALE_PATHS`][django-locale-paths] 설정에 추가합니다.

2. 이제 번역하려는 언어의 하위 폴더를 만듭니다. 폴더는 [로케일 이름][django-locale-name] 표기법을 사용하여 이름을 지정해야 합니다. 예: `de`, `pt_BR`, `es_AR`.

3. 이제 REST 프레임워크 소스 코드에서 [기본 번역 파일][django-po-source]을 번역 폴더로 복사합니다.

4. 방금 복사한 `django.po` 파일을 편집하여 모든 오류 메시지를 번역합니다.

5. `manage.py compilemessages -l pt_BR` 명령을 실행하여 Django가 사용할 수 있는 번역을 만듭니다. `<...>/locale/pt_BR/LC_MESSAGES`의 `django.po` 파일을 처리하는 메시지가 표시됩니다.

6. 개발 서버를 다시 시작하여 변경 사항이 적용되는지 확인합니다.

프로젝트 코드베이스 내에 있는 사용자 정의 오류 메시지만 번역하는 경우 `LOCALE_PATHS` 폴더에 REST 프레임워크 소스 `django.po` 파일을 복사할 필요가 없으며, 대신 Django의 표준 `makemessages` 프로세스를 실행할 수 있습니다.

## 언어 결정 방법

요청별 언어 기본 설정을 허용하려면 `MIDDLEWARE` 설정에 `django.middleware.locale.LocaleMiddleware`를 포함해야 합니다.

언어 기본 설정을 결정하는 방법에 대한 자세한 내용은 [Django 문서][django-language-preference]에서 확인할 수 있습니다. 참조용으로, 방법은 다음과 같습니다:

1. 먼저, 요청된 URL에 언어 접두사가 있는지 확인합니다.
2. 그렇지 않은 경우, 현재 사용자의 세션에서 `LANGUAGE_SESSION_KEY` 키를 찾습니다.
3. 그래도 안 되면, 쿠키를 확인합니다.
4. 그래도 안 되면, `Accept-Language` HTTP 헤더를 확인합니다.
5. 그래도 안 되면, 전역 `LANGUAGE_CODE` 설정을 사용합니다.

API 클라이언트의 경우, 세션 및 쿠키는 세션 인증을 사용하지 않는 한 사용할 수 없으며, 일반적으로 언어 URL 접두사보다 `Accept-Language` 헤더를 사용하는 것이 더 나은 방법입니다.

[cite]: https://youtu.be/Wa0VfS2q94Y
[django-translation]: https://docs.djangoproject.com/en/stable/topics/i18n/translation
[custom-exception-handler]: ../api-guide/exceptions.md#custom-exception-handling
[transifex-project]: https://www.transifex.com/projects/p/django-rest-framework/
[django-po-source]: https://raw.githubusercontent.com/encode/django-rest-framework/master/rest_framework/locale/en_US/LC_MESSAGES/django.po
[django-language-preference]: https://docs.djangoproject.com/en/stable/topics/i18n/translation/#how-django-discovers-language-preference
[django-locale-paths]: https://docs.djangoproject.com/en/stable/ref/settings/#std:setting-LOCALE_PATHS
[django-locale-name]: https://docs.djangoproject.com/en/stable/topics/i18n/#term-locale-name
