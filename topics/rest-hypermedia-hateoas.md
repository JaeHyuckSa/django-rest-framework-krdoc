# REST, 하이퍼미디어 및 HATEOAS

> "REST"라는 단어를 계속 사용하고 있습니다. 하지만 그것이 무엇을 의미하는지 모르는 것 같습니다.
>
> &mdash; Mike Amundsen, [REST fest 2012 기조연설][cite].

먼저, 면책 조항입니다. "Django REST framework"라는 이름은 2011년 초에 결정되었으며, 개발자가 쉽게 찾을 수 있도록 하기 위해 선택되었습니다. 문서 전반에 걸쳐 "웹 API"라는 더 간단하고 기술적으로 정확한 용어를 사용하려고 노력합니다.

하이퍼미디어 API 설계에 진지하다면, 설계 선택을 안내하는 데 도움이 되는 이 문서 외부의 리소스를 찾아보는 것이 좋습니다.

다음은 "필수 읽기" 범주에 속합니다.

* Roy Fielding의 논문 - [네트워크 기반 소프트웨어 아키텍처의 설계][dissertation].
* Roy Fielding의 "[REST API는 하이퍼텍스트 기반이어야 합니다][hypertext-driven]" 블로그 게시물.
* Leonard Richardson & Mike Amundsen의 [RESTful Web APIs][restful-web-apis].
* Mike Amundsen의 [HTML5 및 Node로 하이퍼미디어 API 구축][building-hypermedia-apis].
* Steve Klabnik의 [하이퍼미디어 API 설계][designing-hypermedia-apis].
* [Richardson 성숙도 모델][maturitymodel].

더 철저한 배경 지식을 원한다면 Klabnik의 [하이퍼미디어 API 읽기 목록][readinglist]을 확인하세요.

## REST 프레임워크로 하이퍼미디어 API 구축

REST 프레임워크는 독립적인 웹 API 도구 키트입니다. 잘 연결된 API를 구축하도록 안내하고 적절한 미디어 유형을 쉽게 설계할 수 있도록 도와주지만, 특정 설계 스타일을 엄격히 강요하지는 않습니다.

## REST 프레임워크가 제공하는 것

REST 프레임워크가 하이퍼미디어 API를 구축할 수 있게 해준다는 것은 자명합니다. 제공되는 브라우저 API는 웹의 하이퍼미디어 언어인 HTML을 기반으로 합니다.

REST 프레임워크는 또한 적절한 미디어 유형을 쉽게 구축할 수 있도록 하는 [직렬화] 및 [파서]/[렌더러] 구성 요소, 잘 연결된 시스템을 구축하기 위한 [하이퍼링크된 관계][fields], 그리고 [콘텐츠 협상][conneg]에 대한 훌륭한 지원을 포함합니다.

## REST 프레임워크가 제공하지 않는 것

REST 프레임워크가 제공하지 않는 것은 [HAL][hal], [Collection+JSON][collection], [JSON API][json-api] 또는 HTML [마이크로포맷]과 같은 기계 읽기 가능한 하이퍼미디어 형식이나 하이퍼미디어 기반 폼 설명 및 의미적으로 라벨이 지정된 하이퍼링크를 포함하는 완전한 HATEOAS 스타일의 API를 자동으로 생성하는 기능입니다. 그렇게 하려면 API 설계에 대한 의견이 반영된 선택을 해야 하는데, 이는 프레임워크의 범위 밖에 있어야 합니다.

[cite]: https://vimeo.com/channels/restfest/49503453
[dissertation]: https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm
[hypertext-driven]: https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven
[restful-web-apis]: http://restfulwebapis.org/
[building-hypermedia-apis]: https://www.amazon.com/Building-Hypermedia-APIs-HTML5-Node/dp/1449306578
[designing-hypermedia-apis]: http://designinghypermediaapis.com/
[readinglist]: http://blog.steveklabnik.com/posts/2012-02-27-hypermedia-api-reading-list
[maturitymodel]: https://martinfowler.com/articles/richardsonMaturityModel.html

[hal]: http://stateless.co/hal_specification.html
[collection]: http://www.amundsen.com/media-types/collection/
[json-api]: http://jsonapi.org/
[microformats]: http://microformats.org/wiki/Main_Page
[serialization]: ../api-guide/serializers.md
[parser]: ../api-guide/parsers.md
[renderer]: ../api-guide/renderers.md
[fields]: ../api-guide/fields.md
[conneg]: ../api-guide/content-negotiation.md
