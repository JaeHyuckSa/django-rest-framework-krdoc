# API 문서화

> REST API는 리소스를 표현하고 응용 프로그램 상태를 구동하는 데 사용되는 미디어 타입을 정의하는 데 거의 모든 설명 노력을 기울여야 합니다.
>
> &mdash; Roy Fielding, [REST APIs must be hypertext driven][cite]

REST 프레임워크는 API를 문서화하기 위한 다양한 선택지를 제공합니다. 다음은 가장 인기 있는 일부를 포함한 비포괄적인 목록입니다.

## OpenAPI 지원을 위한 서드 파티 패키지

### drf-spectacular

[drf-spectacular][drf-spectacular]는 확장성, 커스터마이징 및 클라이언트 생성을 중점으로 한 [OpenAPI 3][open-api] 스키마 생성 라이브러리입니다. OpenAPI 스키마를 생성하고 표시하는 권장 방법입니다.

이 라이브러리는 가능한 한 많은 스키마 정보를 추출하는 동시에, 쉬운 커스터마이징을 위한 데코레이터와 확장을 제공합니다. [swagger-codegen][swagger], [SwaggerUI][swagger-ui] 및 [Redoc][redoc], 국제화, 버전 관리, 인증, 다형성(동적 요청 및 응답), 쿼리/경로/헤더 매개변수, 문서화 등을 명시적으로 지원합니다. 또한 여러 인기 있는 DRF 플러그인을 기본적으로 지원합니다.

### drf-yasg

[drf-yasg][drf-yasg]는 Django Rest Framework에서 제공하는 스키마 생성기를 사용하지 않고 구현된 [Swagger / OpenAPI 2][swagger] 생성 도구입니다.

가능한 한 많은 [OpenAPI 2][open-api] 사양을 구현하는 것을 목표로 합니다 - 중첩된 스키마, 명명된 모델, 응답 본문, 열거형/패턴/최소/최대 유효성 검사기, 폼 매개변수 등 - 및 `swagger-codegen`과 같은 코드 생성 도구에서 사용할 수 있는 문서를 생성합니다.

또한 매우 유용한 인터랙티브 문서 뷰어를 `swagger-ui` 형태로 제공합니다:

![스크린샷 - drf-yasg][image-drf-yasg]

---

## 내장된 OpenAPI 스키마 생성 (폐기 예정)

