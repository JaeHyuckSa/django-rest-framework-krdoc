# 브라우저 API

> 우리가 하고 있는 일을 생각하는 습관을 길러야 한다는 것은 매우 잘못된 진리입니다. 정확히 그 반대가 사실입니다. 문명은 우리가 생각하지 않고도 수행할 수 있는 중요한 작업의 수를 늘림으로써 발전합니다.
>
> &mdash; [Alfred North Whitehead][cite], An Introduction to Mathematics (1911)

API는 애플리케이션 *프로그래밍* 인터페이스를 의미하지만, 인간도 API를 읽을 수 있어야 합니다. 누군가는 프로그래밍을 해야 합니다. Django REST Framework는 `HTML` 형식이 요청되면 각 리소스에 대해 인간 친화적인 HTML 출력을 생성하는 것을 지원합니다. 이러한 페이지는 리소스를 쉽게 탐색할 수 있게 해주며, `POST`, `PUT`, `DELETE`를 사용하여 리소스에 데이터를 제출하는 양식을 제공합니다.

## URL

리소스 출력에 완전히 자격을 갖춘 URL을 포함하면 사람이 쉽게 탐색할 수 있도록 'url화'되어 클릭 가능하게 됩니다. `rest_framework` 패키지에는 이 목적을 위한 [`reverse`][drfreverse] 도우미가 포함되어 있습니다.

## 형식

기본적으로 API는 헤더에 지정된 형식을 반환합니다. 브라우저의 경우 HTML입니다. 요청에서 `?format=`을 사용하여 형식을 지정할 수 있으므로, URL에 `?format=json`을 추가하여 브라우저에서 원시 JSON 응답을 볼 수 있습니다. [Firefox][ffjsonview]와 [Chrome][chromejsonview]용 JSON 뷰어 확장 프로그램이 있습니다.

## 인증

브라우저 가능한 API에 빠르게 인증을 추가하려면 `rest_framework` 네임스페이스 아래 `"login"` 및 `"logout"`이라는 경로를 추가합니다. DRF는 이를 위해 기본 경로를 제공하며, URL 구성에 추가할 수 있습니다:

```python
urlpatterns = [
    # ...
    url(r"^api-auth/", include("rest_framework.urls", namespace="rest_framework"))
]
```

## 사용자 정의

브라우저 API는 [트위터의 Bootstrap][bootstrap] (v 3.4.1)으로 제작되어 있어 외관을 쉽게 사용자 정의할 수 있습니다.

기본 스타일을 사용자 정의하려면 `rest_framework/api.html`이라는 템플릿을 생성하고 `rest_framework/base.html`에서 확장합니다. 예를 들어:

**templates/rest_framework/api.html**

    {% extends "rest_framework/base.html" %}

    ...  # 필요한 사용자 정의로 블록 덮어쓰기

### 기본 테마 재정의

기본 테마를 교체하려면 `api.html`에 `bootstrap_theme` 블록을 추가하고 원하는 Bootstrap 테마 CSS 파일에 대한 `link`를 삽입합니다. 이렇게 하면 포함된 테마가 완전히 교체됩니다.

    {% block bootstrap_theme %}
        <link rel="stylesheet" href="/path/to/my/bootstrap.css" type="text/css">
    {% endblock %}

적합한 미리 제작된 교체 테마는 [Bootswatch][bswatch]에서 사용할 수 있습니다. Bootswatch 테마를 사용하려면 테마의 `bootstrap.min.css` 파일을 다운로드하여 프로젝트에 추가하고 위에서 설명한 대로 기본 테마를 교체합니다. 새로운 테마의 Bootstrap 버전이 기본 테마와 일치하는지 확인하세요.

또한 기본적으로 `navbar-inverse`인 네비게이션 바 변형을 `bootstrap_navbar_variant` 블록을 사용하여 변경할 수 있습니다. 비어 있는 `{% block bootstrap_navbar_variant %}{% endblock %}`은 기본 Bootstrap 네비게이션 바 스타일을 사용합니다.

