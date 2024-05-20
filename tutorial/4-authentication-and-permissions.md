# 튜토리얼 4: 인증 및 권한

현재 우리 API는 코드 스니펫을 편집하거나 삭제하는 것에 대해 아무런 제한이 없습니다. 다음과 같은 고급 동작을 추가하고 싶습니다:

* 코드 스니펫은 항상 생성자와 연결되어야 합니다.
* 인증된 사용자만 스니펫을 생성할 수 있습니다.
* 스니펫의 생성자만 해당 스니펫을 업데이트하거나 삭제할 수 있습니다.
* 인증되지 않은 요청은 전체 읽기 전용 액세스 권한을 가져야 합니다.

## 모델에 정보 추가하기

`Snippet` 모델 클래스에 몇 가지 변경을 가할 것입니다.
먼저, 몇 가지 필드를 추가합니다. 이 필드 중 하나는 코드 스니펫을 생성한 사용자를 나타내는 데 사용됩니다. 다른 필드는 코드의 하이라이트된 HTML 표현을 저장하는 데 사용됩니다.

`models.py`에서 `Snippet` 모델에 다음 두 필드를 추가합니다.

    owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)
    highlighted = models.TextField()

또한 모델이 저장될 때 `pygments` 코드 하이라이트 라이브러리를 사용하여 하이라이트 필드를 채워야 합니다.

추가 가져오기가 필요합니다:

    from pygments.lexers import get_lexer_by_name
    from pygments.formatters.html import HtmlFormatter
    from pygments import highlight

이제 모델 클래스에 `.save()` 메서드를 추가할 수 있습니다:

    def save(self, *args, **kwargs):
        """
        `pygments` 라이브러리를 사용하여 코드 스니펫의 하이라이트된 HTML 표현을 생성합니다.
        """
        lexer = get_lexer_by_name(self.language)
        linenos = 'table' if self.linenos else False
        options = {'title': self.title} if self.title else {}
        formatter = HtmlFormatter(style=self.style, linenos=linenos,
                                  full=True, **options)
        self.highlighted = highlight(self.code, lexer, formatter)
        super().save(*args, **kwargs)

모든 작업이 완료되면 데이터베이스 테이블을 업데이트해야 합니다.
일반적으로 데이터베이스 마이그레이션을 생성하여 이를 수행하지만, 이 튜토리얼의 목적을 위해 데이터베이스를 삭제하고 다시 시작하겠습니다.

    rm -f db.sqlite3
    rm -r snippets/migrations
    python manage.py makemigrations snippets
    python manage.py migrate

또한 API를 테스트하기 위해 몇 명의 다른 사용자를 생성하고 싶을 수도 있습니다. 이를 가장 빠르게 수행하는 방법은 `createsuperuser` 명령을 사용하는 것입니다.

    python manage.py createsuperuser

## 사용자 모델에 대한 엔드포인트 추가

이제 작업할 사용자가 있으므로, 해당 사용자의 표현을 API에 추가하는 것이 좋습니다. 새로운 직렬화기를 생성하는 것은 간단합니다. `serializers.py`에 다음을 추가합니다:

    from django.contrib.auth.models import User

    class UserSerializer(serializers.ModelSerializer):
        snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

        class Meta:
            model = User
            fields = ['id', 'username', 'snippets']

`'snippets'`는 User 모델에서 *역방향* 관계이므로, `ModelSerializer` 클래스를 사용할 때 기본적으로 포함되지 않습니다. 따라서 명시적인 필드를 추가해야 했습니다.