**폐기 공지: REST 프레임워크의 내장된 OpenAPI 스키마 생성 지원은 대신 이 기능을 제공할 수 있는 서드 파티 패키지로 대체되기 위해 폐기됩니다. 대체로 [drf-spectacular](#drf-spectacular) 패키지 사용을 권장합니다.**

OpenAPI 스키마에서 HTML 문서 페이지를 생성할 수 있는 여러 패키지가 있습니다.

두 가지 인기 있는 옵션은 [Swagger UI][swagger-ui]와 [ReDoc][redoc]입니다.

두 가지 모두 정적 스키마 파일 또는 동적 `SchemaView` 엔드포인트의 위치만 제공하면 됩니다.

### Swagger UI를 사용하는 최소 예제

스키마 문서의 예제를 따라 동적 `SchemaView`를 라우팅했다고 가정하면, Swagger UI를 사용하기 위한 최소 Django 템플릿은 다음과 같습니다:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Swagger</title>
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="stylesheet" type="text/css" href="//unpkg.com/swagger-ui-dist@3/swagger-ui.css" />
  </head>
  <body>
    <div id="swagger-ui"></div>
    <script src="//unpkg.com/swagger-ui-dist@3/swagger-ui-bundle.js"></script>
    <script>
    const ui = SwaggerUIBundle({
        url: "{% url schema_url %}",
        dom_id: '#swagger-ui',
        presets: [
          SwaggerUIBundle.presets.apis,
          SwaggerUIBundle.SwaggerUIStandalonePreset
        ],
        layout: "BaseLayout",
        requestInterceptor: (request) => {
          request.headers['X-CSRFToken'] = "{{ csrf_token }}"
          return request;
        }
      })
    </script>
  </body>
</html>
```

이것을 `swagger-ui.html`이라는 이름으로 템플릿 폴더에 저장하세요. 그런 다음 프로젝트의 URL 설정에 `TemplateView`를 라우트하세요:

```python
from django.views.generic import TemplateView

urlpatterns = [
    # ...
    # Swagger UI 템플릿을 제공하기 위해 TemplateView를 라우트합니다.
    #   * `SchemaView`의 뷰 이름을 포함한 `extra_context`를 제공합니다.
    path(
        "swagger-ui/",
        TemplateView.as_view(
            template_name="swagger-ui.html",
            extra_context={"schema_url": "openapi-schema"},
        ),
        name="swagger-ui",
    ),
]
```

[Swagger UI 문서][swagger-ui]에서 고급 사용법을 참조하세요.

### ReDoc의 최소 예제.

동적 `SchemaView`를 라우팅하기 위한 스키마 문서의 예제를 따랐다고 가정하면, ReDoc을 사용하는 최소한의 Django 템플릿은 다음과 같습니다:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>ReDoc</title>
    <!-- 적응형 디자인에 필요 -->
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link href="https://fonts.googleapis.com/css?family=Montserrat:300,400,700|Roboto:300,400,700" rel="stylesheet">
    <!-- ReDoc는 외부 페이지 스타일을 변경하지 않습니다 -->
    <style>
      body {
        margin: 0;
        padding: 0;
      }
    </style>
  </head>
  <body>
    <redoc spec-url='{% url schema_url %}'></redoc>
    <script src="https://cdn.jsdelivr.net/npm/redoc@next/bundles/redoc.standalone.js"> </script>
  </body>
</html>
```

이 파일을 템플릿 폴더에 `redoc.html`로 저장합니다. 그런 다음 프로젝트의 URL 구성에서 `TemplateView`를 라우팅합니다:

```python
from django.views.generic import TemplateView

urlpatterns = [
    # ...
    # ReDoc 템플릿을 제공하기 위해 TemplateView를 라우트합니다.
    #   * `SchemaView`의 뷰 이름과 함께 `extra_context`를 제공합니다.
    path(
        "redoc/",
        TemplateView.as_view(
            template_name="redoc.html", extra_context={"schema_url": "openapi-schema"}
        ),
        name="redoc",
    ),
]

```

[ReDoc 문서][redoc]에서 고급 사용법을 확인하세요.

## 자체 설명 API

REST 프레임워크가 제공하는 브라우저 API를 통해 API가 완전히 자체 설명할 수 있습니다. 각 API 엔드포인트에 대한 문서는 브라우저에서 URL을 방문하여 간단히 확인할 수 있습니다.

![자체 설명 API 스크린샷][image-self-describing-api]

---

#### 제목 설정

브라우저 API에서 사용되는 제목은 뷰 클래스 이름 또는 함수 이름에서 생성됩니다. 마지막 `View` 또는 `ViewSet` 접미사는 제거되며, 문자열은 대문자/소문자 경계 또는 밑줄에서 공백으로 구분됩니다.

예를 들어, `UserListView` 뷰는 브라우저 API에서 `User List`로 표시됩니다.

뷰셋을 사용할 때는 각 생성된 뷰에 적절한 접미사가 추가됩니다. 예를 들어, `UserViewSet` 뷰셋은 `User List`와 `User Instance`라는 뷰를 생성합니다.

#### 설명 설정

브라우저 API의 설명은 뷰 또는 뷰셋의 도크스트링에서 생성됩니다.

파이썬 `Markdown` 라이브러리가 설치된 경우 도크스트링에서 [마크다운 구문][markdown]을 사용할 수 있으며, 이는 브라우저 API에서 HTML로 변환됩니다. 예를 들어:

    class AccountListView(views.APIView):
        """
        시스템의 모든 **활성화된** 계정 목록을 반환합니다.

        계정이 활성화되는 방법에 대한 자세한 내용은 [여기][ref]를 참조하십시오.

        [ref]: http://example.com/activating-accounts
        """

뷰셋을 사용할 때는 기본 도크스트링이 생성된 모든 뷰에 사용됩니다. 목록 및 검색 뷰와 같은 각 뷰에 대한 설명을 제공하려면 [스키마를 문서로 사용: 예제][schemas-examples]에서 설명한 대로 도크스트링 섹션을 사용하십시오.

#### `OPTIONS` 메서드

REST 프레임워크 API는 `OPTIONS` HTTP 메서드를 사용하여 프로그래밍 방식으로 액세스할 수 있는 설명도 지원합니다. 뷰는 `OPTIONS` 요청에 대해 이름, 설명 및 수락하고 응답하는 다양한 미디어 유형을 포함한 메타데이터로 응답합니다.

제네릭 뷰를 사용하는 경우, `OPTIONS` 요청은 사용 가능한 모든 `POST` 또는 `PUT` 작업에 대한 메타데이터와 직렬화기에서 사용할 수 있는 필드에 대한 설명도 응답합니다.

`OPTIONS` 요청에 대한 응답 동작을 수정하려면 `options` 뷰 메서드를 재정의하거나 사용자 지정 메타데이터 클래스를 제공하십시오. 예를 들어:

    def options(self, request, *args, **kwargs):
        """
        OPTIONS 응답에서 뷰 설명을 포함하지 않습니다.
        """
        meta = self.metadata_class()
        data = meta.determine_metadata(request, self)
        data.pop('description')
        return Response(data=data, status=status.HTTP_200_OK)

자세한 내용은 [메타데이터 문서][metadata-docs]를 참조하십시오.

---

## 하이퍼미디어 접근 방식

완전한 RESTful API는 보낸 응답에 하이퍼미디어 컨트롤을 사용하여 사용할 수 있는 작업을 제시해야 합니다.

이 접근 방식에서는 사용할 수 있는 API 엔드포인트를 미리 문서화하는 대신 *미디어 유형*에 집중하여 설명합니다. 특정 URL에서 사용할 수 있는 작업은 엄격하게 고정된 것이 아니라 반환된 문서에 있는 링크 및 폼 컨트롤에 의해 제공됩니다.

하이퍼미디어 API를 구현하려면 API에 적합한 미디어 유형을 결정하고 해당 미디어 유형에 대한 사용자 지정 렌더러 및 파서를 구현해야 합니다. 문서의 [REST, Hypermedia & HATEOAS][hypermedia-docs] 섹션에는 배경 읽기에 대한 포인터와 다양한 하이퍼미디어 형식에 대한 링크가 포함되어 있습니다.

[cite]: https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven

[hypermedia-docs]: rest-hypermedia-hateoas.md
[metadata-docs]: ../api-guide/metadata.md
[schemas-examples]: ../api-guide/schemas.md#examples

[image-drf-yasg]: ../img/drf-yasg.png
[image-self-describing-api]: ../img/self-describing.png

[drf-yasg]: https://github.com/axnsan12/drf-yasg/
[drf-spectacular]: https://github.com/tfranzel/drf-spectacular/
[markdown]: https://daringfireball.net/projects/markdown/syntax
[open-api]: https://openapis.org/
[redoc]: https://github.com/Rebilly/ReDoc
[swagger]: https://swagger.io/
[swagger-ui]: https://swagger.io/tools/swagger-ui/