전체 예제:

    {% extends "rest_framework/base.html" %}

    {% block bootstrap_theme %}
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootswatch@3.4.1/flatly/bootstrap.min.css" type="text/css">
    {% endblock %}

    {% block bootstrap_navbar_variant %}{% endblock %}

기본 bootstrap 테마를 재정의하는 것보다 더 구체적인 CSS 조정을 원한다면 `style` 블록을 재정의할 수 있습니다.

---

![Cerulean 테마][cerulean]

*Bootswatch 'Cerulean' 테마 스크린샷*

---

![Slate 테마][slate]

*Bootswatch 'Slate' 테마 스크린샷*

---

### 사용자 정의를 위한 타사 패키지

직접 사용자 정의하는 대신 타사 패키지를 사용할 수 있습니다. API 사용자 정의를 위한 두 가지 패키지가 있습니다:

* [rest-framework-redesign][rest-framework-redesign] - Bootstrap 5를 사용하여 API를 사용자 정의하기 위한 패키지입니다. 현대적이고 세련된 디자인으로, 다크 모드를 지원합니다.
* [rest-framework-material][rest-framework-material] - Django REST Framework를 위한 머티리얼 디자인입니다.

---

![Django REST Framework Redesign][rfr]

*rest-framework-redesign 스크린샷*

---

![Django REST Framework Material][rfm]

*rest-framework-material 스크린샷*

---

### 블록

`api.html`에서 사용할 수 있는 브라우저 API 기본 템플릿의 모든 블록입니다.

* `body`                       - 전체 html `<body>`.
* `bodyclass`                  - 기본적으로 비어 있는 `<body>` 태그의 클래스 속성.
* `bootstrap_theme`            - Bootstrap 테마의 CSS.
* `bootstrap_navbar_variant`   - 네비게이션 바의 CSS 클래스.
* `branding`                   - 네비게이션 바의 브랜딩 섹션, [Bootstrap 컴포넌트][bcomponentsnav] 참조.
* `breadcrumbs`                - 리소스 중첩을 보여주는 링크로, 사용자가 리소스를 거슬러 올라갈 수 있도록 합니다. 이를 보존하는 것이 좋지만, breadcrumbs 블록을 사용하여 재정의할 수 있습니다.
* `script`                     - 페이지의 JavaScript 파일.
* `style`                      - 페이지의 CSS 스타일시트.
* `title`                      - 페이지 제목.
* `userlinks`                  - 기본적으로 로그인/로그아웃 링크가 포함된 헤더 오른쪽의 링크 목록입니다. 링크를 추가하려면 `{{ block.super }}`을 사용하여 인증 링크를 보존하세요.

#### 컴포넌트

모든 표준 [Bootstrap 컴포넌트][bcomponents]를 사용할 수 있습니다.

#### 툴팁

브라우저 API는 Bootstrap 툴팁 컴포넌트를 사용합니다. `js-tooltip` 클래스와 `title` 속성을 가진 모든 요소는 호버 이벤트 시 해당 제목 내용을 툴팁으로 표시합니다.

### 로그인 템플릿

브랜딩을 추가하고 로그인 템플릿의 외관을 사용자 정의하려면 `login.html`이라는 템플릿을 생성하여 프로젝트에 추가하세요. 예: `templates/rest_framework/login.html`. 이 템플릿은 `rest_framework/login_base.html`에서 확장해야 합니다.

브랜딩 블록을 포함하여 사이트 이름이나 브랜딩을 추가할 수 있습니다:

    {% extends "rest_framework/login_base.html" %}

    {% block branding %}
        <h3 style="margin: 0 0 20px;">My Site Name</h3>
    {% endblock %}

또한 `api.html`과 유사하게 `bootstrap_theme` 또는 `style` 블록을 추가하여 스타일을 사용자 정의할 수 있습니다.

### 고급 사용자 정의

#### 컨텍스트

템플릿에 사용할 수 있는 컨텍스트:

* `allowed_methods`     : 리소스에서 허용하는 메서드 목록
* `api_settings`        : API 설정
* `available_formats`   : 리소스에서 허용하는 형식 목록
* `breadcrumblist`      : 중첩된 리소스 체인을 따르는 링크 목록
* `content`             : API 응답의 내용
* `description`         : docstring에서 생성된 리소스 설명
* `name`                : 리소스의 이름
* `post_form`           : POST 양식에 사용할 양식 인스턴스 (허용되는 경우)
* `put_form`            : PUT 양식에 사용할 양식 인스턴스 (허용되는 경우)
* `display_edit_forms`  : POST, PUT 및 PATCH 양식이 표시되는지 여부를 나타내는 불리언 값
* `request`             : 요청 객체
* `response`            : 응답 객체
* `version`             : Django REST Framework의 버전
* `view`                : 요청을 처리하는 뷰
* `FORMAT_PARAM`        : 뷰가 형식 재정의를 수락할 수 있음
* `METHOD_PARAM`        : 뷰가 메서드 재정의를 수락할 수 있음

템플릿에 전달되는 컨텍스트를 사용자 정의하려면 `BrowsableAPIRenderer.get_context()` 메서드를 재정의할 수 있습니다.

#### base.html 사용 안 함

더 고급 사용자 정의를 위해, 예를 들어 Bootstrap 기반이 없거나 사이트의 나머지 부분과 더 밀접하게 통합되려면 `api.html`이 `base.html`을 확장하지 않도록 선택할 수 있습니다. 그런 다음 페이지 내용과 기능은 전적으로 사용자에게 달려 있습니다.

#### 많은 항목이 있는 `ChoiceField` 처리

관계 또는 `ChoiceField`에 너무 많은 항목이 있을 때 모든 옵션을 포함하는 위젯을 렌더링하면 매우 느려질 수 있으며, 브라우저 API 렌더링 성능이 저하될 수 있습니다.

이 경우 가장 간단한 옵션은 선택 입력을 표준 텍스트 입력으로 교체하는 것입니다. 예를 들어:

     author = serializers.HyperlinkedRelatedField(
        queryset=User.objects.all(),
        style={'base_template': 'input.html'}
    )

#### 자동완성

대안이지만 더 복잡한 옵션은 필요한 경우에만 사용 가능한 옵션의 하위 집합을 로드하고 렌더링하는 자동완성 위젯으로 입력을 교체하는 것입니다. 이를 수행하려면 사용자 정의 자동완성 HTML 템플릿을 작성해야 합니다.

[자동완성 위젯을 위한 다양한 패키지][autocomplete-packages]가 있으며, [django-autocomplete-light][django-autocomplete-light]와 같은 패키지를 참조할 수 있습니다. 이러한 구성 요소를 표준 위젯으로 간단히 포함할 수 없으며 HTML 템플릿을 명시적으로 작성해야 합니다. 이는 REST 프레임워크 3.0이 이제 템플릿 HTML 생성을 사용하기 때문에 더 이상 `widget` 키워드 인수를 지원하지 않기 때문입니다.

---

[cite]: https://en.wikiquote.org/wiki/Alfred_North_Whitehead
[drfreverse]: ../api-guide/reverse.md
[ffjsonview]: https://addons.mozilla.org/en-US/firefox/addon/jsonview/
[chromejsonview]: https://chrome.google.com/webstore/detail/chklaanhfefbnpoihckbnefhakgolnmc
[bootstrap]: https://getbootstrap.com/
[cerulean]: ../img/cerulean.png
[slate]: ../img/slate.png
[bswatch]: https://bootswatch.com/
[bcomponents]: https://getbootstrap.com/2.3.2/components.html
[bcomponentsnav]: https://getbootstrap.com/2.3.2/components.html#navbar
[autocomplete-packages]: https://www.djangopackages.com/grids/g/auto-complete/
[django-autocomplete-light]: https://github.com/yourlabs/django-autocomplete-light
[rest-framework-redesign]: https://github.com/youzarsiph/rest-framework-redesign
[rest-framework-material]: https://github.com/youzarsiph/rest-framework-material
[rfr]: ../img/rfr.png
[rfm]: ../img/rfm.png
