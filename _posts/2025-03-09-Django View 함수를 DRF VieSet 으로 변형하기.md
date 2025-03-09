## 과제 목표

**"왜 모두 ViewSet을 사용할까?"**  
지난해, 웹 개발자로 첫 프로젝트를 마주했을 때 Django의 Function-Based View(FBV)는 나에게 '안전한 선택'이었다. 직관적인 요청-응답 구조와 익숙한 방식이었기 때문이다. 하지만 상사들의 코드 중 최근에 작성된 코드일수록 DRF ViewSet이 빈번히 등장하는 것을 보며 언제 ViewSet을 사용하는 것이 좋은 것일까 라는 의문이 생겼다.

**DRF 적응기 + 언제 ViewSet을 도입해야 하는가에 대한 판단력 기르기"**  
DRF의 Tutorial 을 1회 따라하고 난 지금, 단순히 이론으로만 알고 있던 'ViewSet의 장점'을 직접 (반half)실전을 통해 체감해보고 싶다는 생각을 하게 되었다.

## 기존 코드 (삽입)

**즐겨찾기 등록(삽입)**

-   사용자마다 즐겨찾기 신청 개수 제한이 존재한다.
-   사용자가 즐겨찾기를 등록할 때 붙이는 제목은 중복이 불가능하다. (중복 불가 범위는 동일 사용자)

### serializers.py

```
class FavoriteSerializer(serializers.ModelSerializer):
    '''
    Favortie 역직렬화를 위한 Serializer
    '''
    class Meta:
        model = Favorite
        exclude = [...]

    def before_validate(self, data):
        processed_data = {
            'name': str(data.pop('name')),
            ...
        }

        '''
        데이터 가공
        '''

        processed_data['param'].update({'data': data})
        self.initial_data = processed_data
        return self.initial_data
```

### favorite.py

```
def is_available_favorite(favorite_name, user):
    '''
    즐겨찾기 제목의 중복 여부 검사
    즐겨찾기 등록 개수 검사
    '''
    favorite_list = Favorite QuerySet 조회(특정 사용자의 즐겨찾기만 가져와야 함.)
    if favorite_list:
        # 제한 개수 검사.
        # 중복 이름 검사
    return FavoriteResultReason.결과(), 사유

@api_view(['POST'])  
def insert_favorite(request):  
    resp = Response 인스턴스 생성

    if 유저 유효성 확인:  
        # favorite dat 유효성 검사
        favorite_serializer = favoriteSerializer()
        favorite_serializer.before_validate(request.POST.dict()) # 직접 구현한 역직렬화 데이터 변환 함수
        if favorite_serializer.is_valid():  
            favorite_data = favorite_serializer.validated_data  
        else:  
            return resp.실패()

        # favorite 등록 제한 개수 / favorite 제목 중복 검사  
        res_code, res_msg = is_available_favorite(favorite_data.get('name'))  
        if 등록 불가한 content 이면:  
            resp.실패()
        else:  
            # 즐겨찾기 인스턴스 삽입
            favorite = favorite_serializer.save(client.id, user_id)  
            if favorite:
                pass
    else:  
        return resp.로그인()
    return JsonResponse(resp)
```

-   Favortie 역직렬화를 위한 Serializer 구현
-   Serializer 의 유효성 검사 전에 수행할 데어터 가공 함수 구현 (before\_validate)
-   View 함수 구현: 요청 데이터 유효성 검사, 사용자 입력 데이터 유효성 검사

## 바뀐 코드 (삽입)

### serializers.py

```
from rest_framework import serializers
from .models import Favorite

class FavoriteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Favorite
        fields = ['fav_name']

    def to_internal_value(self, data): # before_validate() 을 대체
        processed_data = data.copy()

        '''
        데이터 가공
        ''' 

        return super().to_internal_value(processed_data)

    def validate(self, attrs): # 추가 유효성 검사
        if 'name' not in attrs:
            raise serializers.ValidationError("Name is required")
        return attrs
```

