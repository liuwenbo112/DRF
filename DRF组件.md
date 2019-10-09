### 解析器组件

实现项目：D:\BaiduNetdiskDownload\课上代码\drfserver

或者      D:\anaconda\Scripts\drfserver

-解析器组件是用来解析用户请求的数据的（application/json), content-type

-必须继承APIView

-request.data触发解析

#### 序列化组件，接口设计

##### 2.1 Django自带的serializer

​	2.1.1 from django.serializers import serialize
​	2.1.2 origin_data = Book.objects.all()
​	2.1.3 serialized_data = serialize("json", origin_data)

##### 2.2 DRF的序列化组件

###### 第一步 导入serializers
```python
from rest_framework import serializers
from .models import Book
```

######  第二步 定义序列化类，model方法，解决字段多的问题

~~~python
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = "__all__" # 表示全部字段,也可以用元组包括所要选择的字段 例如('title','price')
        # extra_kwargs　表示　publish和authors字段是只写的 除了
        extra_kwargs = {
            'publish':{'write_only':True},
            'authors':{'write_only':True},
        }
    # ready_only 表示该字段只读
    publish_name = serializers.CharField(max_length=32,read_only=True,source='publish.name')
    publish_city = serializers.CharField(max_length=32,read_only=True,source='publish.city')
    # 如果是SerializerMethodField()类型的字段则会自动找到以get开头的加上_字段名 的 		   #get_authors_list的方法 
    authors_list = serializers.SerializerMethodField()
    # write_only_fields = ('publish', 'authors')
    def get_authors_list(self, book_obj):
        # 拿到queryset之后开始循环[{}，{}，{}，...]
        author_list = list()
        for author in book_obj.authors.all():
            author_list.append(author.name)
        return author_list
```
~~~

- post接口设计
	总结：
		1. serializers.Serializer无法插入数据，只能自己实现create
		2. 字段太多，不能自动序列化
- get单条数据接口设计
	1. 定义url
	2. 获取queryset数据对象
	3. 开始序列化：serialized_data = BookSerializer(book_obj, many=False)
	4. 返回数据：serialized_data.data
- delete
- put
	1. 定义url
	2. 获取数据对象
		2.1 book_obj = Book.objects.get(pk=1)
	3. 开始序列化（验证数据，save()）
		2.2 verified_data = BookSerializer(instance=book_obj, many=False)
	4. 验证成功写入数据库，验证失败返回错误
		4.1 verified_data.is_valid()

###### 第三步、开始序列化，get、post接口设计

```python
class BookView(APIView):
    # get 接口
    def get(self, request):
        # 第三步，获取queryset
        origin_data = Book.objects.all()

        # 第四步，开始序列化
        serialized_data = BookSerializer(origin_data, many=True)

        return Response(serialized_data.data)
    # post接口
    def post(self, request):

        verified_data = BookSerializer(data=request.data)

        if verified_data.is_valid():
            book = verified_data.save()
            authors = Author.objects.filter(nid__in=request.data['authors'])
            book.authors.add(*authors)
            return Response(verified_data.data)
        else:
            return Response(verified_data.errors)
```

```python
class BookFilterView(APIView):

    # 含参数的get接口设计
    def get(self, request, nid):
        book_obj = Book.objects.get(pk=nid)

        serialized_data = BookSerializer(book_obj, many=False)

        return Response(serialized_data.data)
    # put接口 (更新接口设计)
    def put(self, request, nid):
        book_obj = Book.objects.get(pk=nid)

        verified_data = BookSerializer(data=request.data, instance=book_obj)

        if verified_data.is_valid():
            verified_data.save()
            return Response(verified_data.data)
        else:
            return Response(verified_data.errors)
        # delete 接口（删除接口设计）
        def delete(self, request, nid):
            book_obj = Book.objects.get(pk=nid).delete()

            return Response()
        
   # 这里url格式为：
  re_path(r'books/$', views.BookView.as_view()),
   # 序列化组件的url
  re_path(r'books/(\d+)/$',views.BookFilterView.as_view()),

```

#### 视图组件，接口设计

​	- 视图组件是用来优化接口逻辑

##### 1、使用视图组件的mixin进行接口逻辑优化

```python
# 第一步：导入mixin
from rest_framework.mixins import ListModelMixin,CreateModelMixin,DestroyModelMixin,\    UpdateModelMixin,RetrieveModelMixin
from rest_framework.generics import GenericAPIView

