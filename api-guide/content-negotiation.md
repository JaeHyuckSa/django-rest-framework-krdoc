---
source:
    - negotiation.py
---

# 콘텐츠 협상

> HTTP는 여러 가지 "콘텐츠 협상" 메커니즘을 제공하여 여러 표현이 가능한 경우 주어진 응답에 가장 적합한 표현을 선택하는 과정을 지원합니다.
>
> &mdash; [RFC 2616][cite], Fielding et al.

[cite]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec12.html

콘텐츠 협상은 클라이언트 또는 서버의 선호도에 따라 여러 가능한 표현 중 하나를 선택하여 클라이언트에게 반환하는 과정입니다.

## 수락된 렌더러 결정

REST 프레임워크는 사용 가능한 렌더러, 각 렌더러의 우선 순위 및 클라이언트의 `Accept:` 헤더를 기반으로 클라이언트에 반환할 미디어 유형을 결정하는 간단한 스타일의 콘텐츠 협상을 사용합니다. 이 스타일은 부분적으로 클라이언트 주도적이며, 부분적으로 서버 주도적입니다.

1. 더 구체적인 미디어 유형이 덜 구체적인 미디어 유형보다 우선합니다.
2. 여러 미디어 유형이 동일한 구체성을 가진 경우, 주어진 뷰에 대해 구성된 렌더러의 순서를 기반으로 우선 순위가 결정됩니다.

예를 들어, 다음과 같은 `Accept` 헤더가 주어진 경우:

    application/json; indent=4, application/json, application/yaml, text/html, */*

각 미디어 유형의 우선 순위는 다음과 같습니다:

* `application/json; indent=4`
* `application/json`, `application/yaml` 및 `text/html`
* `*/*`

요청된 뷰가 `YAML` 및 `HTML` 렌더러만으로 구성된 경우, REST 프레임워크는 `renderer_classes` 목록 또는 `DEFAULT_RENDERER_CLASSES` 설정에서 첫 번째로 나열된 렌더러를 선택합니다.

`HTTP Accept` 헤더에 대한 자세한 내용은 [RFC 2616][accept-header]을 참조하십시오.

---

**참고:** REST 프레임워크는 우선 순위를 결정할 때 "q" 값을 고려하지 않습니다. "q" 값을 사용하면 캐싱에 부정적인 영향을 미치며, 저자의 의견으로는 콘텐츠 협상에 불필요하고 과도하게 복잡한 접근 방식입니다.

이 접근 방식은 HTTP 사양이 서버 기반 선호도와 클라이언트 기반 선호도를 어떻게 가중치로 두어야 하는지를 의도적으로 명시하지 않기 때문에 유효한 접근 방식입니다.

---

# 사용자 정의 콘텐츠 협상

REST 프레임워크에 사용자 정의 콘텐츠 협상 스키마를 제공할 필요는 거의 없지만, 필요하다면 가능합니다. 사용자 정의 콘텐츠 협상 스키마를 구현하려면 `BaseContentNegotiation`을 재정의하십시오.

REST 프레임워크의 콘텐츠 협상 클래스는 요청에 대한 적절한 파서를 선택하고, 응답에 대한 적절한 렌더러를 선택하는 역할을 하므로, `.select_parser(request, parsers)` 및 `.select_renderer(request, renderers, format_suffix)` 메서드를 모두 구현해야 합니다.

`select_parser()` 메서드는 사용 가능한 파서 목록에서 하나의 파서 인스턴스를 반환하거나, 요청을 처리할 수 있는 파서가 없는 경우 `None`을 반환해야 합니다.

`select_renderer()` 메서드는 (렌더러 인스턴스, 미디어 유형)의 두 요소 튜플을 반환하거나 `NotAcceptable` 예외를 발생시켜야 합니다.

## 예시

다음은 적절한 파서 또는 렌더러를 선택할 때 클라이언트 요청을 무시하는 사용자 정의 콘텐츠 협상 클래스입니다.

    from rest_framework.negotiation import BaseContentNegotiation

    class IgnoreClientContentNegotiation(BaseContentNegotiation):
        def select_parser(self, request, parsers):
            """
            `.parser_classes` 목록에서 첫 번째 파서를 선택합니다.
            """
            return parsers[0]

        def select_renderer(self, request, renderers, format_suffix):
            """
            `.renderer_classes` 목록에서 첫 번째 렌더러를 선택합니다.
            """
            return (renderers[0], renderers[0].media_type)

## 콘텐츠 협상 설정

기본 콘텐츠 협상 클래스는 `DEFAULT_CONTENT_NEGOTIATION_CLASS` 설정을 사용하여 전역적으로 설정할 수 있습니다. 예를 들어, 다음 설정은 예시의 `IgnoreClientContentNegotiation` 클래스를 사용합니다.

    REST_FRAMEWORK = {
        'DEFAULT_CONTENT_NEGOTIATION_CLASS': 'myapp.negotiation.IgnoreClientContentNegotiation',
    }

또한 `APIView` 클래스 기반 뷰를 사용하여 개별 뷰 또는 뷰셋에 사용할 콘텐츠 협상을 설정할 수도 있습니다.

    from myapp.negotiation import IgnoreClientContentNegotiation
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class NoNegotiationView(APIView):
        """
        콘텐츠 협상을 수행하지 않는 예시 뷰입니다.
        """
        content_negotiation_class = IgnoreClientContentNegotiation

        def get(self, request, format=None):
            return Response({
                'accepted media type': request.accepted_renderer.media_type
            })

[accept-header]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html
