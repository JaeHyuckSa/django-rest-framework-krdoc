# 튜토리얼 2: 요청과 응답

이제부터 REST 프레임워크의 핵심을 다루기 시작합니다.
몇 가지 필수 구성 요소를 소개하겠습니다.

## 요청 객체

REST 프레임워크는 일반 `HttpRequest`를 확장한 `Request` 객체를 도입하여 더 유연한 요청 파싱을 제공합니다. `Request` 객체의 핵심 기능은 `request.data` 속성으로, 이는 `request.POST`와 유사하지만, 웹 API 작업에 더 유용합니다.

    request.POST  # 폼 데이터만 처리합니다. 'POST' 메서드에서만 작동합니다.
    request.data  # 임의의 데이터를 처리합니다. 'POST', 'PUT', 'PATCH' 메서드에서 작동합니다.

## 응답 객체

REST 프레임워크는 `Response` 객체도 도입합니다. 이는 렌더링되지 않은 콘텐츠를 받아 클라이언트에게 반환할 올바른 콘텐츠 유형을 결정하는 콘텐츠 협상을 사용하는 `TemplateResponse` 유형입니다.

    return Response(data)  # 클라이언트가 요청한 콘텐츠 유형으로 렌더링합니다.

## 상태 코드

뷰에서 숫자 HTTP 상태 코드를 사용하는 것은 항상 명확하지 않으며, 오류 코드를 잘못 사용하는 경우를 놓치기 쉽습니다. REST 프레임워크는 `status` 모듈에서 `HTTP_400_BAD_REQUEST`와 같은 각 상태 코드에 대한 더 명확한 식별자를 제공합니다. 숫자 식별자 대신 이를 사용하는 것이 좋습니다.

## API 뷰 래핑

REST 프레임워크는 API 뷰를 작성할 때 사용할 수 있는 두 가지 래퍼를 제공합니다.

1. 함수 기반 뷰에서 사용할 수 있는 `@api_view` 데코레이터.
2. 클래스 기반 뷰에서 사용할 수 있는 `APIView` 클래스.

이 래퍼들은 뷰에서 `Request` 인스턴스를 수신하고, 콘텐츠 협상이 수행될 수 있도록 `Response` 객체에 컨텍스트를 추가하는 등의 기능을 제공합니다.

래퍼들은 적절한 경우 `405 Method Not Allowed` 응답을 반환하고, 잘못된 입력으로 `request.data`에 접근할 때 발생하는 `ParseError` 예외를 처리하는 등의 동작도 제공합니다.

## 모든 것 통합하기

좋습니다. 이제 이러한 새로운 구성 요소를 사용하여 뷰를 약간 리팩터링해보겠습니다.

    from rest_framework import status
    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer


    @api_view(['GET', 'POST'])
    def snippet_list(request):
        """
        모든 코드 스니펫을 나열하거나 새 스니펫을 생성합니다.
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        elif request.method == 'POST':
            serializer = SnippetSerializer(data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

우리의 인스턴스 뷰는 이전 예제에 비해 개선되었습니다. 코드가 더 간결해졌으며, 이제 폼 API를 사용하는 것처럼 느껴집니다. 또한 명명된 상태 코드를 사용하여 응답의 의미를 더 명확하게 했습니다.

다음은 `views.py` 모듈에 있는 개별 스니펫에 대한 뷰입니다.

    @api_view(['GET', 'PUT', 'DELETE'])
    def snippet_detail(request, pk):
        """
        코드 스니펫을 조회, 업데이트 또는 삭제합니다.
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return Response(status=status.HTTP_404_NOT_FOUND)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        elif request.method == 'PUT':
            serializer = SnippetSerializer(snippet, data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        elif request.method == 'DELETE':
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)

이 모든 것이 매우 익숙하게 느껴질 것입니다 - 일반 Django 뷰를 사용하는 것과 크게 다르지 않습니다.

더 이상 요청이나 응답을 특정 콘텐츠 유형에 명시적으로 연결하지 않는다는 점을 주목하세요. `request.data`는 들어오는 `json` 요청을 처리할 수 있지만, 다른 형식도 처리할 수 있습니다. 마찬가지로 데이터를 포함한 응답 객체를 반환하지만, REST 프레임워크가 응답을 올바른 콘텐츠 유형으로 렌더링하도록 합니다.

## URL에 선택적 형식 접미사 추가