from .models import Book,Publish,Author
# 第二步：导入（或创建）自定义的序列化类
from .app_serializers import BookSerializer
```

```python
class BookView(ListModelMixin,CreateModelMixin,GenericAPIView): 
    # 第三步：获取queryset对象    		
    queryset = Book.objects.all()    
    # 指明序列化类     
    serializer_class = BookSerializer
    # 第四步：序列化
    # get 接口
    def get(self,request,*args,**kwargs):        
        return self.list(request,*args,**kwargs)   
    # post 接口
    def post(self,request,*args,**kwargs):        
        return self.create(request,*args,**kwargs)
```

```python
class BookFilterView(RetrieveModelMixin,DestroyModelMixin,UpdateModelMixin,GenericAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

    # 含有参数的get接口 获取一条数据
    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    # put 接口 更新一条数据
    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    # delete 接口 删除一条数据
    def delete(self,request, *args, **kwargs):
        return self.destroy(request,*args,**kwargs)

# urls设计格式    
re_path(r'books/$', views.BookView.as_view()),
# # 注意：单条数据操作的url是这样的：含有参数的url格式有变化，变成：
re_path(r'books/(?P<pk>\d+)/$,views.BookFilterView.as_view())
```

##### 2、使用视图组件的view进行接口逻辑优化

```python	
from rest_framework import generics

class BookView(generics.ListCreateAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

class BookFilterView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
#  注意：url设计与利用mixin进行接口逻辑优化一致
```

##### 3、 使用视图组件的viewset进行接口逻辑优化

```python
from rest_framework.viewsets import ModelViewSet

class BookView(ModelViewSet):    
    queryset = Book.objects.all()  
    # 需给定自定义的序列化类BookSerializer
    serializer_class = BookSerializer
    
# 需要注意的是利用viewset进行接口逻辑优化时，需要把urls路由改为：
# viewset 的url的方式
re_path(r'books/$',views.BookView.as_view({
    'get':'list',
    'post':'create',
})),
re_path(r'books/(?P<pk>\d+)/$',views.BookView.as_view({
    'get':'retrieve',
    'put':'update',
    'delete':'destroy'
})),
```

#### 认证组件

##### 生成token序列码

get_token.py

```python
import uuid
def generate_token():
    random_str = str(uuid.uuid4()).replace('-','')
    return random_str
```

##### 登录操作

```python
# 导入生成token模块
from .utils import get_token
# 登录
class UserView(APIView):

    def post(self, request):
        # 定义返回的消息体
        response = dict()
        # 定义用户需要的信息
        fields = {'username', 'password'}
        # 　定义一个用户信息字典
        user_info = dict()
        # 判断fields是否是set(request.data)的子集
        if fields.issubset(set(request.data.keys())):
            # username = request.data['username']
            # password = request.data['password']
            for key in fields:
                user_info[key] = request.data[key]

        user_instance = User.objects.filter(**user_info).first()
        if user_instance is not None:
            access_token = get_token.generate_token()
            UserToken.objects.update_or_create(user=user_instance, defaults={
                'token': access_token
            })
            response["status_code"] = 200
            response["status_message"] = "登录成功"
            response["access_token"] = access_token
            response["username"] = user_info['username']
            response["user_role"] = user_instance.get_user_type_display()
        else:
            response["status_code"] = 201
            response["status_message"] = "登录失败，用户名或密码错误"
        return JsonResponse(response)
```

##### 认证操作

```python
from rest_framework.viewsets import ModelViewSet
from rest_framework.exceptions import APIException
from rest_framework.authentication import BaseAuthentication
# 认证组件
# 第一步:定义认证类
class UserAuth(BaseAuthentication):
    # 当继承BaseAuthentication时不用写下面注释掉的
    # def authenticate_header(self,request):
    #     pass
    
    # 所有的认证逻辑都在authenticate中写
    def authenticate(self,request):
        user_token = request.query_params.get("token")
        try:
            token = UserToken.objects.filter(token=user_token).first()
            return token.user.username,token.token
        except Exception:
            raise APIException("没有认证")


```

```python
class BookView(ModelViewSet):
    # 第二步:指定认证类 局部认证
    authentication_classes = [UserAuth]
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

##### 全局认证

在项目文件的setting.py中设置：

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'serializer.authentication_classes.UserAuth',
    )
}
# 其中UserAuth是在serializer下authentication_classes.py文件中定义的认证类
# UserAuth类的内容与前面认证组件的内容一致
```

```python
# 由于在setting.py中设置了全局认证，则在视图类中无需设置authentication_classes
class BookView(ModelViewSet):
    # 第二步:指定认证类 如果在项目的setting.py 中的REST_FRAMEWORK设置了全局认证 则在这不需要
    # authentication_classes = [UserAuth]
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

在 postman中验证访问的格式：

 http://127.0.0.1:8000/serializer/books/1/?token=0a0b94c42840459686aa7a7aea013e7a

注意：当有多个认证类的时候，需要注意的是，如果需要返回数据，请在最后一个认证类中返回

#### 权限组件

```python
# 自定义一个权限类
class UserPerm():
    def has_permission(self,request,view):
        # 如果用户的类型等于 3 也就是 Vvip 通过权限验证
        if request.user.user_type == 3:
            return True
        else:
            return False
    # 需要注意的是
    # 在对单条数据进行操作时需加入下面函数
    def has_object_permission(self, request, view, obj):
        return True
```

```python
class BookView(ModelViewSet):
    # 第二步:指定认证类 如果在项目的setting.py 中的REST_FRAMEWORK设置了全局认证 则在这不需要，这里是局部认证
    authentication_classes = [UserAuth]

    # 权限认证 指定权限认证类，局部认证，同样可以通过REST_FRAMEWORK的DEFAULT_PERMISSION_CLASSES键值设置全局认证
    permission_classes = [UserPerm]

    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
```

#### 频率组件

使用DRF的简单频率控制来控制用户访问频率（局部）

```python
# 导入模块
from rest_framework.throttling import SimpleRateThrottle
# 定义并继承 SimpleRateThrottle
class RateThrottle(SimpleRateThrottle):
    # 指定访问频率  这里是每分钟5次
    rate = '5/m'
    def get_cache_key(self, request, view):
        return self.get_ident(request)
```

```python
class BookView(ModelViewSet):
    # 频率组件 指定频率组件类 局部认证，同样可以通过REST_FRAMEWORK的DEFAULT_THROTTLE_CLASSES键值设置全局认证
    throttle_classes = [RateThrottle]

    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

使用DRF的简单频率控制来控制用户访问频率（全局）

```python
# 导入模块
from rest_framework.throttling import SimpleRateThrottle
# 定义并继承 SimpleRateThrottle
class RateThrottle(SimpleRateThrottle):
    # 指定访问频率  这里是每分钟5次
    scope = 'visit_rate'
    def get_cache_key(self, request, view):
        return self.get_ident(request)
```

在项目文件的setting.py中

```python
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES":(
        "serializer.authentication_classes.RateThrottle"
    ),
    "DEFAULT_THROTTLE_RATES":{
        "visit_rate":"5/m",
    }
}
```

#### url注册器

```python
# 在项目的app文件下的urls。py文件中
# url注册器
# 第一步：导入模块
from rest_framework import routers
# 第二步：生成一个注册器实例对象
router = routers.DefaultRouter()
# 第三步：将需要自动生成url的接口注册
router.register(r"books",views.BookView)
# 第四步：开始自动生成url
urlpatterns = [
    re_path('^',include(router.urls)),
]
```

#### 响应器组件

```python
# 导入模块
from rest_framework.renderers import JSONRenderer,BrowsableAPIRenderer

