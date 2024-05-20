---
source:
    - exceptions.py
---

# 예외

> 예외는 프로그램 구조 내에서 중앙 또는 고수준에서 오류 처리를 깔끔하게 조직할 수 있게 해줍니다.
>
> &mdash; Doug Hellmann, [Python Exception Handling Techniques][cite]

## REST 프레임워크 뷰에서의 예외 처리

REST 프레임워크의 뷰는 다양한 예외를 처리하고 적절한 오류 응답을 반환합니다.

처리되는 예외는 다음과 같습니다:

* REST 프레임워크 내에서 발생한 `APIException`의 서브클래스.
* Django의 `Http404` 예외.
* Django의 `PermissionDenied` 예외.

각 경우에 대해 REST 프레임워크는 적절한 상태 코드와 콘텐츠 유형으로 응답을 반환합니다. 응답 본문에는 오류의 성격에 대한 추가 세부 정보가 포함됩니다.

대부분의 오류 응답에는 응답 본문에 `detail` 키가 포함됩니다.

예를 들어, 다음과 같은 요청:

    DELETE http://api.example.com/foo/bar HTTP/1.1
    Accept: application/json

해당 리소스에서 `DELETE` 메서드가 허용되지 않음을 나타내는 오류 응답을 받을 수 있습니다:

    HTTP/1.1 405 Method Not Allowed
    Content-Type: application/json
    Content-Length: 42

    {"detail": "Method 'DELETE' not allowed."}

유효성 검사 오류는 약간 다르게 처리되며, 응답에서 필드 이름을 키로 사용합니다. 유효성 검사 오류가 특정 필드에 국한되지 않은 경우 `NON_FIELD_ERRORS_KEY` 설정에 설정된 문자열 값 또는 "non_field_errors" 키를 사용합니다.

유효성 검사 오류의 예는 다음과 같습니다:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json
    Content-Length: 94

    {"amount": ["A valid integer is required."], "description": ["This field may not be blank."]}

## 사용자 정의 예외 처리

API 뷰에서 발생한 예외를 응답 객체로 변환하는 핸들러 함수를 만들어 사용자 정의 예외 처리를 구현할 수 있습니다. 이를 통해 API에서 사용하는 오류 응답 스타일을 제어할 수 있습니다.

함수는 처리할 예외를 첫 번째 인수로 받고, 현재 처리 중인 뷰와 같은 추가 컨텍스트를 포함하는 사전을 두 번째 인수로 받아야 합니다. 예외 핸들러 함수는 `Response` 객체를 반환하거나 예외를 처리할 수 없는 경우 `None`을 반환해야 합니다. 핸들러가 `None`을 반환하면 예외가 다시 발생하며 Django는 표준 HTTP 500 '서버 오류' 응답을 반환합니다.

예를 들어, 모든 오류 응답에 HTTP 상태 코드를 응답 본문에 포함시키려는 경우 다음과 같이 사용자 정의 예외 핸들러를 작성할 수 있습니다:

    HTTP/1.1 405 Method Not Allowed
    Content-Type: application/json
    Content-Length: 62

    {"status_code": 405, "detail": "Method 'DELETE' not allowed."}

응답 스타일을 변경하려면 다음과 같은 사용자 정의 예외 핸들러를 작성할 수 있습니다:

    from rest_framework.views import exception_handler

    def custom_exception_handler(exc, context):
        # REST 프레임워크의 기본 예외 처리기를 먼저 호출하여 표준 오류 응답을 가져옵니다.
        response = exception_handler(exc, context)

        # 이제 HTTP 상태 코드를 응답에 추가합니다.
        if response is not None:
            response.data['status_code'] = response.status_code

        return response

기본 핸들러는 컨텍스트 인수를 사용하지 않지만, 예외 핸들러가 현재 처리 중인 뷰와 같은 추가 정보가 필요한 경우 `context['view']`로 액세스할 수 있습니다.

또한 예외 핸들러는 `EXCEPTION_HANDLER` 설정 키를 사용하여 설정에서 구성해야 합니다. 예를 들어:

    REST_FRAMEWORK = {
        'EXCEPTION_HANDLER': 'my_project.my_app.utils.custom_exception_handler'
    }

명시되지 않은 경우, `'EXCEPTION_HANDLER'` 설정은 REST 프레임워크에서 제공하는 표준 예외 처리기로 기본 설정됩니다:

    REST_FRAMEWORK = {
        'EXCEPTION_HANDLER': 'rest_framework.views.exception_handler'
    }

