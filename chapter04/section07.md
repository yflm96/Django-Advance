# 查询操作

查找是数据库操作中一个非常重要的技术。查询一般就是使用 `filter `、 `exclude `以及 `get `三个方法来实现。我们可以在调用这些方法的时候传递不同的参数来实现查询需求。在 `ORM `层面，这、些查询条件都是使用 `field + __ + condition` 的方式来使用的。以下将那些常用的查询条件来一一解释。


## 查询条件

字段查询是指如何指定`SQL WHERE`子句的内容。它们用作`QuerySet`的`filter()，exclude()和get()`方法的关键字参数

|字段查询参数|说明|
|---|---|
|exact|	精确匹配|
|iexact|	不区分大小写的精确匹配|
|contains|	包含匹配|
|icontains|	不区分大小写的包含匹配|
|in|	在..之内的匹配|
|gt|	大于|
|gte|	大于等于|
|lt|	小于|
|lte|	小于等于|
|startswith|	从开头匹配|
|istartswith| 不区分大小写从开头匹配|
|endswith|	从结尾处匹配|
|iendswith|	不区分大小写从结尾处匹配|
|range|	范围匹配|
|date|	日期匹配|
|year|	年份|
|month|	月份|
|day|	日期|
|week|	第几周|
|week_day|	周几|
|time|	时间|
|hour|	小时|
|minute|	分钟|
|second|	秒|
|isnull|	判断是否为空|
|search|	1.10中被废弃|
|regex|	区分大小写的正则匹配|
|iregex|	不区分大小写的正则匹配|

### exact

使用精确的`= `进行查找。如果提供的是一个 `None `，那么在 `SQL `层面就是被解释为 `NULL `。示例代码如下：
```python
    article = Article.objects.get(id__exact=14)
    article = Article.objects.get(id__exact=None)
```
以上的两个查找在翻译为 SQL 语句为如下：
```sql
    select ... from article where id=14;
    select ... from article where id IS NULL;
```

如果使用`filter`则返回的是`QuerySet`，如果使用的是`get`则返回的是`Model`，没有`query`属性。
访问`QuerySet`实例的`query`属性，可以查看`ORM`将些查询转换的`SQL`。
```python
def index(request):
    article = Article.objects.filter(id__exact=1)
    print(article)
    print('*'*30)
    print(article.query)
    return HttpResponse("Success!")
    
# 显示的结果如下：
<QuerySet [<Article: 港珠澳大桥>]>
[25/Oct/2018 09:51:47] "GET / HTTP/1.1" 200 8
******************************
SELECT `article_article`.`id`, `article_article`.`title`, `article_article`.`content` FROM `article_article` WHERE `article_article`.`id` = 1
```

**注意**
在Windows操作系统上，`MySQL`的排序规则`collation`无论是什么都是大小写不敏感的。
在Linux操作系统上，`MySQL`的排序规则 `collation`是`utf8_bin`，那么是大小写敏感的。

### iexact

使用 `like `进行查找。示例代码如下：
```python
    article = Article.objects.filter(title__iexact='hello world')
```
那么以上的查询就等价于以下的 `SQL `语句：
```sql
    select ... from article where title like 'hello world';
```
注意上面这个 `sql `语句，因为在 `MySQL `中，没有一个叫做 `ilike `的。所以 `exact `和 `iexact `的区别实际上就是 `LIKE `和 `= `的区别，在大部分 `collation=utf8_general_ci` 情况下都是一样的（ `collation `是用来对字符串比较的）。

> `LIKE`和`=`:大部分情况下都是等价的，只有少数情况下是不等价的。
> `exact`和`iexact`: 区别就是`=`和`LIKE`的区别，因为`exact`会被`ORM`转换成`=`,`iexact`会被`ORM`转换成`LIKE`。


### contains

**大小写敏感**，判断某个字段是否包含了某个数据。示例代码如下：
```python
    articles = Article.objects.filter(title__contains='hello')
```
在翻译成 `SQL `语句为如下：
```sql
    select ... where title like LIKE BINARY '%hello%';
```
要注意的是，在使用 `contains `的时候，翻译成的 `sql `语句左右两边是有百分号的，意味着使用的是模糊查询。而 `iexact `翻译成 `sql `语句左右两边是没有百分号的，意味着使用的是精确的查询。

### icontains

