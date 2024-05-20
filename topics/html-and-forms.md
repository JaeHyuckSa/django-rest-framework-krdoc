# HTML 및 폼

REST 프레임워크는 API 스타일 응답과 일반 HTML 페이지 모두에 적합합니다. 또한 시리얼라이저를 HTML 폼으로 사용하고 템플릿에서 렌더링할 수 있습니다.

## HTML 렌더링

HTML 응답을 반환하려면 `TemplateHTMLRenderer` 또는 `StaticHTMLRenderer`를 사용해야 합니다.

`TemplateHTMLRenderer` 클래스는 응답에 컨텍스트 데이터 사전이 포함되기를 기대하며, 뷰 또는 응답에서 지정해야 하는 템플릿을 기반으로 HTML 페이지를 렌더링합니다.

`StaticHTMLRenderer` 클래스는 사전 렌더링된 HTML 콘텐츠 문자열을 포함하는 응답을 기대합니다.

정적 HTML 페이지는 일반적으로 API 응답과 다른 동작을 가지므로 내장된 제네릭 뷰에 의존하지 않고 HTML 뷰를 명시적으로 작성해야 할 것입니다.

여기 "Profile" 인스턴스 목록을 HTML 템플릿에 렌더링하여 반환하는 뷰 예제가 있습니다:

**views.py**:

    from my_project.example.models import Profile
    from rest_framework.renderers import TemplateHTMLRenderer
    from rest_framework.response import Response
    from rest_framework.views import APIView


    class ProfileList(APIView):
        renderer_classes = [TemplateHTMLRenderer]
        template_name = 'profile_list.html'

        def get(self, request):
            queryset = Profile.objects.all()
            return Response({'profiles': queryset})

**profile_list.html**:

    <html><body>
    <h1>Profiles</h1>
    <ul>
        {% for profile in profiles %}
        <li>{{ profile.name }}</li>
        {% endfor %}
    </ul>
    </body></html>

## 폼 렌더링

시리얼라이저는 `render_form` 템플릿 태그를 사용하고 시리얼라이저 인스턴스를 템플릿에 컨텍스트로 포함시켜 폼으로 렌더링할 수 있습니다.

다음 뷰는 모델 인스턴스를 보기 및 업데이트하기 위해 템플릿에서 시리얼라이저를 사용하는 예제를 보여줍니다:

**views.py**:

    from django.shortcuts import get_object_or_404
    from my_project.example.models import Profile
    from rest_framework.renderers import TemplateHTMLRenderer
    from rest_framework.views import APIView


    class ProfileDetail(APIView):
        renderer_classes = [TemplateHTMLRenderer]
        template_name = 'profile_detail.html'

        def get(self, request, pk):
            profile = get_object_or_404(Profile, pk=pk)
            serializer = ProfileSerializer(profile)
            return Response({'serializer': serializer, 'profile': profile})

        def post(self, request, pk):
            profile = get_object_or_404(Profile, pk=pk)
            serializer = ProfileSerializer(profile, data=request.data)
            if not serializer.is_valid():
                return Response({'serializer': serializer, 'profile': profile})
            serializer.save()
            return redirect('profile-list')

**profile_detail.html**:

    {% load rest_framework %}

    <html><body>

    <h1>Profile - {{ profile.name }}</h1>

    <form action="{% url 'profile-detail' pk=profile.pk %}" method="POST">
        {% csrf_token %}
        {% render_form serializer %}
        <input type="submit" value="Save">
    </form>

    </body></html>

### 템플릿 팩 사용

`render_form` 태그는 폼과 폼 필드를 렌더링하는 데 사용할 템플릿 디렉토리를 지정하는 선택적 `template_pack` 인수를 가집니다.

REST 프레임워크에는 모두 Bootstrap 3을 기반으로 한 세 가지 내장 템플릿 팩이 포함되어 있습니다. 내장 스타일은 `horizontal`, `vertical`, 및 `inline`입니다. 기본 스타일은 `horizontal`입니다. 이러한 템플릿 팩 중 하나를 사용하려면 Bootstrap 3 CSS도 포함시켜야 합니다.

다음 HTML은 CDN에 호스팅된 Bootstrap 3 CSS 버전에 링크합니다:

    <head>
        …
        <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/css/bootstrap.min.css">
    </head>

서드 파티 패키지는 필요한 폼 및 필드 템플릿을 포함하는 템플릿 디렉토리를 번들링하여 대체 템플릿 팩을 포함할 수 있습니다.