예외 핸들러는 발생한 예외에 의해 생성된 응답에만 호출됩니다. 시리얼라이저 유효성 검사 실패 시 제네릭 뷰에서 반환되는 `HTTP_400_BAD_REQUEST` 응답과 같이 뷰에서 직접 반환된 응답에는 사용되지 않습니다.

---

# API 참조

## APIException

**서명:** `APIException()`

`APIView` 클래스 또는 `@api_view` 내에서 발생한 모든 예외의 **기본 클래스**입니다.

사용자 정의 예외를 제공하려면 `APIException`을 서브클래싱하고 클래스에 `.status_code`, `.default_detail`, `.default_code` 속성을 설정하십시오.

예를 들어, API가 가끔 도달할 수 없는 제3자 서비스에 의존하는 경우 "503 Service Unavailable" HTTP 응답 코드를 위한 예외를 구현할 수 있습니다. 다음과 같이 할 수 있습니다:

    from rest_framework.exceptions import APIException

    class ServiceUnavailable(APIException):
        status_code = 503
        default_detail = 'Service temporarily unavailable, try again later.'
        default_code = 'service_unavailable'

#### API 예외 검사

API 예외의 상태를 검사할 수 있는 다양한 속성이 있습니다. 이를 사용하여 프로젝트에 대한 사용자 정의 예외 처리를 빌드할 수 있습니다.

사용 가능한 속성 및 메서드는 다음과 같습니다:

* `.detail` - 오류에 대한 텍스트 설명을 반환합니다.
* `.get_codes()` - 오류의 코드 식별자를 반환합니다.
* `.get_full_details()` - 텍스트 설명과 코드 식별자를 모두 반환합니다.

대부분의 경우 오류 세부 정보는 단순 항목일 것입니다:

    >>> print(exc.detail)
    You do not have permission to perform this action.
    >>> print(exc.get_codes())
    permission_denied
    >>> print(exc.get_full_details())
    {'message':'You do not have permission to perform this action.','code':'permission_denied'}

유효성 검사 오류의 경우 오류 세부 정보는 항목의 목록 또는 사전일 것입니다:

    >>> print(exc.detail)
    {"name":"This field is required.","age":"A valid integer is required."}
    >>> print(exc.get_codes())
    {"name":"required","age":"invalid"}
    >>> print(exc.get_full_details())
    {"name":{"message":"This field is required.","code":"required"},"age":{"message":"A valid integer is required.","code":"invalid"}}

## ParseError

**서명:** `ParseError(detail=None, code=None)`

`request.data`에 접근할 때 요청에 잘못된 데이터가 포함된 경우 발생합니다.

기본적으로 이 예외는 "400 Bad Request" HTTP 상태 코드로 응답합니다.

## AuthenticationFailed

**서명:** `AuthenticationFailed(detail=None, code=None)`

잘못된 인증 정보가 포함된 요청이 들어오면 발생합니다.

기본적으로 이 예외는 "401 Unauthenticated" HTTP 상태 코드로 응답하지만, 사용 중인 인증 스키마에 따라 "403 Forbidden" 응답으로 이어질 수도 있습니다. 자세한 내용은 [인증 문서][authentication]를 참조하십시오.

## NotAuthenticated

**서명:** `NotAuthenticated(detail=None, code=None)`

인증되지 않은 요청이 권한 검사를 통과하지 못하면 발생합니다.

기본적으로 이 예외는 "401 Unauthenticated" HTTP 상태 코드로 응답하지만, 사용 중인 인증 스키마에 따라 "403 Forbidden" 응답으로 이어질 수도 있습니다. 자세한 내용은 [인증 문서][authentication]를 참조하십시오.

## PermissionDenied

**서명:** `PermissionDenied(detail=None, code=None)`

인증된 요청이 권한 검사를 통과하지 못하면 발생합니다.

기본적으로 이 예외는 "403 Forbidden" HTTP 상태 코드로 응답합니다.

## NotFound

**서명:** `NotFound(detail=None, code=None)`

주어진 URL에 리소스가 존재하지 않을 때 발생합니다. 이 예외는 표준 `Http404` Django 예외와 동일합니다.

기본적으로 이 예외는 "404 Not Found" HTTP 상태 코드로 응답합니다.

## MethodNotAllowed

**서명:** `MethodNotAllowed(method, detail=None, code=None)`

뷰의 핸들러 메서드와 매핑되지 않는 요청이 들어오면 발생합니다.