### favorite.py

```
from .models import Favorite
from .serializers import FavoriteSerializer
from .constant import MAX_FAVORITES

class FavoriteViewSet(viewsets.ViewSet):
    serializer_class = FavoriteSerializer
    permission_classes = [permissions.IsAuthenticated]

    @action(detail=False, methods=['post'])
    def create(self, request):
        serializer = self.get_serializer(data=request.data)

        if not serializer.is_valid():
            return return Response(
            serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        # 등록 가능 여부 검증
        favorite_name = serializer.validated_data.get('fav_name')
        validation_result = self.is_available_favorite(favorite_name)
        if validation_result['is_available'] is False:
            return Response(status=status.HTTP_400_BAD_REQUEST)

        # 즐겨찾기 저장
        try:
            favorite = serializer.save('사용자 정보')
            return Response(status=status.HTTP_201_CREATED)

        except Exception as e:
            return Response(status=status.HTTP_500_INTERNAL_SERVER_ERROR)

    def is_available_favorite(self, favorite_name):
        # AI 도움 받은 부분. 쿼리 실행을 줄일 수 있다.
        exists_query = Favorite.objects.filter(Q(name=favorite_name))
        .aggregate(
            total=Count('id'),
            duplicates=Sum(Case(When(name=favorite_name, then=1), default=0))
        ) # get_queryset() 은 parameter 를 갖도록 재구현 해야 함.

        if exists_query['total'] >= MAX_FAVORITES:
            pass # return 실패
        if exists_query['duplicates'] > 0:
            pass # return 실패

        return {'is_available': True}
```

-   ViewSet 을 사용하면서 기존에 직접 구현했던 인증, Serializer 생성이 자동으로 처리됨.

## 기존 코드 (조회)

**즐겨찾기 조회**

-   특정 사용자의 즐겨 찾기만 조회해야 함. 사용자 정보는 Request 에 포함.
-   즐겨찾기의 type 에 따라 'kw' 값을 다르게 저장해야 한다. kw 는 Favorite 의 속성이 아니고 FE 에서만 필요한 data 이다.
-   즐겨찾기의 특정 속성 값(신규 여부)과 등록일을 기준으로 내림차순 정렬해야 한다.

### serializers.py

```
class FavoriteListVo:
    '''
    Favorite 인스턴스 직렬화를 위한 VO 클래스
    '''
    def __init__(self, id, name, is_new, content_type, updated_at, created_at):  
        self.id = id  
        self.name = name  
        self.is_new = is_new
        self.type = content_type
        self.created_at = created_at  
        self.updated_at = updated_at  

    def to_dict(self):  
        fav = {  
            'fav_id': str(self.id),  
            'fav_name': str(self.name),  
            'fav_new': True if self.is_new else False,
            'fav_type': str(self.type),
            'fav_updated_at': convert_date_format(self.updated_at, new_format='%Y-%m-%d %H:%M:%S'),  
            'fav_created_at': convert_date_format(self.created_at, new_format='%Y-%m-%d %H:%M:%S'),
        }

        fav['kw'] = str(self.name) if self.type == favType.CONTENT_A else ''  
        return fav
```

### favorite.py

```
@api_view(['GET'])  
def favorite_list(request):
    favorite_queryset = Favorite QuerySet 조회(특정 사용자의 것만)
    total_count = len(favorite_queryset)

    new_list = []
    old_list = []
    result_list = []
    for favorite in favorite_queryset:
        favorite_vo = favoriteListVo(...)
        favorite = favorite_vo.to_dict()  
        if favorite['favorite_new']:  
            new_list.append(favorite)  
        else:  
            old_list.append(favorite)

    if new_list:
        new_list.sort(key=lambda x: x['favorite_created_at'], reverse=True)  
        result_list.extend(new_list)  

    if old_list:
        old_list.sort(key=lambda x: (x['favorite_updated_at'], x['favorite_created_at']), reverse=True)  
        result_list.extend(old_list)  

    total_count = len(result_list)  

    context = request.GET.dict()
    context.update({'list': result_list[start_no:end_no], 'page':~ })  
    return render(request, 'favorite_list.html', context)
```

