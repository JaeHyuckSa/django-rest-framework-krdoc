# 튜토리얼 1: 직렬화

## 소개

이 튜토리얼에서는 간단한 pastebin 코드 하이라이팅 웹 API를 만드는 방법을 다룹니다. 진행하면서 REST 프레임워크를 구성하는 다양한 구성 요소를 소개하고, 모든 것이 어떻게 맞물려 작동하는지 종합적으로 이해할 수 있도록 합니다.

이 튜토리얼은 꽤 깊이 있게 다루므로, 시작하기 전에 쿠키와 좋아하는 음료 한 잔을 준비하는 것이 좋습니다. 간단한 개요만 원한다면 [빠른 시작 가이드][quickstart] 문서를 참조하세요.

---

**참고**: 이 튜토리얼의 코드는 GitHub의 [encode/rest-framework-tutorial][repo] 저장소에서 볼 수 있습니다. 완성된 구현체는 테스트를 위한 샌드박스 버전으로 [여기에서][sandbox] 이용할 수 있습니다.

---

## 새 환경 설정

무엇보다도 먼저, [venv]를 사용하여 새 가상 환경을 생성합니다. 이렇게 하면 다른 프로젝트와 독립적으로 패키지 구성을 유지할 수 있습니다.

    python3 -m venv env
    source env/bin/activate

가상 환경 안에 들어가면 패키지 요구 사항을 설치할 수 있습니다.

    pip install django
    pip install djangorestframework
    pip install pygments  # 코드를 하이라이팅하는 데 사용

**참고:** 언제든지 가상 환경을 종료하려면 `deactivate`를 입력하면 됩니다. 자세한 내용은 [venv 문서][venv]를 참조하세요.

## 시작하기

이제 코딩을 시작할 준비가 되었습니다.
먼저 작업할 새 프로젝트를 만듭니다.

    cd ~
    django-admin startproject tutorial
    cd tutorial

완료되면, 간단한 웹 API를 만들기 위한 앱을 생성합니다.

    python manage.py startapp snippets

새 `snippets` 앱과 `rest_framework` 앱을 `INSTALLED_APPS`에 추가해야 합니다. `tutorial/settings.py` 파일을 편집합니다:

    INSTALLED_APPS = [
        ...
        'rest_framework',
        'snippets',
    ]

이제 준비가 되었습니다.

## 작업할 모델 생성

이 튜토리얼에서는 코드 스니펫을 저장하는 간단한 `Snippet` 모델을 생성하는 것으로 시작합니다. `snippets/models.py` 파일을 편집합니다. 참고: 좋은 프로그래밍 관행에는 주석이 포함됩니다. 이 튜토리얼 코드의 저장소 버전에는 주석이 포함되어 있지만, 여기에서는 코드 자체에 집중하기 위해 생략했습니다.

    from django.db import models
    from pygments.lexers import get_all_lexers
    from pygments.styles import get_all_styles

    LEXERS = [item for item in get_all_lexers() if item[1]]
    LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
    STYLE_CHOICES = sorted([(item, item) for item in get_all_styles()])


    class Snippet(models.Model):
        created = models.DateTimeField(auto_now_add=True)
        title = models.CharField(max_length=100, blank=True, default='')
        code = models.TextField()
        linenos = models.BooleanField(default=False)
        language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
        style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

        class Meta:
            ordering = ['created']

스니펫 모델에 대한 초기 마이그레이션을 생성하고 처음으로 데이터베이스를 동기화해야 합니다.

    python manage.py makemigrations snippets
    python manage.py migrate snippets

## 직렬화 클래스 생성

