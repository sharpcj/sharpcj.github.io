---
layout: post
title: "Django 分页查询并返回jsons数据，中文乱码解决方法"
date:  2018-08-01 23:25:12 +0800
categories: ["技术", "编程", "Python"]
tag: ["Python", "django"]
---

# 一、引子
Django 分页查询并返回 json ，需要将返回的 queryset 序列化， demo 如下：

```
# coding=UTF-8

import os

from django.core import serializers
from django.core.paginator import Paginator, PageNotAnInteger, EmptyPage
from django.shortcuts import render
from django.http import HttpResponse
from mypage.models import Product


# Create your views here.


def getAllProducts(request):
    products_list = Product.objects.all()
    paginator = Paginator(products_list, 10)  # Show 10 products per page
    page = request.GET.get('page', 0)
    try:
        products = paginator.page(page)
    except PageNotAnInteger:
        # If page is not an integer, deliver first page.
        products = paginator.page(10)
    except EmptyPage:
        # If page is out of range (e.g. 9999), deliver last page of results.
        products = paginator.page(paginator.num_pages)

    json_data = serializers.serialize("json", products, ensure_ascii=False)
    return HttpResponse(json_data, content_type='application/json; charset=utf-8')
```

很容易出现的一个错误是中文乱码,重点在于 `json_data = serializers.serialize("json", products, ensure_ascii=False)` 中第三个参数。