**大小写不敏感**的匹配查询。示例代码如下：
```python
    articles = Article.objects.filter(title__icontains='hello')
```
在翻译成 `SQL `语句为如下：
```sql
    select ... where title like '%hello%';
```

在`MySQL`中`LIKE BINARY`是**区别大小写**的，而`LIKE`是**不区别分大小写**的

### in

提取那些给定的 `field `的值是否在给定的容器中。容器可以为 `list `、 `tuple `或者任何一个可以迭代的对象，包括 `QuerySet `对象。示例代码如下：
```python
    articles = Article.objects.filter(id__in=[1,2,3])
```
以上代码在翻译成 `SQL `语句为如下：
```sql
    select ... where id in (1,3,4)
```
当然也可以传递一个 `QuerySet `对象进去。示例代码如下：
```python
    inner_qs = Article.objects.filter(title__contains='hello')
    categories = Category.objects.filter(article__in=inner_qs)
```
以上代码的意思是获取那些文章标题包含 `hello `的所有分类。将翻译成以下 `SQL `语句，示例代码如下：
```sql
    select ...from category where article.id in (select id from article where title like '%hello%');
```

如果在做反向过滤的时候，过滤的字段是模型的主键，可以直接省略主键的字段，`article__id__in`可以写成`article__in` 。

### gt

某个 `field `的值要大于给定的值。示例代码如下：
```python
    articles = Article.objects.filter(id__gt=4)
```
以上代码的意思是将所有 id 大于4的文章全部都找出来。将翻译成以下 SQL 语句：
```sql
    select ... where id > 4;
```

### gte

类似于 `gt `，是大于等于。


### lt

类似于 `gt `,是小于。

### lte

类似于 `gt`, 是小于等于。

### startswith

判断某个字段的值是否是以某个值开始的。**大小写敏感**。示例代码如下：
```python
    articles = Article.objects.filter(title__startswith='hello')
```
以上代码的意思是提取所有标题以 `hello `字符串开头的文章。将翻译成以下 `SQL `语句：
```sql
    select ... where title LIKE BINARY 'hello%';
```

### istartswith

类似于 `startswith `，但是大小写是不敏感的。


### endswith

判断某个字段的值是否以某个值结束。大小写敏感。示例代码如下：
```python
    articles = Article.objects.filter(title__endswith='world')
```
以上代码的意思是提取所有标题以 `world `结尾的文章。将翻译成以下 `SQL `语句：
```sql
    select ... where title LIKE BINARY  '%world';
```

### iendswith

类似于 `endswith `，只不过大小写不敏感。


### range

判断某个 `field `的值是否在给定的区间中。示例代码如下：
```python
    from django.utils.timezone import make_aware
    from datetime import datetime
    
    start_date = make_aware(datetime(year=2018,month=1,day=1))
    end_date = make_aware(datetime(year=2018,month=3,day=29,hour=16))
    articles = Article.objects.filter(pub_date__range=(start_date,end_date))
```
以上代码的意思是提取所有发布时间在 `2018/1/1` 到 `2018/12/12` 之间的文章。将翻译成以下的 `SQL `语句：
```sql
    select ... from article where pub_time between '2018-01-01' and '2018-12-12';
```
需要注意的是，以上提取数据，**不会包含最后一个值**。也就是不会包含 2018/12/12 的文章。而且另外一个重点，因为我们在 `settings.py` 中指定了 `USE_TZ=True` ，并且设置了 `TIME_ZONE='Asia/Shanghai'` ，因此我们在提取数据的时候要使用 `django.utils.timezone.make_aware` 先将 `datetime.datetime` 从 `navie `时间转换为 `aware `时间。 `make_aware `会将指定的时间转换为 `TIME_ZONE `中指定的时区的时间。`range`接受的参数时间为须为`aware`时间类型。

### date