class BookView(ModelViewSet):
    # 响应器组件 指定响应器组件
    renderer_classes = [JSONRenderer] # 两个任选，默认是BrowsableAPIRenderer
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

#### 分页器组件

##### 1、分页器组件使用方式介绍

```python
1、导入模块
from rest_framework.pagination import PageNumberPagination
2、获取数据
books = Book.objects.all()
3、创建一个分页器对象
paginater = PageNumberPagination()
4、开始分页
paged_books = paginater.paginate_queryset(books,request)
5、开始序列化
serialized_books = BookSerializer(paged_books,many=True)
6、返回数据
return Response(serialized_books.data)

```

##### 2、分页器组件局部实现

```python
# 主要参数介绍
page_zize:用来控制每页显示多少条数据（全局参数名为PAGE_SIZE）
page_query_param:用来提供直接访问某页的数据
page_sie_query_param:临时调整当前显示多少条数据
max_page_size:控制page_sie_query_param参数能调整的最大参数
```

```python
1、导入模块
from rest_framework.pagination import PageNumberPagination

2、自定义一个分页类并继承 PageNumberPagination

class MyPagination(PageNumberPagination):
    page_size = 3
    page_query_param = 'page'
    page_sie_query_param = 'size'
    max_page_size = 5
    
# 3、实例化一个分页类对象
# paginater = MyPagination()
# 4、开始分页
# paged_books = paginater.paginate_queryset(books,request)
# 5、开始序列化
# serialized_books = BookSerializer(paged_books,many=True)
# 6、返回数据
# return Response(serialized_books.data)
```

