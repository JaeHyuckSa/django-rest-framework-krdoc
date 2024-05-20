# AJAX, CSRF 및 CORS 작업

> "자신의 웹사이트에서 발생할 수 있는 CSRF / XSRF 취약점을 자세히 살펴보세요. 이는 매우 쉬운 공격 방식으로, 공격자가 쉽게 악용할 수 있지만, 소프트웨어 개발자가 직관적으로 이해하기는 어렵습니다. 적어도 한 번 당해보기 전까지는요."
>
> &mdash; [Jeff Atwood][cite]

## 자바스크립트 클라이언트

웹 API와 상호 작용하기 위해 자바스크립트 클라이언트를 구축하는 경우, 클라이언트가 웹사이트의 다른 부분에서 사용되는 동일한 인증 정책을 사용할 수 있는지, CSRF 토큰이나 CORS 헤더를 사용할 필요가 있는지 고려해야 합니다.

API와 상호 작용하는 동일한 컨텍스트 내에서 이루어지는 AJAX 요청은 일반적으로 `SessionAuthentication`을 사용합니다. 이는 사용자가 로그인한 후에 이루어지는 모든 AJAX 요청이 웹사이트의 다른 부분에서 사용되는 동일한 세션 기반 인증을 사용하여 인증될 수 있음을 보장합니다.

API와 통신하는 다른 사이트에서 이루어지는 AJAX 요청은 일반적으로 `TokenAuthentication`과 같은 비세션 기반 인증 스킴을 사용해야 합니다.

## CSRF 보호

[크로스 사이트 요청 위조(Cross Site Request Forgery, CSRF)][csrf] 보호는 사용자가 웹사이트에서 로그아웃하지 않고 유효한 세션을 유지할 때 발생할 수 있는 특정 유형의 공격을 방지하는 메커니즘입니다. 이 경우 악의적인 사이트가 로그인된 세션의 컨텍스트 내에서 대상 사이트에 대해 작업을 수행할 수 있습니다.

이러한 공격을 방지하려면 두 가지 작업을 수행해야 합니다:

1. `GET`, `HEAD` 및 `OPTIONS`와 같은 '안전한' HTTP 작업이 서버 측 상태를 변경할 수 없도록 합니다.
2. `POST`, `PUT`, `PATCH` 및 `DELETE`와 같은 '안전하지 않은' HTTP 작업은 항상 유효한 CSRF 토큰이 필요하도록 합니다.

`SessionAuthentication`을 사용하는 경우, `POST`, `PUT`, `PATCH` 또는 `DELETE` 작업에 대해 유효한 CSRF 토큰을 포함해야 합니다.

AJAX 요청을 수행하려면 [Django 문서][csrf-ajax]에 설명된 대로 HTTP 헤더에 CSRF 토큰을 포함해야 합니다.

## CORS

[크로스-오리진 리소스 공유(Cross-Origin Resource Sharing, CORS)][cors]는 클라이언트가 다른 도메인에 호스팅된 API와 상호 작용할 수 있도록 하는 메커니즘입니다. CORS는 서버가 특정 헤더 세트를 포함하여 브라우저가 교차 도메인 요청을 허용해야 하는 시기와 방법을 결정할 수 있도록 요구합니다.

REST 프레임워크에서 CORS를 처리하는 가장 좋은 방법은 미들웨어에 필요한 응답 헤더를 추가하는 것입니다. 이렇게 하면 뷰의 동작을 변경하지 않고도 CORS가 투명하게 지원됩니다.

[Adam Johnson][adamchainz]은 REST 프레임워크 API와 올바르게 작동하는 것으로 알려진 [django-cors-headers] 패키지를 유지 관리합니다.

[cite]: https://blog.codinghorror.com/preventing-csrf-and-xsrf-attacks/
[csrf]: https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)
[csrf-ajax]: https://docs.djangoproject.com/en/stable/howto/csrf/#using-csrf-protection-with-ajax
[cors]: https://www.w3.org/TR/cors/
[adamchainz]: https://github.com/adamchainz
[django-cors-headers]: https://github.com/adamchainz/django-cors-headers