针对某些 `date `或者 `datetime `类型的字段。可以指定 `date `的范围。并且这个时间过滤，还可以使用链式调用。示例代码如下：
```python
    articles = Article.objects.filter(pub_date__date=date(2018,3,29))
```
以上代码的意思是查找时间为 `2018/3/29` 这一天发表的所有文章。将翻译成以下的 sql 语句：
```sql
    select ... WHERE DATE(CONVERT_TZ(`front_article`.`pub_date`, 'UTC', 'Asia/Shanghai')) =2018-03-29
```
注意，因为默认情况下 `MySQL `的表中是没有存储时区相关的信息的。因此我们需要下载一些时区表的文件，然后添加到 `Mysql `的配置路径中。如果你用的是 `windows `操作系统。那么在 [http://dev.mysql.com/downloads/timezones.html](http://dev.mysql.com/downloads/timezones.html) 下载 `timezone_2018d_posix.zip - POSIX standard` 。然后将下载下来的所有文件拷贝到 `C:\ProgramData\MySQL\MySQL Server5.7\Data\mysql` 中，如果提示文件名重复，那么选择覆盖即可。
如果用的是 `linux `或者 `mac `系统，那么在命令行中执行以下命令： `mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -D mysql -u root -p` ，然后输入密码，从系统中加载时区文件更新到 `mysql `中。

### year

根据年份进行查找。示例代码如下：
```python
    articles = Article.objects.filter(pub_date__year=2018)
    articles = Article.objects.filter(pub_date__year__gte=2017)
```
以上的代码在翻译成 SQL 语句为如下:
```sql
    select ... where pub_date between '2018-01-01' and '2018-12-31';
    select ... where pub_date >= '2017-01-01';
```
### month

同`year`，根据月份进行查找。

### day

同`year`，根据日期进行查找。

### week_day

`Django 1.11` 新增的查找方式。同 `year `，根据星期几进行查找。1表示星期天，7表示星期六， 2-6 代表的是星期一到星期五。

### time

根据时间进行查找。示例代码如下：
```python
    articles = Article.objects.filter(pub_date__time=datetime.time(12,12,12));
```
以上的代码是获取每一天中12点12分12秒发表的所有文章。
如果涉及到秒的可以参考如下代码
```python
    from datatime import time
    
    s_time = time(15,46,25)
    e_time = time(15,46,26)
    articles = Article.objects.filter(create_time__time__range=(s_time,e_time))
```

更多的关于时间的过滤，请参考 Django 官方文档:
[https://docs.djangoproject.com/en/2.1/ref/models/querysets/#time](https://docs.djangoproject.com/en/2.1/ref/models/querysets/#time)

### isnull

根据值是否为空进行查找。示例代码如下：
```python
    articles = Article.objects.filter(pub_date__isnull=False)
```
以上的代码的意思是获取所有发布日期不为空的文章。将来翻译成 `SQL `语句如下：
```sql
    select ... where pub_date is not null;
```

### regex和iregex

大小写敏感和大小写不敏感的正则表达式。示例代码如下：
```python
    articles = Article.objects.filter(title__regex=r'^hello')
```
以上代码的意思是提取所有标题以 `hello `字符串开头的文章。将翻译成以下的 `SQL `语句：
```sql
    select ... where title regexp binary '^hello';
```
`iregex`是大小写不敏感的。


## 根据关联的表进行查询

假如现在有两个 `ORM `模型，一个是 `Article `，一个是 `Category `。代码如下：
```python
    class Category(models.Model):
    """文章分类表"""
        name = models.CharField(max_length=100)
        
    class Article(models.Model):
    """文章表"""
        title = models.CharField(max_length=100,null=True)
        category = models.ForeignKey("Category",on_delete=models.CASCADE)
```
比如想要获取文章标题中包含"hello"的所有的分类。那么可以通过以下代码来实现：
```python
    categories = Category.object.filter(article__title__contains("hello"))
```


## 聚合函数

Django的`django.db.models`模块提供聚合函数。

如果你用原生 `SQL `，则可以使用聚合函数来提取数据。比如提取某个商品销售的数量，那么可以使用 `Count `，如果想要知道商品销售的平均价格，那么可以使用 `Avg `。
聚合函数是通过 `aggregate`方法来实现的。在讲解这些聚合函数的用法的时候，都是基于以下的模型对象来实现的。
```python
    from django.db import models
    class Author(models.Model):
        """作者模型"""
        name = models.CharField(max_length=100)
        age = models.IntegerField()
        email = models.EmailField()
    
        class Meta:
        db_table = 'author'
    
    class Publisher(models.Model):
        """出版社模型"""
        name = models.CharField(max_length=300)
        
        class Meta:
            db_table = 'publisher'
            
    class Book(models.Model):
        """图书模型"""
        name = models.CharField(max_length=300)
        pages = models.IntegerField()
        price = models.FloatField()
        rating = models.FloatField()
        author = models.ForeignKey(Author,on_delete=models.CASCADE)
        publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE)
        
        class Meta:
        db_table = 'book'
        
    class BookOrder(models.Model):
        """图书订单模型"""
        book = models.ForeignKey("Book",on_delete=models.CASCADE)
        price = models.FloatField()
        
        class Meta:
        db_table = 'book_order'
```
1. `Avg `：求平均值。比如想要获取所有图书的价格平均值。那么可以使用以下代码实现。
```python
    from django.db.models import Avg
    
    result = Book.objects.aggregate(Avg('price'))
    print(result)
```
以上的打印结果是：
```python
    {"price__avg":23.0}
```
其中 `price__avg `的结构是根据 `field__avg `规则构成的。如果想要修改默认的名字，那么可以将 `Avg `赋值给一个关键字参数。示例代码如下：
```python
    from django.db.models import Avg
    
    result = Book.objects.aggregate(my_avg=Avg('price'))
    print(result)
```
那么以上的结果打印为:
```python
    {"my_avg":23}
```
2. Count ：获取指定的对象的个数。示例代码如下：
```python
    from django.db.models import Count
    
    result = Book.objects.aggregate(book_num=Count('id'))
```
以上的 `result `将返回 `Book `表中总共有多少本图书。`Count `类中，还有另外一个参数叫做 `distinct `，默认是等于 `False `，如果是等于 `True `，那么将去掉那些重复的值。比如要获取作者表中所有的不重复的邮箱总共有多少个，那么可以通过以下代码来实现：
```python
    from djang.db.models import Count
    
    result = Author.objects.aggregate(count=Count('email',distinct=True))
```
3. `Max `和 `Min `：获取指定对象的最大值和最小值。比如想要获取 `Author `表中，最大的年龄和最小的年龄分别是多少。那么可以通过以下代码来实现：
```python
    from django.db.models import Max,Min
    result = Author.objects.aggregate(Max('age'),Min('age'))
```
如果最大的年龄是88,最小的年龄是18。那么以上的result将为：
```python
    {"age__max":88,"age__min":18}
```
4. `Sum `：求指定对象的总和。比如要求图书的销售总额。那么可以使用以下代码实现：
```python
    from djang.db.models import Sum
    result = Book.objects.annotate(total=Sum("bookstore__price")).values("name","total")
```
以上的代码 `annotate `的意思是给 `Book `表在查询的时候添加一个字段叫做 `total `，这个字段的数据来源是从 `BookStore `模型的 `price `的总和而来。 `values `方法是只提取 `name `和 `total `两个字段的值。

更多的聚合函数请参考官方文档:
[https://docs.djangoproject.com/en/2.1/ref/models/querysets/#aggregation-functions](https://docs.djangoproject.com/en/2.1/ref/models/querysets/#aggregation-functions)


## aggregate和anotate的区别

1. `aggregate `：返回使用聚合函数后的字段和值。
2. `annotate `：在原来模型字段的基础之上添加一个使用了聚合函数的字段，并且在使用聚合函数的时候，会使用当前这个模型的主键进行分组（group by）。比如以上 `Sum `的例子，如果使用的是 `annotate `，那么将在每条图书的数据上都添加一个字段叫做 `total `，计算这本书的销售总额。而如果使用的是 `aggregate `，那么将求所有图书的销售总额。


## F表达式和Q表达式


### F表达式

到目前为此的例子中，我们都是将模型字段与常量进行比较。但量，如果想要将模型的一个字段与同一个模型的另外一外字段进行比较该怎么办？
使用Django提供了`F表达式`

`F表达式` 是用来优化 ORM 操作数据库的。比如我们要将公司所有员工的薪水都增加1000元，如果按照正常的流程，应该是先从数据库中提取所有的员工工资到`Python`内存中，然后使用`Python`代码在员工工资的基础之上增加1000元，最后再保存到数据库中。这里面涉及的流程就是，首先从数据库中提取数据到`Python`内存中，然后在`Python`内存中做完运算，之后再保存到数据库中。示例代码如下：
```python
    employees = Employee.objects.all()
    for employee in employees:
    employee.salary += 1000
    employee.save()
```
而我们的 `F表达式`就可以优化这个流程，他可以不需要先把数据从数据库中提取出来，计算完成后再保存回去，他可以直接执行 `SQL`语句 ，就将员工的工资增加1000元。示例代码如下：
```python
    from djang.db.models import F
    Employee.object.update(salary=F("salary")+1000)
```
`F表达式` 并不会马上从数据库中获取数据，而是在生成 `SQL `语句的时候，动态的获取传给 `F表达式` 的值。
比如如果想要获取作者中， `name `和 `email `相同的作者数据。如果不使用 `F表达式` ，那么需要使用以下代码来完成：
```python
    authors = Author.objects.all()
    for author in authors:
        if author.name == author.email:
            print(author)
```
如果使用 `F表达式` ，那么一行代码就可以搞定。示例代码如下：
```python
    from django.db.models import F
    authors = Author.objects.filter(name=F("email"))
```

### Q表达式

普通`filter`函数里的条件都是“and”逻辑，如果你想实现“or”逻辑怎么办？用Q查询！
`Q`来自`django.db.models.Q`，用于封装关键字参数的集合，可以作为关键字参数用于`filter`、`exclude`和`get`等函数。

如果想要实现所有价格高于100元，并且评分达到9.0以上评分的图书。那么可以通过以下代码来实现：
```python
    books = Book.objects.filter(price__gte=100,rating__gte=9)
```
以上这个案例是一个并集查询，可以简单的通过传递多个条件进去来实现。但是如果想要实现一些复杂的查询语句，比如要查询所有价格低于10元，或者是评分低于9分的图书。那就没有办法通过传递多个条件进去实现了。这时候就需要使用 `Q表达式` 来实现了。示例代码:
```python
    from django.db.models import Q
    books = Book.objects.filter(Q(price__lte=10) | Q(rating__lte=9))
```
以上是进行或运算，当然还可以进行其他的运算，比如有 `& `和 `~`（非） 等。一些用 `Q表达式`的例子如下：
```python
    from django.db.models import Q
    
    # 获取id等于3的图书
    books = Book.objects.filter(Q(id=3))
    # 获取id等于3，或者名字中包含文字"记"的图书
    books = Book.objects.filter(Q(id=3)|Q(name__contains("记")))
    # 获取价格大于100，并且书名中包含"记"的图书
    books = Book.objects.filter(Q(price__gte=100)&Q(name__contains("记")))
    # 获取书名包含“记”，但是id不等于3的图书
    books = Book.objects.filter(Q(name__contains='记') & ~Q(id=3))
```

### 缓存与查询集

每个`QuerySet`都包含一个缓存，用于减少对数据库的实际操作。理解这个概念，有助于你提高查询效率。

对于新创建的`QuerySet`，它的缓存是空的。当QuerySet第一次被提交后，数据库执行实际的查询操作，Django会把查询的结果保存在`QuerySet`的缓存内，随后的对于该`QuerySet`的提交将重用这个缓存的数据。

要想高效的利用查询结果，降低数据库负载，你必须善于利用缓存。看下面的例子，这会造成2次实际的数据库操作，加倍数据库的负载，同时由于时间差的问题，可能在两次操作之间数据被删除或修改或添加，导致脏数据的问题：
```python
>>> print([e.headline for e in Entry.objects.all()])
>>> print([e.pub_date for e in Entry.objects.all()])
```
为了避免上面的问题，好的使用方式如下，这只产生一次实际的查询操作，并且保持了数据的一致性：
```python
>>> queryset = Entry.objects.all()
>>> print([p.headline for p in queryset]) # 提交查询
>>> print([p.pub_date for p in queryset]) # 重用查询缓存
```
何时不会被缓存

有一些操作不会缓存QuerySet，例如切片和索引。这就导致这些操作没有缓存可用，每次都会执行实际的数据库查询操作。例如：
```python
>>> queryset = Entry.objects.all()
>>> print(queryset[5]) # 查询数据库
>>> print(queryset[5]) # 再次查询数据库
```
但是，如果已经遍历过整个QuerySet，那么就相当于缓存过，后续的操作则会使用缓存，例如：
```python
>>> queryset = Entry.objects.all()
>>> [entry for entry in queryset] # 查询数据库
>>> print(queryset[5]) # 使用缓存
>>> print(queryset[5]) # 使用缓存
```
下面的这些操作都将遍历QuerySet并建立缓存：
```python
>>> [entry for entry in queryset]
>>> bool(queryset)
>>> entry in queryset
>>> list(queryset)
```
注意：简单的打印QuerySet并不会建立缓存，因为__repr__()调用只返回全部查询集的一个切片。