사용 가능한 세 가지 템플릿 팩을 각각 렌더링하는 방법을 살펴보겠습니다. 이 예제에서는 "Login" 폼을 제공하기 위해 단일 시리얼라이저 클래스를 사용합니다.

    class LoginSerializer(serializers.Serializer):
        email = serializers.EmailField(
            max_length=100,
            style={'placeholder': 'Email', 'autofocus': True}
        )
        password = serializers.CharField(
            max_length=100,
            style={'input_type': 'password', 'placeholder': 'Password'}
        )
        remember_me = serializers.BooleanField()

---

#### `rest_framework/vertical`

표준 Bootstrap 레이아웃을 사용하여 폼 레이블을 해당 제어 입력 위에 배치합니다.

*이것이 기본 템플릿 팩입니다.*

    {% load rest_framework %}

    ...

    <form action="{% url 'login' %}" method="post" novalidate>
        {% csrf_token %}
        {% render_form serializer template_pack='rest_framework/vertical' %}
        <button type="submit" class="btn btn-default">Sign in</button>
    </form>

![수직 폼 예제](../img/vertical.png)

---

#### `rest_framework/horizontal`

2/10 열 분할을 사용하여 레이블과 제어를 나란히 배치합니다.

*이것은 브라우저 API 및 관리자 렌더러에서 사용되는 폼 스타일입니다.*

    {% load rest_framework %}

    ...

    <form class="form-horizontal" action="{% url 'login' %}" method="post" novalidate>
        {% csrf_token %}
        {% render_form serializer %}
        <div class="form-group">
            <div class="col-sm-offset-2 col-sm-10">
                <button type="submit" class="btn btn-default">Sign in</button>
            </div>
        </div>
    </form>

![수평 폼 예제](../img/horizontal.png)

---

#### `rest_framework/inline`

모든 제어를 인라인으로 표시하는 컴팩트한 폼 스타일입니다.

    {% load rest_framework %}

    ...

    <form class="form-inline" action="{% url 'login' %}" method="post" novalidate>
        {% csrf_token %}
        {% render_form serializer template_pack='rest_framework/inline' %}
        <button type="submit" class="btn btn-default">Sign in</button>
    </form>

![인라인 폼 예제](../img/inline.png)

## 필드 스타일

시리얼라이저 필드는 `style` 키워드 인수를 사용하여 렌더링 스타일을 사용자 지정할 수 있습니다. 이 인수는 사용되는 템플릿과 레이아웃을 제어하는 옵션 사전입니다.

필드 스타일을 사용자 지정하는 가장 일반적인 방법은 `base_template` 스타일 키워드 인수를 사용하여 템플릿 팩에서 사용할 템플릿을 선택하는 것입니다.

예를 들어, `CharField`를 기본 HTML 입력 대신 HTML 텍스트 영역으로 렌더링하려면 다음과 같이 사용합니다:

    details = serializers.CharField(
        max_length=1000,
        style={'base_template': 'textarea.html'}
    )

필드가 *포함된 템플릿 팩*의 일부가 아닌 사용자 지정 템플릿을 사용하여 렌더링되도록 하려면 `template` 스타일 옵션을 사용하여 템플릿 이름을 완전히 지정할 수 있습니다:

    details = serializers.CharField(
        max_length=1000,
        style={'template': 'my-field-templates/custom-input.html'}
    )

필드 템플릿은 필드 유형에 따라 추가 스타일 속성도 사용할 수 있습니다. 예를 들어, `textarea.html` 템플릿은 제어 크기에 영향을 미칠 수 있는 `rows` 속성도 허용합니다.

    details = serializers.CharField(
        max_length=1000,
        style={'base_template': 'textarea.html', 'rows': 10}
    )

`base_template` 옵션과 관련된 스타일 옵션의 전체 목록은 아래에 나와 있습니다.

base_template          | 유효한 필드 유형                                           | 추가 스타일 옵션
-----------------------|-------------------------------------------------------------|-----------------------------------------------
input.html             | 모든 문자열, 숫자 또는 날짜/시간 필드                       | input_type, placeholder, hide_label, autofocus
textarea.html          | `CharField`                                                 | rows, placeholder, hide_label
select.html            | `ChoiceField` 또는 관계형 필드 유형                         | hide_label
radio.html             | `ChoiceField` 또는 관계형 필드 유형                         | inline, hide_label
select_multiple.html   | `MultipleChoiceField` 또는 `many=True`인 관계형 필드         | hide_label
checkbox_multiple.html | `MultipleChoiceField` 또는 `many=True`인 관계형 필드         | inline, hide_label
checkbox.html          | `BooleanField`                                              | hide_label
fieldset.html          | 중첩 시리얼라이저                                           | hide_label
list_fieldset.html     | `ListField` 또는 `many=True`인 중첩 시리얼라이저             | hide_label