Postman中

page=1表示访问第一页的数据

http://127.0.0.1:8000/serializer/books/?page=1&size=5&token=2cc723adf9424341bde4033bd0ae0453

```python
# 以图书视图类为例
class BookView(ModelViewSet):
    
    # 分页器组件 指定分页器组件
    pagination_class = MyPagination

    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

#### ContentType

```
Django contenttypes 应用
contenttypes 是Django内置的一个应用，可以追踪项目中所有app和model的对应关系，并记录在ContentType表中。

每当我们创建了新的model并执行数据库迁移后，ContentType表中就会自动新增一条记录。比如我在应用app01的models.py中创建表
那么这个表有什么作用呢？这里提供一个场景，网上商城购物时，会有各种各样的优惠券，比如通用优惠券，满减券，或者是仅限特定品类的优惠券。在数据库中，可以通过外键将优惠券和不同品类的商品表关联起来：

​```python
from django.db import models

class Electrics(models.Model):
    """
    id  name
    1   日立冰箱
    2   三星电视
    3   小天鹅洗衣机
    """
    name = models.CharField(max_length=32)

class Foods(models.Model):
    """
    id   name
    1    面包
    2    烤鸭
    """
    name = models.CharField(max_length=32)

class Clothes(models.Model):
    name = models.CharField(max_length=32)

class Coupon(models.Model):
    """
    id     name            Electrics        Foods           Clothes        more...
    1     通用优惠券       null              null            null           
    2     冰箱满减券         2               null            null
    3     面包狂欢节        null              1              null
	"""
    name = models.CharField(max_length=32)
    electric_obj = models.ForeignKey(to='Electrics', null=True)
    food_obj = models.ForeignKey(to='Foods', null=True)
    cloth_obj = models.ForeignKey(to='Clothes', null=True)
​```


```

如果是通用优惠券，那么所有的ForeignKey为null，如果仅限某些商品，那么对应商品ForeignKey记录该商品的id，不相关的记录为null。但是这样做是有问题的：实际中商品品类繁多，而且很可能还会持续增加，那么优惠券表中的外键将越来越多，但是每条记录仅使用其中的一个或某几个外键字段。

通过使用contenttypes 应用中提供的特殊字段GenericForeignKey，我们可以很好的解决这个问题。只需要以下三步：

在model中定义ForeignKey字段，并关联到ContentType表。通常这个字段命名为“content_type”
在model中定义PositiveIntegerField字段，用来存储关联表中的主键。通常这个字段命名为“object_id”
在model中定义GenericForeignKey字段，传入上述两个字段的名字。
为了更方便查询商品的优惠券，我们还可以在商品类中通过GenericRelation字段定义反向关系。

示例代码：

```python
from django.db import models
from django.contrib.contenttypes.models import ContentType
from django.contrib.contenttypes.fields import GenericForeignKey

class Electrics(models.Model):
    name = models.CharField(max_length=32)
    coupons = GenericRelation(to='Coupon')  # 用于反向查询，不会生成表字段
    def __str__(self):
        return self.name
    
class Foods(models.Model):
    name = models.CharField(max_length=32)
    coupons = GenericRelation(to='Coupon')
    def __str__(self):
    	return self.name
    