웹 API를 시작하려면 스니펫 인스턴스를 `json`과 같은 표현으로 직렬화하고 역직렬화하는 방법을 제공해야 합니다. 이를 위해 Django의 폼과 매우 유사하게 작동하는 직렬화기를 선언할 수 있습니다. `snippets` 디렉토리에 `serializers.py` 파일을 만들고 다음을 추가합니다.

    from rest_framework import serializers
    from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


    class SnippetSerializer(serializers.Serializer):
        id = serializers.IntegerField(read_only=True)
        title = serializers.CharField(required=False, allow_blank=True, max_length=100)
        code = serializers.CharField(style={'base_template': 'textarea.html'})
        linenos = serializers.BooleanField(required=False)
        language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
        style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

        def create(self, validated_data):
            """
            검증된 데이터를 바탕으로 새 `Snippet` 인스턴스를 생성하여 반환합니다.
            """
            return Snippet.objects.create(**validated_data)

        def update(self, instance, validated_data):
            """
            검증된 데이터를 바탕으로 기존 `Snippet` 인스턴스를 업데이트하여 반환합니다.
            """
            instance.title = validated_data.get('title', instance.title)
            instance.code = validated_data.get('code', instance.code)
            instance.linenos = validated_data.get('linenos', instance.linenos)
            instance.language = validated_data.get('language', instance.language)
            instance.style = validated_data.get('style', instance.style)
            instance.save()
            return instance

직렬화 클래스의 첫 번째 부분은 직렬화/역직렬화되는 필드를 정의합니다. `create()` 및 `update()` 메서드는 `serializer.save()`를 호출할 때 완전한 인스턴스가 생성되거나 수정되는 방법을 정의합니다.

직렬화 클래스는 Django `Form` 클래스와 매우 유사하며, `required`, `max_length`, `default`와 같은 다양한 필드에 대해 유사한 검증 플래그를 포함합니다.

필드 플래그는 HTML로 렌더링할 때와 같은 특정 상황에서 직렬화기가 어떻게 표시되어야 하는지 제어할 수도 있습니다. 위의 `{'base_template': 'textarea.html'}` 플래그는 Django `Form` 클래스의 `widget=widgets.Textarea`를 사용하는 것과 동등합니다. 이는 이후 튜토리얼에서 볼 수 있듯이 브라우저 API가 어떻게 표시되어야 하는지 제어하는 데 특히 유용합니다.

나중에 볼 수 있듯이 `ModelSerializer` 클래스를 사용하여 시간을 절약할 수도 있지만, 지금은 직렬화기 정의를 명시적으로 유지하겠습니다.

## 직렬화기 사용하기

더 나아가기 전에 새 직렬화기 클래스를 사용하는 방법에 익숙해집니다. Django 쉘로 들어갑니다.

    python manage.py shell

몇 가지 가져오기를 마친 후, 작업할 코드 스니펫을 몇 개 생성합니다.

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser

    snippet = Snippet(code='foo = "bar"\n')
    snippet.save()

    snippet = Snippet(code='print("hello, world")\n')
    snippet.save()

이제 몇 개의 스니펫 인스턴스를 갖게 되었습니다. 이러한 인스턴스 중 하나를 직렬화하는 방법을 살펴보겠습니다.

    serializer = SnippetSerializer(snippet)
    serializer.data
    # {'id': 2, 'title': '', 'code': 'print("hello, world")\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}

이 시점에서 모델 인스턴스를 Python 네이티브 데이터 타입으로 변환했습니다. 직렬화 프로세스를 완료하려면 데이터를 `json`으로 렌더링합니다.

    content = JSONRenderer().render(serializer.data)
    content
    # b'{"id": 2, "title": "", "code": "print(\\"hello, world\\")\\n", "linenos": false, "language": "python", "style": "friendly"}'

역직렬화도 유사합니다. 먼저 스트림을 Python 네이티브 데이터 타입으로 파싱합니다...

    import io

    stream = io.BytesIO(content)
    data = JSONParser().parse(stream)

...그런 다음 해당 네이티브 데이터 타입을 완전히 채워진 객체 인스턴스로 복원합니다.

    serializer = SnippetSerializer(data=data)
    serializer.is_valid()
    # True
    serializer.validated_data
    # OrderedDict([('title', ''), ('code', 'print("hello, world")\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
    serializer.save()
    # <Snippet: Snippet object>

