# 튜토리얼 6: ViewSets 및 Routers

REST 프레임워크에는 `ViewSets`를 처리하기 위한 추상화가 포함되어 있으며, 이는 개발자가 API의 상태와 상호 작용을 모델링하는 데 집중할 수 있도록 하며, 공통 규칙을 기반으로 URL 구성을 자동으로 처리할 수 있게 합니다.

`ViewSet` 클래스는 `View` 클래스와 거의 동일하지만, `get` 또는 `put`과 같은 메서드 핸들러가 아닌 `retrieve` 또는 `update`와 같은 작업을 제공합니다.

`ViewSet` 클래스는 일반적으로 URL conf를 정의하는 복잡성을 처리하는 `Router` 클래스를 사용하여 마지막 순간에 뷰 세트로 인스턴스화될 때만 메서드 핸들러 세트에 바인딩됩니다.

## ViewSets로 리팩터링

현재 뷰 세트를 가져와서 view sets로 리팩터링하겠습니다.

먼저 `UserList`와 `UserDetail` 클래스를 단일 `UserViewSet` 클래스로 리팩터링합니다. `snippets/views.py` 파일에서 두 뷰 클래스를 제거하고 단일 ViewSet 클래스로 교체할 수 있습니다:

    from rest_framework import viewsets


    class UserViewSet(viewsets.ReadOnlyModelViewSet):
        """
        이 viewset은 자동으로 `list` 및 `retrieve` 작업을 제공합니다.
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer

여기서는 `ReadOnlyModelViewSet` 클래스를 사용하여 기본 '읽기 전용' 작업을 자동으로 제공합니다. 여전히 `queryset`과 `serializer_class` 속성을 설정하고 있으며, 정규 뷰를 사용할 때와 마찬가지로 두 개의 개별 클래스에 동일한 정보를 제공할 필요가 없습니다.

다음으로 `SnippetList`, `SnippetDetail`, `SnippetHighlight` 뷰 클래스를 교체하겠습니다. 세 가지 뷰를 제거하고 다시 한 번 단일 클래스로 교체할 수 있습니다.

    from rest_framework import permissions
    from rest_framework import renderers
    from rest_framework.decorators import action
    from rest_framework.response import Response


    class SnippetViewSet(viewsets.ModelViewSet):
        """
        이 ViewSet은 자동으로 `list`, `create`, `retrieve`,
        `update` 및 `destroy` 작업을 제공합니다.

        추가로 `highlight` 작업도 제공합니다.
        """
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer
        permission_classes = [permissions.IsAuthenticatedOrReadOnly,
                              IsOwnerOrReadOnly]

        @action(detail=True, renderer_classes=[renderers.StaticHTMLRenderer])
        def highlight(self, request, *args, **kwargs):
            snippet = self.get_object()
            return Response(snippet.highlighted)

        def perform_create(self, serializer):
            serializer.save(owner=self.request.user)

이번에는 `ModelViewSet` 클래스를 사용하여 기본 읽기 및 쓰기 작업의 전체 세트를 얻었습니다.

또한 `@action` 데코레이터를 사용하여 `highlight`라는 사용자 지정 작업을 생성했습니다. 이 데코레이터는 표준 `create`/`update`/`delete` 스타일에 맞지 않는 사용자 지정 엔드포인트를 추가하는 데 사용할 수 있습니다.

기본적으로 `@action` 데코레이터를 사용하는 사용자 지정 작업은 `GET` 요청에 응답합니다. `POST` 요청에 응답하는 작업이 필요하면 `methods` 인수를 사용할 수 있습니다.

사용자 지정 작업의 URL은 기본적으로 메서드 이름 자체에 따라 달라집니다. URL을 구성하는 방식을 변경하려면 데코레이터 키워드 인수로 `url_path`를 포함할 수 있습니다.

## URL에 ViewSets를 명시적으로 바인딩

핸들러 메서드는 URLConf를 정의할 때만 작업에 바인딩됩니다.
내부에서 무슨 일이 일어나는지 확인하기 위해 먼저 ViewSets에서 일련의 뷰를 명시적으로 생성해 보겠습니다.

`snippets/urls.py` 파일에서 `ViewSet` 클래스를 일련의 구체적인 뷰로 바인딩합니다.

    from rest_framework import renderers

    from snippets.views import api_root, SnippetViewSet, UserViewSet

    snippet_list = SnippetViewSet.as_view({
        'get': 'list',
        'post': 'create'
    })
    snippet_detail = SnippetViewSet.as_view({
        'get': 'retrieve',
        'put': 'update',
        'patch': 'partial_update',
        'delete': 'destroy'
    })
    snippet_highlight = SnippetViewSet.as_view({
        'get': 'highlight'
    }, renderer_classes=[renderers.StaticHTMLRenderer])
    user_list = UserViewSet.as_view({
        'get': 'list'
    })
    user_detail = UserViewSet.as_view({
        'get': 'retrieve'
    })

여기서는 각 뷰에 대해 HTTP 메서드를 필요한 작업에 바인딩하여 각 `ViewSet` 클래스에서 여러 뷰를 생성하고 있습니다.

이제 리소스를 구체적인 뷰로 바인딩했으므로 URL conf에 일반적으로 뷰를 등록할 수 있습니다.

    urlpatterns = format_suffix_patterns([
        path('', api_root),
        path('snippets/', snippet_list, name='snippet-list'),
        path('snippets/<int:pk>/', snippet_detail, name='snippet-detail'),
        path('snippets/<int:pk>/highlight/', snippet_highlight, name='snippet-highlight'),
        path('users/', user_list, name='user-list'),
        path('users/<int:pk>/', user_detail, name='user-detail')
    ])

## Routers 사용하기

`View` 클래스 대신 `ViewSet` 클래스를 사용하므로 URL conf를 직접 설계할 필요가 없습니다. 리소스를 뷰 및 URL에 연결하는 규칙은 `Router` 클래스를 사용하여 자동으로 처리할 수 있습니다. 적절한 view sets를 라우터에 등록하고 나머지는 라우터가 처리하도록 하면 됩니다.

다음은 다시 연결된 `snippets/urls.py` 파일입니다.

    from django.urls import path, include
    from rest_framework.routers import DefaultRouter

    from snippets import views

    # 라우터를 생성하고 ViewSets를 등록합니다.
    router = DefaultRouter()
    router.register(r'snippets', views.SnippetViewSet, basename='snippet')
    router.register(r'users', views.UserViewSet, basename='user')

    # API URL은 이제 라우터에 의해 자동으로 결정됩니다.
    urlpatterns = [
        path('', include(router.urls)),
    ]

ViewSets를 라우터에 등록하는 것은 urlpattern을 제공하는 것과 유사합니다. 두 개의 인수 - 뷰에 대한 URL 접두사와 view set 자체를 포함합니다.

우리가 사용하고 있는 `DefaultRouter` 클래스는 API 루트 뷰도 자동으로 생성하므로, 이제 `views` 모듈에서 `api_root` 함수를 삭제할 수 있습니다.

## 뷰와 ViewSets 간의 트레이드 오프

ViewSets를 사용하는 것은 정말 유용한 추상화가 될 수 있습니다. 이는 API 전반에 걸쳐 URL 규칙이 일관되도록 보장하며, 작성해야 할 코드 양을 최소화하고, URL conf의 세부 사항보다는 API가 제공하는 상호 작용 및 표현에 집중할 수 있게 합니다.

하지만 항상 올바른 접근 방식은 아닙니다. 함수 기반 뷰 대신 클래스 기반 뷰를 사용할 때와 마찬가지로 고려해야 할 트레이드 오프가 있습니다. ViewSets를 사용하면 API 뷰를 개별적으로 작성하는 것보다 덜 명시적입니다.
