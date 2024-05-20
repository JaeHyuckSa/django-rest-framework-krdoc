# 캐싱

> 어떤 여자가 매우 날카로운 의식을 가지고 있었지만 거의 기억을 하지 못했습니다 ... 그녀는 일할 만큼은 기억했고, 열심히 일했습니다.
> - Lydia Davis

REST 프레임워크에서의 캐싱은 Django에서 제공하는 캐시 유틸리티와 잘 작동합니다.

---

## API 뷰와 뷰셋에서 캐시 사용하기

Django는 클래스 기반 뷰에 데코레이터를 사용할 수 있도록 [`method_decorator`][decorator]를 제공합니다. 이를 다른 캐시 데코레이터, 예를 들어 [`cache_page`][page], [`vary_on_cookie`][cookie], [`vary_on_headers`][headers]와 함께 사용할 수 있습니다.

```python
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie, vary_on_headers

from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework import viewsets


class UserViewSet(viewsets.ViewSet):
    # 쿠키 사용: 각 사용자의 요청 URL을 2시간 동안 캐시
    @method_decorator(cache_page(60 * 60 * 2))
    @method_decorator(vary_on_cookie)
    def list(self, request, format=None):
        content = {
            "user_feed": request.user.get_user_feed(),
        }
        return Response(content)


class ProfileView(APIView):
    # 인증 사용: 각 사용자의 요청 URL을 2시간 동안 캐시
    @method_decorator(cache_page(60 * 60 * 2))
    @method_decorator(vary_on_headers("Authorization"))
    def get(self, request, format=None):
        content = {
            "user_feed": request.user.get_user_feed(),
        }
        return Response(content)


class PostView(APIView):
    # 요청된 URL의 페이지를 캐시
    @method_decorator(cache_page(60 * 60 * 2))
    def get(self, request, format=None):
        content = {
            "title": "Post title",
            "body": "Post content",
        }
        return Response(content)
```

**참고**: cache_page 데코레이터는 상태 코드가 200인 GET 및 HEAD 응답만을 캐시합니다.

[page]: https://docs.djangoproject.com/en/dev/topics/cache/#the-per-view-cache
[cookie]: https://docs.djangoproject.com/en/dev/topics/http/decorators/#django.views.decorators.vary.vary_on_cookie
[headers]: https://docs.djangoproject.com/en/dev/topics/http/decorators/#django.views.decorators.vary.vary_on_headers
[decorator]: https://docs.djangoproject.com/en/dev/topics/class-based-views/intro/#decorating-the-class