폼 작업과 API가 얼마나 유사한지 주목하세요. 직렬화기를 사용하는 뷰를 작성하기 시작하면 유사성이 더 명확해질 것입니다.

모델 인스턴스 대신 쿼리셋을 직렬화할 수도 있습니다. 이를 위해 직렬화기 인수에 `many=True` 플래그를 추가하기만 하면 됩니다.

    serializer = SnippetSerializer(Snippet.objects.all(), many=True)
    serializer.data
    # [OrderedDict([('id', 1), ('title', ''), ('code', 'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 2), ('title', ''), ('code', 'print("hello, world")\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 3), ('title', ''), ('code', 'print("hello, world")'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]

## ModelSerializers 사용하기

`SnippetSerializer` 클래스는 `Snippet` 모델에 포함된 많은 정보를 복제하고 있습니다. 코드를 조금 더 간결하게 유지할 수 있다면 좋을 것입니다.

Django가 `Form` 클래스와 `ModelForm` 클래스를 제공하는 것과 동일하게, REST 프레임워크에는 `Serializer` 클래스와 `ModelSerializer` 클래스가 모두 포함되어 있습니다.

`ModelSerializer` 클래스를 사용하여 직렬화기를 리팩터링하는 방법을 살펴보겠습니다.
`snippets/serializers.py` 파일을 다시 열고, `SnippetSerializer` 클래스를 다음으로 교체합니다.

    class SnippetSerializer(serializers.ModelSerializer):
        class Meta:
            model = Snippet
            fields = ['id', 'title', 'code', 'linenos', 'language', 'style']

직렬화기 인스턴스의 모든 필드를 인스펙트할 수 있는 직렬화기의 멋진 속성 중 하나는 그 표현을 출력하는 것입니다. `python manage.py shell`로 Django 쉘을 열고 다음을 시도하세요:

    from snippets.serializers import SnippetSerializer
    serializer = SnippetSerializer()
    print(repr(serializer))
    # SnippetSerializer():
    #    id = IntegerField(label='ID', read_only=True)
    #    title = CharField(allow_blank=True, max_length=100, required=False)
    #    code = CharField(style={'base_template': 'textarea.html'})
    #    linenos = BooleanField(required=False)
    #    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
    #    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...

`ModelSerializer` 클래스가 특별히 마법적인 일을 하지 않는다는 것을 기억하는 것이 중요합니다. 단순히 직렬화 클래스 생성에 대한 단축키일 뿐입니다:

* 자동으로 결정된 필드 집합.
* `create()` 및 `update()` 메서드에 대한 간단한 기본 구현.

## 직렬화기를 사용한 일반 Django 뷰 작성

새 직렬화기 클래스를 사용하여 일부 API 뷰를 작성하는 방법을 살펴보겠습니다.
지금은 REST 프레임워크의 다른 기능을 사용하지 않고, 일반 Django 뷰로 작성하겠습니다.

`snippets/views.py` 파일을 편집하고 다음을 추가합니다.

    from django.http import HttpResponse, JsonResponse
    from django.views.decorators.csrf import csrf_exempt
    from rest_framework.parsers import JSONParser
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer

API의 루트는 기존 스니펫을 모두 나열하거나 새 스니펫을 생성하는 뷰가 될 것입니다.

    @csrf_exempt
    def snippet_list(request):
        """
        모든 코드 스니펫을 나열하거나 새 스니펫을 생성합니다.
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return JsonResponse(serializer.data, safe=False)

        elif request.method == 'POST':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(data=data)
            if serializer.is_valid():
                serializer.save()
                return JsonResponse(serializer.data, status=201)
            return JsonResponse(serializer.errors, status=400)

CSRF 토큰이 없는 클라이언트에서 이 뷰에 POST할 수 있도록 뷰를 `csrf_exempt`로 표시해야 합니다. 일반적으로 이런 식으로 하고 싶지는 않지만, 지금은 우리의 목적에 맞을 것입니다.

또한 개별 스니펫에 해당하는 뷰가 필요하며, 이를 사용하여 스니펫을 조회, 업데이트 또는 삭제할 수 있습니다.

    @csrf_exempt
    def snippet_detail(request, pk):
        """
        코드 스니펫을 조회, 업데이트 또는 삭제합니다.
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return HttpResponse(status=404)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return JsonResponse(serializer.data)

        elif request.method == 'PUT':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(snippet, data=data)
            if serializer.is_valid():
                serializer.save()
                return JsonResponse(serializer.data)
            return JsonResponse(serializer.errors, status=400)

        elif request.method == 'DELETE':
            snippet.delete()
            return HttpResponse(status=204)

마지막으로 이 뷰들을 연결해야 합니다. `snippets/urls.py` 파일을 생성합니다:

    from django.urls import path
    from snippets import views

    urlpatterns = [
        path('snippets/', views.snippet_list),
        path('snippets/<int:pk>/', views.snippet_detail),
    ]

루트 URL 구성도 연결해야 합니다. `tutorial/urls.py` 파일에서 스니펫 앱의 URL을 포함하도록 합니다.

    from django.urls import path, include

    urlpatterns = [
        path('', include('snippets.urls')),
    ]

현재 몇 가지 경계 사례를 제대로 처리하지 않고 있다는 점을 주목할 필요가 있습니다. 잘못된 `json`을 보내거나, 뷰에서 처리하지 않는 메서드로 요청이 이루어지면 500 "서버 오류" 응답을 받게 됩니다. 하지만 지금은 이 정도면 충분합니다.

## 첫 번째 웹 API 시도 테스트

이제 스니펫을 제공하는 샘플 서버를 시작할 수 있습니다.

쉘에서 나와...

    quit()

...Django 개발 서버를 시작합니다.

    python manage.py runserver

    Validating models...

    0 errors found
    Django version 4.0, using settings 'tutorial.settings'
    Starting Development server at http://127.0.0.1:8000/
    Quit the server with CONTROL-C.

다른 터미널 창에서 서버를 테스트할 수 있습니다.

[curl][curl] 또는 [httpie][httpie]를 사용하여 API를 테스트할 수 있습니다. Httpie는 Python으로 작성된 사용자 친화적인 http 클라이언트입니다. 이를 설치합니다.

pip을 사용하여 httpie를 설치할 수 있습니다:

    pip install httpie

마지막으로 모든 스니펫 목록을 가져올 수 있습니다:

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

또는 ID를 참조하여 특정 스니펫을 가져올 수 있습니다:

    http http://127.0.0.1:8000/snippets/2/

    HTTP/1.1 200 OK
    ...
    {
      "id": 2,
      "title": "",
      "code": "print(\"hello, world\")\n",
      "linenos": false,
      "language": "python",
      "style": "friendly"
    }

마찬가지로, 웹 브라우저에서 이 URL을 방문하여 동일한 json을 표시할 수 있습니다.

## 현재 위치

여기까지 잘 해왔습니다. Django의 폼 API와 매우 유사한 직렬화 API와 일부 일반 Django 뷰를 만들었습니다.

현재 API 뷰는 `json` 응답을 제공하는 것 외에는 특별히 하는 일이 없으며, 아직 정리해야 할 오류 처리 경계 사례가 몇 가지 있지만, 작동하는 웹 API입니다.

[튜토리얼 2부][tut-2]에서 개선하는 방법을 살펴보겠습니다.

[quickstart]: quickstart.md
[repo]: https://github.com/encode/rest-framework-tutorial
[sandbox]: https://restframework.herokuapp.com/
[venv]: https://docs.python.org/3/library/venv.html
[tut-2]: 2-requests-and-responses.md
[httpie]: https://github.com/httpie/httpie#installation
[curl]: https://curl.haxx.se/
