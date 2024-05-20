# 튜토리얼 3: 클래스 기반 뷰

API 뷰를 함수 기반 뷰 대신 클래스 기반 뷰를 사용하여 작성할 수도 있습니다. 이 패턴은 공통 기능을 재사용할 수 있게 하며, 코드를 [DRY][dry]하게 유지하는 데 도움이 되는 강력한 패턴입니다.

## 클래스 기반 뷰를 사용하여 API 재작성하기

루트 뷰를 클래스 기반 뷰로 다시 작성하는 것부터 시작하겠습니다. 이를 위해 `views.py`를 약간 리팩터링합니다.

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from django.http import Http404
    from rest_framework.views import APIView
    from rest_framework.response import Response
    from rest_framework import status


    class SnippetList(APIView):
        """
        모든 스니펫을 나열하거나 새 스니펫을 생성합니다.
        """
        def get(self, request, format=None):
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        def post(self, request, format=None):
            serializer = SnippetSerializer(data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

지금까지는 잘 진행되고 있습니다. 이전 경우와 매우 유사하게 보이지만, 다양한 HTTP 메서드 간의 분리가 더 잘 되어 있습니다. `views.py`에서 인스턴스 뷰도 업데이트해야 합니다.

    class SnippetDetail(APIView):
        """
        스니펫 인스턴스를 조회, 업데이트 또는 삭제합니다.
        """
        def get_object(self, pk):
            try:
                return Snippet.objects.get(pk=pk)
            except Snippet.DoesNotExist:
                raise Http404

        def get(self, request, pk, format=None):
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        def put(self, request, pk, format=None):
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet, data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        def delete(self, request, pk, format=None):
            snippet = self.get_object(pk)
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)

좋습니다. 다시 한 번, 현재는 함수 기반 뷰와 매우 유사하게 보입니다.

이제 클래스 기반 뷰를 사용하고 있으므로 `snippets/urls.py`를 약간 리팩터링해야 합니다.

    from django.urls import path
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = [
        path('snippets/', views.SnippetList.as_view()),
        path('snippets/<int:pk>/', views.SnippetDetail.as_view()),
    ]

    urlpatterns = format_suffix_patterns(urlpatterns)

좋습니다. 개발 서버를 실행하면 모든 것이 이전과 동일하게 작동해야 합니다.

## 믹스인 사용하기

클래스 기반 뷰를 사용하면 재사용 가능한 동작을 쉽게 구성할 수 있다는 큰 장점이 있습니다.

지금까지 사용한 생성/조회/업데이트/삭제 작업은 우리가 만드는 모든 모델 기반 API 뷰에 대해 매우 유사할 것입니다. 이러한 공통 동작은 REST 프레임워크의 믹스인 클래스에 구현되어 있습니다.

믹스인 클래스를 사용하여 뷰를 구성하는 방법을 살펴보겠습니다. `views.py` 모듈을 다시 살펴보겠습니다.

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import mixins
    from rest_framework import generics

    class SnippetList(mixins.ListModelMixin,
                      mixins.CreateModelMixin,
                      generics.GenericAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.list(request, *args, **kwargs)

        def post(self, request, *args, **kwargs):
            return self.create(request, *args, **kwargs)

여기서 정확히 무슨 일이 일어나고 있는지 살펴보겠습니다. 우리는 `GenericAPIView`를 사용하여 뷰를 구축하고 있으며, `ListModelMixin`과 `CreateModelMixin`을 추가하고 있습니다.

기본 클래스는 핵심 기능을 제공하며, 믹스인 클래스는 `.list()` 및 `.create()` 동작을 제공합니다. 그런 다음 `get` 및 `post` 메서드를 적절한 동작에 명시적으로 바인딩하고 있습니다. 지금까지는 간단합니다.

    class SnippetDetail(mixins.RetrieveModelMixin,
                        mixins.UpdateModelMixin,
                        mixins.DestroyModelMixin,
                        generics.GenericAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.retrieve(request, *args, **kwargs)

        def put(self, request, *args, **kwargs):
            return self.update(request, *args, **kwargs)

        def delete(self, request, *args, **kwargs):
            return self.destroy(request, *args, **kwargs)

매우 유사합니다. 여기서도 `GenericAPIView` 클래스를 사용하여 핵심 기능을 제공하고, 믹스인을 추가하여 `.retrieve()`, `.update()` 및 `.destroy()` 동작을 제공합니다.

## 제네릭 클래스 기반 뷰 사용하기

믹스인 클래스를 사용하여 이전보다 약간 적은 코드로 뷰를 다시 작성했지만, 한 단계 더 나아갈 수 있습니다. REST 프레임워크는 `views.py` 모듈을 더욱 줄일 수 있는 이미 혼합된 제네릭 뷰 세트를 제공합니다.

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import generics


    class SnippetList(generics.ListCreateAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer


    class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

와우, 매우 간결합니다. 무료로 많은 것을 얻었으며, 우리의 코드는 깔끔하고, 명료하며, 관용적인 Django처럼 보입니다.

다음으로 [튜토리얼 4부][tut-4]로 넘어가서 API의 인증 및 권한을 처리하는 방법을 살펴보겠습니다.

[dry]: https://en.wikipedia.org/wiki/Don't_repeat_yourself
[tut-4]: 4-authentication-and-permissions.md
