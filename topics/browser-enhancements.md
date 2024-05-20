# 브라우저 향상 기능

> "오버로드된 POST에 대한 두 가지 논란의 여지가 없는 사용법이 있습니다. 첫 번째는 PUT 또는 DELETE를 지원하지 않는 웹 브라우저와 같은 클라이언트를 위해 HTTP의 일관된 인터페이스를 *모방*하는 것입니다."
>
> &mdash; [RESTful Web Services][cite], Leonard Richardson & Sam Ruby.

브라우저 API가 기능할 수 있도록 하기 위해 REST 프레임워크에서 제공해야 하는 몇 가지 브라우저 향상 기능이 있습니다.

버전 3.3.0부터 이러한 기능은 [ajax-form][ajax-form] 라이브러리를 사용하여 자바스크립트로 활성화됩니다.

## 브라우저 기반의 PUT, DELETE 등...

[AJAX form 라이브러리][ajax-form]은 HTML 폼에서 브라우저 기반의 `PUT`, `DELETE` 및 기타 메서드를 지원합니다.

라이브러리를 포함한 후, 폼에서 `data-method` 속성을 사용하세요. 예를 들어:

    <form action="/" data-method="PUT">
        <input name='foo'/>
        ...
    </form>

참고로, 3.3.0 이전에는 이 지원이 자바스크립트 기반이 아닌 서버 측에서 이루어졌습니다. 요청 파싱에서 미묘한 문제를 일으키기 때문에 [Ruby on Rails][rails]에서 사용되는 메서드 오버로딩 스타일은 더 이상 지원되지 않습니다.

## 폼이 아닌 콘텐츠의 브라우저 기반 제출

JSON과 같은 콘텐츠 타입의 브라우저 기반 제출은 `data-override='content-type'` 및 `data-override='content'` 속성이 있는 폼 필드를 사용하여 [AJAX form 라이브러리][ajax-form]에서 지원됩니다.

예를 들어:

    <form action="/">
        <input data-override='content-type' value='application/json' type='hidden'/>
        <textarea data-override='content'>{}</textarea>
        <input type="submit"/>
    </form>

참고로, 3.3.0 이전에는 이 지원이 자바스크립트 기반이 아닌 서버 측에서 이루어졌습니다.

## URL 기반 형식 접미사

REST 프레임워크는 `?format=json` 스타일의 URL 매개변수를 사용할 수 있으며, 이는 뷰에서 반환해야 하는 콘텐츠 타입을 결정하는 데 유용한 단축키가 될 수 있습니다.

이 동작은 `URL_FORMAT_OVERRIDE` 설정을 사용하여 제어됩니다.

## HTTP 헤더 기반 메서드 재정의

버전 3.3.0 이전에는 요청 메서드를 재정의하기 위해 `X-HTTP-Method-Override` 반확장 헤더가 지원되었습니다. 이 동작은 더 이상 코어에 포함되지 않지만, 필요에 따라 미들웨어를 사용하여 추가할 수 있습니다.

예를 들어:

    METHOD_OVERRIDE_HEADER = 'HTTP_X_HTTP_METHOD_OVERRIDE'

    class MethodOverrideMiddleware:

        def __init__(self, get_response):
            self.get_response = get_response

        def __call__(self, request):
            if request.method == 'POST' and METHOD_OVERRIDE_HEADER in request.META:
                request.method = request.META[METHOD_OVERRIDE_HEADER]
            return self.get_response(request)

## URL 기반 수락 헤더

버전 3.3.0까지 REST 프레임워크는 `?accept=application/json` 스타일의 URL 매개변수에 대한 기본 지원을 포함하고 있었으며, 이는 `Accept` 헤더를 재정의할 수 있었습니다.

콘텐츠 협상 API 도입 이후 이 동작은 더 이상 코어에 포함되지 않지만, 필요에 따라 사용자 정의 콘텐츠 협상 클래스를 사용하여 추가할 수 있습니다.

예를 들어:

    class AcceptQueryParamOverride():
        def get_accept_list(self, request):
            header = request.META.get('HTTP_ACCEPT', '*/*')
            header = request.query_params.get('_accept', header)
            return [token.strip() for token in header.split(',')]

## HTML5는 PUT 및 DELETE 폼을 지원하지 않나요?

아닙니다. 한때 `PUT` 및 `DELETE` 폼을 지원하려 했지만, 나중에 [명세에서 제외되었습니다][html5]. `PUT` 및 `DELETE` 지원 및 폼 인코딩된 데이터 외의 콘텐츠 타입 지원에 대한 [지속적인 논의][put_delete]가 있습니다.

[cite]: https://www.amazon.com/RESTful-Web-Services-Leonard-Richardson/dp/0596529260
[ajax-form]: https://github.com/tomchristie/ajax-form
[rails]: https://guides.rubyonrails.org/form_helpers.html#how-do-forms-with-put-or-delete-methods-work
[html5]: https://www.w3.org/TR/html5-diff/#changes-2010-06-24
[put_delete]: http://amundsen.com/examples/put-delete-forms/