-   조회를 위한 별도의 VO 구현: 데이터 직렬화를 위해 to\_dict() 구현
-   View 함수에는 querySet 정렬 로직 포함

## 바뀐 코드(조회)

### favorite.py

```
# 등록 때 사용했던 FavoriteSerializer 재사용
class FavoriteSerializer(serializers.ModelSerializer):
    fav_id = serializers.CharField(source='id')
    fav_name = serializers.CharField(source='name')
    fav_new = serializers.BooleanField(source='is_new')
    fav_type = serializers.CharField(source='content_type')
    fav_updated_at = serializers.SerializerMethodField()
    fav_created_at = serializers.SerializerMethodField()
    kw = serializers.SerializerMethodField()

    class Meta:
        model = Favorite
        fields = [
            'fav_id', 'fav_name', 'fav_new', 
            'fav_type', 'fav_updated_at', 
            'fav_created_at', 'kw'
        ]

    def get_fav_updated_at(self, obj): # 시리얼라이저가 객체를 직렬화할 때 호출
        return obj.updated_at.strftime('%Y-%m-%d %H:%M:%S')

    def get_fav_updated_atself, obj): # 시리얼라이저가 객체를 직렬화할 때 호출
        return obj.created_at.strftime('%Y-%m-%d %H:%M:%S')

    def get_kw(self, obj): # 시리얼라이저가 객체를 직렬화할 때 호출
        return str(obj.name) if obj.content_type == 'CONTENT_A' else ''
```

### favorite.py

```
# 등록 때 사용했던 FavoriteViewSet 재사용
class FavoriteViewSet(viewsets.GenericViewSet, mixins.ListModelMixin):
    queryset = Favorite.objects.all()
    serializer_class = FavoriteSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        qs = super().get_queryset().filter(
            user=self.request.user
        ).order_by('-created_at')

        # 정렬
        new_list = []
        old_list = []
        for item in queryset:
            if item.is_new:
                new_list.append(item)
            else:
                old_list.append(item)

        new_list_sorted = sorted(
            new_list, 
            key=lambda x: x.created_at, 
            reverse=True
        )

        old_list_sorted = sorted(
            old_list,
            key=lambda x: (x.updated_at, x.created_at),
            reverse=True
        )

        combined_list = new_list_sorted + old_list_sorted      
        return combined_list

    @action(detail=False, methods=['get'])
    def html_list(self, request):
        queryset = self.get_queryset()
        serializer = self.get_serializer(queryset, many=True)

        context = request.GET.dict()
        context.update({
            'list': serializer.data,
            'total_count': len(queryset)
        })

        return render(
            request,
            'favorite_list.html',
            context
        )
```

-   정렬 로직은 get\_queryset() 으로 이동
-   list API는 ViewSet이 제공하는 list 함수를 사용하게 됨.
-   list View 에 해당하는 함수는 정말 '조회'의 기능만 수행함.
-   삽입 때 사용한 FavoriteViewSet 재활용
-   삽입 때 사용한 FavoriteSerializer 재활용

## 왜 이 작업을 기록했는가?

막상 ViewSet 전환 작업을 진행해보니 생각만큼 쉽지 않다고 느꼈다. 코드가 단순해진 것 같지도 않고, 유지 보수 측면에서도 어느 부분이 더 나아진 것인지 확신하지 못한다. DRF Tutorial 을 진행했을 때와 확실히 달랐다. 그만큼 아직 DRF 에 덜 익숙하다는 방증일 것이다.

따라서 지금보다 DRF 에 더 익숙해지기 위해 DRF 에 대한 이해도가 올라갈 때마다 틈틈이 이 글을 복기하며 발전시킬 계획이다.