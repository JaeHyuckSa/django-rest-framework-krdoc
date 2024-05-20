> HTTP 요청을 절약하기 위해 관련 문서를 요청과 함께 보내는 것이 편리할 수 있습니다.
>
> &mdash; [Ember Data를 위한 JSON API 사양][cite].

# 쓰기 가능한 중첩 시리얼라이저

평탄한 데이터 구조는 서비스의 개별 엔터티 간의 구분을 명확히 하는 데 도움이 되지만, 중첩된 데이터 구조를 사용하는 것이 더 적절하거나 편리한 경우도 있습니다.

중첩된 데이터 구조는 읽기 전용인 경우에는 간단히 시리얼라이저 클래스를 중첩하면 됩니다. 그러나 쓰기 가능한 중첩 시리얼라이저를 사용할 때는 다양한 모델 인스턴스 간의 종속성과 단일 작업에서 여러 인스턴스를 저장하거나 삭제해야 하는 필요성 때문에 몇 가지 미묘한 차이가 있습니다.

## 일대다 데이터 구조

*예시: **읽기 전용** 중첩 시리얼라이저. 여기에는 복잡한 문제가 없습니다.*

    class ToDoItemSerializer(serializers.ModelSerializer):
        class Meta:
            model = ToDoItem
            fields = ['text', 'is_completed']

    class ToDoListSerializer(serializers.ModelSerializer):
        items = ToDoItemSerializer(many=True, read_only=True)

        class Meta:
            model = ToDoList
            fields = ['title', 'items']

시리얼라이저에서 출력된 예시입니다.

    {
        'title': 'Leaving party preparations',
        'items': [
            {'text': 'Compile playlist', 'is_completed': True},
            {'text': 'Send invites', 'is_completed': False},
            {'text': 'Clean house', 'is_completed': False}
        ]
    }

중첩된 일대다 데이터 구조를 업데이트하는 방법을 살펴보겠습니다.

### 유효성 검사 오류

### 항목 추가 및 제거

### PATCH 요청 만들기


[cite]: http://jsonapi.org/format/#url-based-json-api