또한 `views.py`에 몇 가지 뷰를 추가하겠습니다. 사용자 표현에 대해 읽기 전용 뷰만 사용하고 싶으므로, `ListAPIView` 및 `RetrieveAPIView` 제네릭 클래스 기반 뷰를 사용하겠습니다.

    from django.contrib.auth.models import User


    class UserList(generics.ListAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer


    class UserDetail(generics.RetrieveAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer

`UserSerializer` 클래스를 가져오는 것도 잊지 마세요:

    from snippets.serializers import UserSerializer

마지막으로 URL 구성에서 해당 뷰를 참조하여 API에 추가해야 합니다. `snippets/urls.py`의 패턴에 다음을 추가합니다.

    path('users/', views.UserList.as_view()),
    path('users/<int:pk>/', views.UserDetail.as_view()),

## 스니펫을 사용자와 연결하기

현재로서는 코드 스니펫을 생성하더라도, 해당 스니펫 인스턴스와 생성자를 연결할 방법이 없습니다. 사용자는 직렬화된 표현의 일부로 전송되지 않지만, 들어오는 요청의 속성입니다.

이를 처리하는 방법은 스니펫 뷰에서 인스턴스 저장 관리 방식을 수정하고, 들어오는 요청이나 요청된 URL에 암시적으로 포함된 정보를 처리할 수 있도록 `.perform_create()` 메서드를 재정의하는 것입니다.

`SnippetList` 뷰 클래스에 다음 메서드를 추가합니다:

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)

이제 직렬화기의 `create()` 메서드에는 요청에서 검증된 데이터와 함께 추가 `'owner'` 필드가 전달됩니다.

## 직렬화기 업데이트

이제 스니펫이 생성한 사용자와 연결되었으므로, `SnippetSerializer`를 업데이트하여 이를 반영하겠습니다. `serializers.py`의 직렬화기 정의에 다음 필드를 추가합니다:

    owner = serializers.ReadOnlyField(source='owner.username')

**참고**: 내장 `Meta` 클래스의 필드 목록에 `'owner',`를 추가하는 것을 잊지 마세요.

이 필드는 매우 흥미로운 작업을 수행합니다. `source` 인수는 필드를 채우는 데 사용되는 속성을 제어하며, 직렬화된 인스턴스의 모든 속성을 가리킬 수 있습니다. 위에서 보여준 것처럼 점 표기법을 사용할 수도 있으며, 이는 Django의 템플릿 언어에서 사용되는 방식과 유사하게 주어진 속성을 순회합니다.

추가한 필드는 다른 형식의 필드(`CharField`, `BooleanField` 등)와는 대조적으로 비형식 필드인 `ReadOnlyField` 클래스입니다. 비형식 `ReadOnlyField`는 항상 읽기 전용이며, 직렬화된 표현에 사용되지만, 역직렬화될 때 모델 인스턴스를 업데이트하는 데는 사용되지 않습니다. 여기에서는 `CharField(read_only=True)`를 사용할 수도 있습니다.

## 뷰에 필요한 권한 추가

이제 코드 스니펫이 사용자와 연결되었으므로, 인증된 사용자만 코드 스니펫을 생성, 업데이트 및 삭제할 수 있도록 하고 싶습니다.

REST 프레임워크에는 주어진 뷰에 누가 접근할 수 있는지를 제한하는 데 사용할 수 있는 여러 권한 클래스가 포함되어 있습니다. 이 경우 우리가 찾고 있는 것은 `IsAuthenticatedOrReadOnly`로, 인증된 요청은 읽기-쓰기 권한을 갖고, 인증되지 않은 요청은 읽기 전용 권한을 갖게 됩니다.

먼저 뷰 모듈에 다음 가져오기를 추가합니다.

    from rest_framework import permissions

그런 다음, `SnippetList` 및 `SnippetDetail` 뷰 클래스 **모두에** 다음 속성을 추가합니다.

    permission_classes = [permissions.IsAuthenticatedOrReadOnly]

## 브라우저 API에 로그인 추가

현재 브라우저를 열고 브라우저 API로 이동하면 더 이상 새로운 코드 스니펫을 생성할 수 없음을 알 수 있습니다. 이를 위해서는 사용자로 로그인할 수 있어야 합니다.

브라우저 API에서 사용할 수 있는 로그인 뷰를 추가하려면 프로젝트 수준의 `urls.py` 파일에서 URL 구성을 편집합니다.

파일 상단에 다음 가져오기를 추가합니다:

    from django.urls import path, include

그리고 파일 끝에 브라우저 API의 로그인 및 로그아웃 뷰를 포함하는 패턴을 추가합니다.

    urlpatterns += [
        path('api-auth/', include('rest_framework.urls')),
    ]

패턴의 `'api-auth/'` 부분은 실제로 사용하고자 하는 URL로 변경할 수 있습니다.

이제 브라우저를 열고 페이지를 새로 고침하면 페이지 오른쪽 상단에 'Login' 링크가 표시됩니다. 이전에 생성한 사용자 중 하나로 로그인하면 다시 코드 스니펫을 생성할 수 있습니다.

몇 개의 코드 스니펫을 생성한 후, '/users/' 엔드포인트로 이동하여 각 사용자의 'snippets' 필드에 연결된 스니펫 ID 목록이 포함된 표현을 확인합니다.

## 객체 수준 권한

실제로는 모든 코드 스니펫이 누구에게나 보이기를 원하지만, 해당 코드 스니펫을 생성한 사용자만 업데이트하거나 삭제할 수 있도록 하고 싶습니다.

이를 위해 사용자 지정 권한을 만들어야 합니다.

snippets 앱에 `permissions.py`라는 새 파일을 만듭니다.

    from rest_framework import permissions


    class IsOwnerOrReadOnly(permissions.BasePermission):
        """
        객체의 소유자만 편집할 수 있도록 허용하는 사용자 지정 권한입니다.
        """

        def has_object_permission(self, request, view, obj):
            # 읽기 권한은 모든 요청에 허용됩니다.
            # 따라서 GET, HEAD, OPTIONS 요청은 항상 허용합니다.
            if request.method in permissions.SAFE_METHODS:
                return True

            # 쓰기 권한은 스니펫의 소유자에게만 허용됩니다.
            return obj.owner == request.user

이제 `SnippetDetail` 뷰 클래스의 `permission_classes` 속성을 편집하여 해당 사용자 지정 권한을 스니펫 인스턴스 엔드포인트에 추가할 수 있습니다.

    permission_classes = [permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly]

`IsOwnerOrReadOnly` 클래스를 가져오는 것도 잊지 마세요:

    from snippets.permissions import IsOwnerOrReadOnly

이제 다시 브라우저를 열면, 'DELETE' 및 'PUT' 작업이 코드 스니펫을 생성한 사용자로 로그인했을 때만 스니펫 인스턴스 엔드포인트에 나타납니다.

## API 인증

이제 API에 대한 권한 세트가 있으므로, 스니펫을 편집하려면 요청에 대해 인증해야 합니다. 아직 [인증 클래스][authentication]를 설정하지 않았으므로 기본값인 `SessionAuthentication`과 `BasicAuthentication`이 현재 적용됩니다.

웹 브라우저를 통해 API와 상호 작용할 때 로그인할 수 있으며, 브라우저 세션이 요청에 필요한 인증을 제공합니다.

프로그램 방식으로 API와 상호 작용할 때는 각 요청에 인증 자격 증명을 명시적으로 제공해야 합니다.

인증 없이 스니펫을 생성하려고 하면 오류가 발생합니다:

    http POST http://127.0.0.1:8000/snippets/ code="print(123)"

    {
        "detail": "Authentication credentials were not provided."
    }

이전에 생성한 사용자의 사용자 이름과 비밀번호를 포함하여 성공적인 요청을 할 수 있습니다.

    http -a admin:password123 POST http://127.0.0.1:8000/snippets/ code="print(789)"

    {
        "id": 1,
        "owner": "admin",
        "title": "foo",
        "code": "print(789)",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

## 요약

이제 웹 API에 대한 세밀한 권한 세트와 시스템 사용자 및 그들이 생성한 코드 스니펫에 대한 엔드포인트가 있습니다.

[튜토리얼 5부][tut-5]에서는 하이라이트된 스니펫에 대한 HTML 엔드포인트를 생성하여 모든 것을 통합하고, 시스템 내 관계에 대한 하이퍼링크를 사용하여 API의 응집력을 개선하는 방법을 살펴보겠습니다.

[authentication]: ../api-guide/authentication.md
[tut-5]: 5-relationships-and-hyperlinked-apis.md