# 二、Serialize----序列化django对象
官方文档原文：[https://docs.djangoproject.com/en/2.1/topics/serialization/](https://docs.djangoproject.com/en/2.1/topics/serialization/)

django的序列化框架提供了一个把django对象转换成其他格式的机制，通常这些其他的格式都是基于文本的并且用于通过一个管道发送django对象，但一个序列器是可能处理任何一个格式的（基于文本或者不是）

django的序列化类位于django.core下面的serializers文件夹里面，base.py文件里面定义了序列器和反序列器的基类以及一些异常，init.py文件定义了如何根据格式来选择对应的序列器等内容，我们一起来看看吧

init.py 和 base.py 文件的函数原型如下图

```
def serialize(format, queryset, **options):
"""Serialize a queryset (or any iterator that returns database objects) using
a certain serializer."""
s = get_serializer(format)()
s.serialize(queryset, **options)
return s.getvalue()
```

```
class Serializer(object):
   """    Abstract serializer base class.    """
   # Indicates if the implemented serializer is only available for
   # internal Django use.
   internal_use_only = False
   def serialize(self, queryset, **options):
```

那下面我们开始正式讲解django的序列化操作了

## 序列化数据
在最高层的api，序列化数据是非常容易的操作，看上面的函数可知，serialize函数接受一个格式和queryset，返回序列化后的数据：

简单的写法：

```
from django.core import serializers
data = serializers.serialize("xml", SomeModel.objects.all())
```

复杂的写法：

```
XMLSerializer = serializers.get_serializer("xml")
xml_serializer = XMLSerializer()
xml_serializer.serialize(queryset)
data = xml_serializer.getvalue()
```

## 反序列化数据
一样的简单，接受一个格式和一个数据流，返回一个迭代器

```
for obj in serializers.deserialize("xml", data):
    do_something_with(obj)
```

然而，deserialize返回的的是不是简单的django类型对象，而是DeserializedObject实例，并且这些实例是没有保存的，请使用DeserializedObject.save()方法把这些数据保存到数据库

## 序列化格式
django之处很多的序列化格式，有些需要你安装第三方支持的模块，xml，json和yaml是默认支持的

注意事项
如果你是使用utf-8或者其他的非ascii编码数据，然后用json序列器，注意穿一个ensure_ascii参数进去，否则输出的编码将会不正常

```
json_serializer = serializers.get_serializer("json")()
json_serializer.serialize(queryset, ensure_ascii=False, stream=response)
```

## 序列化参数
序列化的是是可以接受额外的参数的，总共有三个参数，如下：

```
self.stream = options.pop("stream", StringIO())
self.selected_fields = options.pop("fields", None)
self.use_natural_keys = options.pop("use_natural_keys", False)
```

## stream
将序列化后的数据输出到该stream流中，接上面的复杂的写法：

```
out = open("file.xml", "w")
xml_serializer.serialize(SomeModel.objects.all(), stream=out)
```

## selected_field
选择序列化的属性，通过制定fields参数，fields是一个元组参数，元素是选择要序列化的属性

```
from django.core import serializers
data = serializers.serialize('xml', SomeModel.objects.all(), fields=('name','size'))
```

## use_natural_keys
是否使用自然的关键字，默认是false（即是使用主键）

默认的外键和多对多关系序列化策略是使用主键，一般情况下是很好地，但有些情况下就不是这样了。比如外键到ContentType的时候，由于ContentType是django的数据库进程同步的时候自动产生的，它们的关键字不是那么容易去预测的。

一个整数id也不总是最方便的索引到一个对象的方法，所以基于这些情况，django提供了`use_natural_keys` 这个参数，

一个natural key是一个可以不使用主键就可以用来区分一个元素的属性组合的元组

### natural keys的反序列化
考虑这两个模型

```
from django.db import models
class Person(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    birthdate = models.DateField()
    class Meta:
        unique_together = (('first_name', 'last_name'),)
class Book(models.Model):
    name = models.CharField(max_length=100)
    author = models.ForeignKey(Person)
```

默认Book 的序列化数据将会使用一个整数索引到一个作者，例如，用json的是，一个Book的序列化数据大概是这样的，42是外键Author的主键

```
{
    "pk": 1,
    "model": "store.book",
    "fields": {
        "name": "Mostly Harmless",
        "author": 42
    }
}
```

但这不是一个很好的方法，不是吗？你需要知道这个主键代表到底是哪个Author，并且要求这个主键是稳定和可预测的。所以，我们可以增加一个natural key的处理函数，请在对应模型的管理模型里面定义一个get_by_natural_key方法，例如：

```
from django.db import models
class PersonManager(models.Manager):
    def get_by_natural_key(self, first_name, last_name):
        return self.get(first_name=first_name, last_name=last_name)
class Person(models.Model):
    objects = PersonManager()
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    birthdate = models.DateField()
    class Meta:
        unique_together = (('first_name', 'last_name'),)
```

这样之后，序列化的结果大概是这样的：

```
{
    "pk": 1,
    "model": "store.book",
    "fields": {
        "name": "Mostly Harmless",
        "author": ["Douglas", "Adams"]
    }
}
```

## natural keys的序列化
如果你想在序列化的时候使用natural key，那你必须在被序列化的模型里面顶一个natural_key方法，并在序列化的时候使用use_natural_keys=True属性如下：

```
class Person(models.Model):
    objects = PersonManager()
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    birthdate = models.DateField()
    def natural_key(self):
        return (self.first_name, self.last_name)
    class Meta:
        unique_together = (('first_name', 'last_name'),)
```

`serializers.serialize('json', [book1, book2], use_natural_keys=True)`

注意：natural_key() 和 get_by_natural_key() 不是同时定义的，如果你只想重载 natural keys 的能力，那么你不必定义 natural_key() 方法；同样，如果你只想在序列化的时候输出这些natural keys，那么你不必定义 get_by_natural_key() 方法

## 序列化过程中的依赖关系
因为 natural keys 依赖数据库查询来解析引用，所以在数据被引用之前必须确保数据是存在的。看下面的例子，如果一个 Book 的 natural key 是书名和作者的组合，你可以这样写：

```
class Book(models.Model):
    name = models.CharField(max_length=100)
    author = models.ForeignKey(Person)

    def natural_key(self):
        return (self.name,) + self.author.natural_key()
```

那么问题来了，如果Author还没有被序列化呢？很明显，Author应该在Book之前被序列化，为此，我们可以添加一个依赖关系如下：

```
def natural_key(self):
    return (self.name,) + self.author.natural_key()
natural_key.dependencies = ['example_app.person']
```

这保证了Person对象是在Book对象之前被序列化的，同样，任何一个引用Book的对象只有在Person和Book对象都被序列化之后才会被序列化

## 继承的模型
如果是使用抽象继承的时候，不必在意这个问题；如果你使用的是多表继承，那么注意了：必须序列化所有的基类，例如：

```
class Place(models.Model):
    name = models.CharField(max_length=50)
class Restaurant(Place):
    serves_hot_dogs = models.BooleanField()
```

如果仅仅序列化 Restaurant 模型，那么只会得到一个 serves_hot_dog 属性，基类的属性将被忽略，你必须同时序列化所有的继承的模型，如下：

```
all_objects = list(Restaurant.objects.all()) + list(Place.objects.all())
data = serializers.serialize('xml', all_objects)
```