class Clothes(models.Model):
    name = models.CharField(max_length=32)
    coupons = GenericRelation(to='Coupon')
    def __str__(self):
    	return self.name
    
class Coupon(models.Model):
    name = models.CharField(max_length=32)
    content_type = models.ForeignKey(to=ContentType) # step 1
    object_id = models.PositiveIntegerField() # step 2
    content_object = GenericForeignKey('content_type', 'object_id') # step 3

    def __str__(self):
        return self.name
```


创建记录和查询

```python
from django.shortcuts import render, HttpResponse
from app01 import models
from django.contrib.contenttypes.models import ContentType

def test(request):
    if request.method == 'GET':
        # ContentType表对象有model_class() 方法，取到对应model
        content = ContentType.objects.filter(app_label='app01',model='electrics').first()  		   # 表名小写
        cloth_class = content.model_class() # cloth_class 就相当于models.Electrics
        res = cloth_class.objects.all()
        print(res)

        # 为三星电视(id=2)创建一条优惠记录
        s_tv = models.Electrics.objects.filter(id=2).first()
        models.Coupon.objects.create(name='电视优惠券', content_object=s_tv)

        # 查询优惠券（id=1）绑定了哪些商品
        coupon_obj = models.Coupon.objects.filter(id=1).first()
        prod = coupon_obj.content_object
        print(prod)

        # 查询三星电视(id=2)的所有优惠券
        res = s_tv.coupons.all()
        print(res)

        # 查询obj的所有优惠券：如果没有定义反向查询字段，通过如下方式：
        content=ContentType.objects.filter(app_label='app01',model='model_name').first()
        res = models.OftenAskedQuestion.objects.filter(content_type=content, object_id=obj.pk).all()

        return HttpResponse('....')
```


当一张表和多个表FK关联，并且多个FK中只能选择其中一个或其中n个时，可以利用contenttypes app，只需定义三个字段就搞定！

### Token认证与cache缓存

LuFeiProject项目

##### 1、创建认证类

```python
from rest_framework.authentication import BaseAuthentication
from rest_framework.exceptions import AuthenticationFailed
from ..models import Token
import datetime
import pytz
from django.core.cache import cache

class Loginauth(BaseAuthentication):
    def authenticate(self,request):
        """
        1、对token设置有效时间
        2、缓存存储
        :param request:
        :return:
        """
        # user_token = request.query_params.get
        token = request.META.get("HTTP_AUTHORIZATION")
        token_obj = Token.objects.filter(key=token).first()

    # 1 校验是否存在token字符串
        # 1、缓存校验
        user = cache.get(token)
        if user:
            print("缓存校验成功")
            return user,token

        # 数据库校验
        # 检验是否存在token字符串
        if not token_obj:
            raise AuthenticationFailed("认证失败")
        else:
            # 校验是否存在有效期内
            now = datetime.datetime.now()
            now = now.replace(tzinfo=pytz.timezone('UTC'))
            delta = now - token_obj.created
            state = delta <= datetime.timedelta(weeks=2)
            if state:
                # 校验成功，写入缓存中
                print("数据库校验成功")
                delta = datetime.timedelta(weeks=2) - delta
                				               cache.set(token_obj.key,token_obj.user,min(delta.total_seconds(),3600*24*7))
                return token_obj.user, token_obj.key
            else:
                raise AuthenticationFailed("认证超时")
```

##### 2、指定认证

```python
class CourseView(ModelViewSet):
    # 指定认证类 Loginauth
    authentication_classes = [Loginauth]
    queryset = Course.objects.all()
    serializer_class = CourseSerializer

class CouserDetailView(ModelViewSet):
    authentication_classes = [Loginauth]
    queryset = CourseDetail.objects.all()
    serializer_class = CourseDetailSerializer
```

### 缓存(redis)

博客地址：https://www.cnblogs.com/wupeiqi/articles/5132791.html

redis是一个非关系型（No-Sql）数据库,对比另一款非关系型数据库memcache,

共同点：

​	1、属于key-value,

​	2、将数据缓存到内存中，而不是磁盘上

不同点：

​	1、redis可以做持久化（可以将数据保存磁盘上）

​	2、支持更丰富的数据类型，key只能存储字符串，value可以是字符串，链表，哈希