기본적으로 이 예외는 "405 Method Not Allowed" HTTP 상태 코드로 응답합니다.

## NotAcceptable

**서명:** `NotAcceptable(detail=None, code=None)`

사용 가능한 렌더러 중 어느 것도 만족시킬 수 없는 `Accept` 헤더와 함께 요청이 들어오면 발생합니다.

기본적으로 이 예외는 "406 Not Acceptable" HTTP 상태 코드로 응답합니다.

## UnsupportedMediaType

**서명:** `UnsupportedMediaType(media_type, detail=None, code=None)`

`request.data`에 접근할 때 요청 데이터의 콘텐츠 유형을 처리할 수 있는 파서가 없는 경우 발생합니다.

기본적으로 이 예외는 "415 Unsupported Media Type" HTTP 상태 코드로 응답합니다.

## Throttled

**서명:** `Throttled(wait=None, detail=None, code=None)`

요청이 스로틀링 검사를 통과하지 못하면 발생합니다.

기본적으로 이 예외는 "429 Too Many Requests" HTTP 상태 코드로 응답합니다.

## ValidationError

**서명:** `ValidationError(detail=None, code=None)`

`ValidationError` 예외는 다른 `APIException` 클래스와 약간 다릅니다:

* `detail` 인수는 오류 세부 정보의 목록 또는 사전일 수 있으며, 중첩된 데이터 구조일 수도 있습니다. 사전을 사용하여 시리얼라이저의 `validate()` 메서드에서 객체 수준 유효성 검사 중에 필드 수준 오류를 지정할 수 있습니다. 예를 들어, `raise serializers.ValidationError({'name': 'Please enter a valid name.'})`
* Django의 내장 유효성 검사 오류와 구분하기 위해 직렬화 모듈을 가져와 완전히 자격이 있는 `ValidationError` 스타일을 사용해야 합니다. 예를 들어, `raise serializers.ValidationError('This field must be an integer value.')`

`ValidationError` 클래스는 시리얼라이저 및 필드 유효성 검사와 유효성 검사 클래스에서 사용해야 합니다. 또한 `raise_exception` 키워드 인수와 함께 `serializer.is_valid`를 호출할 때도 발생합니다:

    serializer.is_valid(raise_exception=True)

제네릭 뷰는 `raise_exception=True` 플래그를 사용하므로, 위에 설명된 대로 사용자 정의 예외 처리기를 사용하여 API에서 전역적으로 유효성 검사 오류 응답 스타일을 재정의할 수 있습니다.

기본적으로 이 예외는 "400 Bad Request" HTTP 상태 코드로 응답합니다.

---

# 일반 오류 뷰

Django REST 프레임워크는 일반적인 JSON `500` 서버 오류 및 `400` 잘못된 요청 응답을 제공하기에 적합한 두 가지 오류 뷰를 제공합니다. (Django의 기본 오류 뷰는 HTML 응답을 제공하며, 이는 API 전용 애플리케이션에는 적합하지 않을 수 있습니다.)

[Django의 오류 뷰 사용자 정의 문서][django-custom-error-views]에 따라 사용하십시오.

## `rest_framework.exceptions.server_error`

상태 코드 `500`과 `application/json` 콘텐츠 유형으로 응답을 반환합니다.

`handler500`으로 설정:

    handler500 = 'rest_framework.exceptions.server_error'

## `rest_framework.exceptions.bad_request`

상태 코드 `400`과 `application/json` 콘텐츠 유형으로 응답을 반환합니다.

`handler400`으로 설정:

    handler400 = 'rest_framework.exceptions.bad_request'

# 서드파티 패키지

다음 서드파티 패키지도 사용할 수 있습니다.

## DRF 표준화된 오류

[drf-standardized-errors][drf-standardized-errors] 패키지는 모든 4xx 및 5xx 응답에 대해 동일한 형식을 생성하는 예외 처리기를 제공합니다. 기본 예외 처리기를 대체할 수 있으며, 전체 예외 처리기를 다시 작성하지 않고도 오류 응답 형식을 사용자 정의할 수 있습니다. 표준화된 오류 응답 형식은 문서화하기 쉽고 API 소비자가 처리하기 쉽습니다.

[cite]: https://doughellmann.com/blog/2009/06/19/python-exception-handling-techniques/
[authentication]: authentication.md
[django-custom-error-views]: https://docs.djangoproject.com/en/dev/topics/http/views/#customizing-error-views
[drf-standardized-errors]: https://github.com/ghazi-git/drf-standardized-errors