응답이 더 이상 단일 콘텐츠 유형에 고정되지 않도록 하기 위해 API 엔드포인트에 형식 접미사를 추가하는 것이 좋습니다. 형식 접미사를 사용하면 특정 형식을 명시적으로 참조하는 URL을 제공하므로, [http://example.com/api/items/4.json][json-url]과 같은 URL을 처리할 수 있습니다.

먼저 두 뷰에 `format` 키워드 인수를 추가합니다.

    def snippet_list(request, format=None):

그리고

    def snippet_detail(request, pk, format=None):

이제 `snippets/urls.py` 파일을 약간 업데이트하여 기존 URL에 `format_suffix_patterns`를 추가합니다.

    from django.urls import path
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = [
        path('snippets/', views.snippet_list),
        path('snippets/<int:pk>/', views.snippet_detail),
    ]

    urlpatterns = format_suffix_patterns(urlpatterns)

이 추가 URL 패턴을 반드시 추가할 필요는 없지만, 특정 형식을 참조하는 간단하고 깔끔한 방법을 제공합니다.

## 어떻게 보이나요?

커맨드 라인에서 [튜토리얼 1부][tut-1]에서 한 것처럼 API를 테스트해보세요. 모든 것이 비슷하게 작동하지만, 잘못된 요청을 보낼 때 더 나은 오류 처리가 있습니다.

이전과 같이 모든 스니펫 목록을 가져올 수 있습니다.

    http http://127.0.0.1:8000/snippets/

    HTTP/1.1 200 OK
    ...
    [
      {
        "id": 1,
        "title": "",
        "code": "foo = \"bar\"\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
      },
      {
        "id": 2,
        "title": "",
        "code": "print(\"hello, world\")\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
      }
    ]

`Accept` 헤더를 사용하여 응답 형식을 제어할 수 있습니다:

    http http://127.0.0.1:8000/snippets/ Accept:application/json  # JSON 요청
    http http://127.0.0.1:8000/snippets/ Accept:text/html         # HTML 요청

또는 형식 접미사를 추가하여 제어할 수 있습니다:

    http http://127.0.0.1:8000/snippets.json  # JSON 접미사
    http http://127.0.0.1:8000/snippets.api   # 브라우저 API 접미사

마찬가지로, `Content-Type` 헤더를 사용하여 전송하는 요청의 형식을 제어할 수 있습니다.

    # 폼 데이터를 사용하여 POST
    http --form POST http://127.0.0.1:8000/snippets/ code="print(123)"

    {
      "id": 3,
      "title": "",
      "code": "print(123)",
      "linenos": false,
      "language": "python",
      "style": "friendly"
    }

    # JSON을 사용하여 POST
    http --json POST http://127.0.0.1:8000/snippets/ code="print(456)"

    {
        "id": 4,
        "title": "",
        "code": "print(456)",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

위의 `http` 요청에 `--debug` 스위치를 추가하면 요청 헤더에서 요청 유형을 확인할 수 있습니다.

이제 [http://127.0.0.1:8000/snippets/][devserver]를 방문하여 웹 브라우저에서 API를 엽니다.

### 브라우저 가능성

API는 클라이언트 요청을 기반으로 응답의 콘텐츠 유형을 선택하므로, 기본적으로 웹 브라우저에서 요청된 리소스의 HTML 형식 표현을 반환합니다. 이를 통해 API는 완전히 웹 브라우저 가능한 HTML 표현을 반환할 수 있습니다.

웹 브라우저 가능한 API를 갖는 것은 큰 사용성 향상을 제공하며, API 개발 및 사용을 훨씬 더 쉽게 만듭니다. 또한 다른 개발자가 API를 검사하고 작업하는 데 필요한 진입 장벽을 크게 낮춥니다.

브라우저 API 기능 및 이를 사용자 정의하는 방법에 대한 자세한 내용은 [브라우저 API][browsable-api] 항목을 참조하세요.

## 다음은 무엇인가요?

[튜토리얼 3부][tut-3]에서는 클래스 기반 뷰를 사용하기 시작하고, 제네릭 뷰가 우리가 작성해야 할 코드의 양을 어떻게 줄이는지 살펴보겠습니다.

[json-url]: http://example.com/api/items/4.json
[devserver]: http://127.0.0.1:8000/snippets/
[browsable-api]: ../topics/browsable-api.md
[tut-1]: 1-serialization.md
[tut-3]: 3-class-based-views.md
