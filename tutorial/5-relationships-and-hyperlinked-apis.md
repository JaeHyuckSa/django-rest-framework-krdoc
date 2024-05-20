# 튜토리얼 5: 관계 및 하이퍼링크 API

현재 API 내의 관계는 기본 키를 사용하여 나타냅니다. 이번 튜토리얼에서는 관계에 하이퍼링크를 사용하여 API의 응집력과 발견 가능성을 향상시켜 보겠습니다.

## API 루트 엔드포인트 생성

현재 'snippets'와 'users'에 대한 엔드포인트가 있지만, API에 대한 단일 진입점은 없습니다. 이를 생성하기 위해, 이전에 소개한 `@api_view` 데코레이터를 사용하여 일반 함수 기반 뷰를 사용하겠습니다. `snippets/views.py`에 다음을 추가합니다:

    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from rest_framework.reverse import reverse


    @api_view(['GET'])
    def api_root(request, format=None):
        return Response({
            'users': reverse('user-list', request=request, format=format),
            'snippets': reverse('snippet-list', request=request, format=format)
        })

여기서 두 가지를 주목해야 합니다. 첫째, 완전한 URL을 반환하기 위해 REST 프레임워크의 `reverse` 함수를 사용합니다. 둘째, URL 패턴은 나중에 `snippets/urls.py`에서 선언할 편의 이름으로 식별됩니다.

## 하이라이트된 스니펫에 대한 엔드포인트 생성

우리의 pastebin API에 아직 누락된 또 다른 중요한 것은 코드 하이라이트 엔드포인트입니다.

다른 모든 API 엔드포인트와 달리, JSON을 사용하지 않고 HTML 표현을 제공하고자 합니다. REST 프레임워크에는 템플릿을 사용하여 렌더링된 HTML을 처리하는 것과 사전 렌더링된 HTML을 처리하는 두 가지 스타일의 HTML 렌더러가 제공됩니다. 이 엔드포인트에는 두 번째 렌더러를 사용하고자 합니다.

코드 하이라이트 뷰를 생성할 때 고려해야 할 다른 점은 사용할 수 있는 기존의 구체적인 제네릭 뷰가 없다는 것입니다. 객체 인스턴스를 반환하는 대신 객체 인스턴스의 속성을 반환합니다.

구체적인 제네릭 뷰를 사용하는 대신 인스턴스를 나타내는 기본 클래스를 사용하고, 자체 `.get()` 메서드를 생성합니다. `snippets/views.py`에 다음을 추가합니다:

    from rest_framework import renderers

    class SnippetHighlight(generics.GenericAPIView):
        queryset = Snippet.objects.all()
        renderer_classes = [renderers.StaticHTMLRenderer]

        def get(self, request, *args, **kwargs):
            snippet = self.get_object()
            return Response(snippet.highlighted)

항상 그렇듯이 새로 생성한 뷰를 URL 구성에 추가해야 합니다.
`snippets/urls.py`에서 새 API 루트에 대한 URL 패턴을 추가합니다:

    path('', views.api_root),

그리고 스니펫 하이라이트에 대한 URL 패턴을 추가합니다:

    path('snippets/<int:pk>/highlight/', views.SnippetHighlight.as_view()),

## API에 하이퍼링크 추가

엔터티 간의 관계를 처리하는 것은 웹 API 설계의 더 도전적인 측면 중 하나입니다. 관계를 나타내는 방법에는 여러 가지가 있습니다:

* 기본 키 사용.
* 엔터티 간 하이퍼링크 사용.
* 관련 엔터티의 고유 식별 슬러그 필드 사용.
* 관련 엔터티의 기본 문자열 표현 사용.
* 부모 표현 내에 관련 엔터티 중첩.
* 기타 사용자 지정 표현.

REST 프레임워크는 이러한 모든 스타일을 지원하며, 순방향 또는 역방향 관계 또는 일반 외래 키와 같은 사용자 지정 매니저를 통해 이를 적용할 수 있습니다.

이 경우 엔터티 간 하이퍼링크 스타일을 사용하고자 합니다. 이를 위해 기존 `ModelSerializer` 대신 `HyperlinkedModelSerializer`를 확장하도록 직렬화기를 수정하겠습니다.

`HyperlinkedModelSerializer`는 `ModelSerializer`와 다음과 같은 차이점이 있습니다:

* 기본적으로 `id` 필드를 포함하지 않습니다.
* `HyperlinkedIdentityField`를 사용하여 `url` 필드를 포함합니다.
* 관계는 `PrimaryKeyRelatedField` 대신 `HyperlinkedRelatedField`를 사용합니다.

기존 직렬화기를 쉽게 재작성하여 하이퍼링크를 사용할 수 있습니다. `snippets/serializers.py`에 다음을 추가합니다:

    class SnippetSerializer(serializers.HyperlinkedModelSerializer):
        owner = serializers.ReadOnlyField(source='owner.username')
        highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

        class Meta:
            model = Snippet
            fields = ['url', 'id', 'highlight', 'owner',
                      'title', 'code', 'linenos', 'language', 'style']


    class UserSerializer(serializers.HyperlinkedModelSerializer):
        snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

        class Meta:
            model = User
            fields = ['url', 'id', 'username', 'snippets']

새로운 `'highlight'` 필드를 추가한 것도 주목하세요. 이 필드는 `url` 필드와 동일한 유형이지만, `'snippet-detail'` URL 패턴 대신 `'snippet-highlight'` URL 패턴을 가리킵니다.

`.json`과 같은 형식 접미사 URL을 포함했기 때문에, `highlight` 필드가 반환하는 모든 형식 접미사 하이퍼링크가 `'.html'` 접미사를 사용해야 함을 나타내야 합니다.

## URL 패턴에 이름 지정하기

하이퍼링크 API를 사용할 경우, URL 패턴에 이름을 지정해야 합니다. 이름을 지정해야 하는 URL 패턴을 살펴보겠습니다.

* API 루트는 `'user-list'`와 `'snippet-list'`를 참조합니다.
* 스니펫 직렬화기에는 `'snippet-highlight'`를 참조하는 필드가 포함됩니다.
* 사용자 직렬화기에는 `'snippet-detail'`을 참조하는 필드가 포함됩니다.
* 스니펫과 사용자 직렬화기에는 기본적으로 `'{model_name}-detail'`을 참조하는 `'url'` 필드가 포함됩니다. 이 경우 `'snippet-detail'`과 `'user-detail'`이 됩니다.

URL 구성에 이 모든 이름을 추가한 후, 최종 `snippets/urls.py` 파일은 다음과 같아야 합니다:

    from django.urls import path
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    # API 엔드포인트
    urlpatterns = format_suffix_patterns([
        path('', views.api_root),
        path('snippets/',
            views.SnippetList.as_view(),
            name='snippet-list'),
        path('snippets/<int:pk>/',
            views.SnippetDetail.as_view(),
            name='snippet-detail'),
        path('snippets/<int:pk>/highlight/',
            views.SnippetHighlight.as_view(),
            name='snippet-highlight'),
        path('users/',
            views.UserList.as_view(),
            name='user-list'),
        path('users/<int:pk>/',
            views.UserDetail.as_view(),
            name='user-detail')
    ])

## 페이지네이션 추가

사용자 및 코드 스니펫에 대한 목록 뷰가 많은 인스턴스를 반환할 수 있으므로, 결과를 페이지네이션하여 API 클라이언트가 각 개별 페이지를 통해 이동할 수 있도록 하고 싶습니다.

`tutorial/settings.py` 파일을 약간 수정하여 기본 목록 스타일을 페이지네이션으로 변경할 수 있습니다. 다음 설정을 추가합니다:

    REST_FRAMEWORK = {
        'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
        'PAGE_SIZE': 10
    }

REST 프레임워크의 설정은 모두 `REST_FRAMEWORK`라는 단일 사전 설정에 네임스페이스화되어 있어 다른 프로젝트 설정과 잘 분리됩니다.

페이지네이션 스타일을 사용자 지정할 수도 있지만, 이 경우 기본 스타일을 유지하겠습니다.

## API 탐색

브라우저를 열고 브라우저 가능한 API로 이동하면, 링크를 따라가는 것만으로도 API를 탐색할 수 있음을 알 수 있습니다.

또한 스니펫 인스턴스에서 'highlight' 링크를 확인할 수 있으며, 이는 하이라이트된 코드 HTML 표현으로 이동합니다.

[튜토리얼 6부][tut-6]에서는 ViewSets와 Routers를 사용하여 API를 구축하는 데 필요한 코드 양을 줄이는 방법을 살펴보겠습니다.

[tut-6]: 6-viewsets-and-routers.md
