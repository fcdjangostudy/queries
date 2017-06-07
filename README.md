# Django Queries

## 질문리스트
* QuerySet evaluated? evaluation


model class = DATABASE TABLE
model instancs = RECORD


### 예제코드

```python
from django.db import models

class Blog(models.Model):
    name = models.CharField(max_length=100)
    tagline = models.TextField()

    def __str__(self):              # __unicode__ on Python 2
        return self.name

class Author(models.Model):
    name = models.CharField(max_length=200)
    email = models.EmailField()

    def __str__(self):              # __unicode__ on Python 2
        return self.name

class Entry(models.Model):
    blog = models.ForeignKey(Blog)
    headline = models.CharField(max_length=255)
    body_text = models.TextField()
    pub_date = models.DateField()
    mod_date = models.DateField()
    authors = models.ManyToManyField(Author)
    n_comments = models.IntegerField()
    n_pingbacks = models.IntegerField()
    rating = models.IntegerField()

    def __str__(self):              # __unicode__ on Python 2
        return self.headline
```


### update 하기(추가하기)

```python
>>> from blog.models import Blog
>>> b = Blog(name='Beatles Blog', tagline='All the latest Beatles news.')
>>> b.save()
>>> 
>>> b5.name = 'New name'
>>> b5.save()
```

#### ForeignKey업데이트도 동일

#### ManyToMany field는 add()메서드 활용


## 검색하기
 Manager를 통해서 QuerySet 생성하기 
 따로 이름을 정해주지 않으면 Manager의 이름은 objects이다.
 The Manager is the main source of QuerySets for a model.

* all()
* filter(**kwargs)
* exclude(**kwargs)

검색 결과 또한 QuerySet 그 자체이므로 같은 메서드를 반복적으로 사용할 수 있다.

```	python
>>> Entry.objects.filter(
...     headline__startswith='What'
... ).exclude(
...     pub_date__gte=datetime.date.today()
... ).filter(
...     pub_date__gte=datetime(2005, 1, 30)
... )
```

* get() : 한개만 가져오기 filter와 달리 DoesNotExist 에러를 발생시킴
* 


이러한 메서드는 QuerySets을 evaluation 하지는 않는다.

## Limiting QuerySets(쿼리셋 자르기)

```python
>>> Entry.objects.all()[:5]
```
위와 같이 슬라이스를 사용한다. 

그러나 음수 슬라이싱(```Entry.objects.all()[-1]```의 형태)은 허용하지 않는다

## Field lookups
```python
>>> Entry.objects.filter(pub_date__lte='2006-01-01')
```
=
```python
SELECT * FROM blog_entry WHERE pub_date <= '2006-01-01';
```
밑에 이걸 LIKE문이라고 한다

* exact
* iexact
* contains
* startswith
* endswith

### 관계 확장 후 조회하기
: 밑줄 두개로 구분

```
>>> Blog.objects.filter(entry__headline__contains='Lennon')
```
: entry 테이블 안의 headline에 Lennon이 포함되었는지 를 검사하여 필터링함.

이 필터는 오류를 내뱉지 않고 없으면 null을 반환
따라서
```python
Blog.objects.filter(entry__authors__name__isnull=True)
```
is_null을 사용

### filter의 작동
```python
Blog.objects.filter(entry__headline__contains='Lennon', entry__pub_date__year=2008)
```
위는 필터가 and옵션으로 작용되어 두 조건이 모두 만족하는 결과 쿼리셋만 반환함.

or 연산을 하기 위해서는

```python
Blog.objects.filter(entry__headline__contains='Lennon').filter(entry__pub_date__year=2008)
```
위와 같이 병렬로 나열해야함.

exclude와 적절히 병행해 가면서 사용하면 원하는 결과를 충분히 찾을 수 있음.

## 다른 필드의 값과 비교하기  F expressions

F()의 인스턴스는 쿼리내의 모델필드에 대한 참조로 작동한다.

```python
>>> from django.db.models import F
>>> Entry.objects.filter(n_comments__gt=F('n_pingbacks'))
```

사칙연산 가능하고 이중밑줄치면 다른 테이블의 인스턴스로도 확장가능하다.

## pk로 자유롭게 조회 가능

## 장고의 자동 이스케이프
 % 와 \ 이런것 


## 장고 쿼리셋 캐싱문제
일반적으로 쿼리셋을 생성하면 캐시에 저장되는 데 이는 임시저장소 이기때문에 따로부르면 부를때마다 작동된다.(아래와 같이)

```python
>>> print([e.headline for e in Entry.objects.all()])
>>> print([e.pub_date for e in Entry.objects.all()])
```
이렇게 되면 임시저장소는 불안정적이기 때문에 값이 날아가거나 변형될 여지가 있다. 따라서 

```python
>>> queryset = Entry.objects.all()
>>> print([p.headline for p in queryset]) # Evaluate the query set.
>>> print([p.pub_date for p in queryset]) # Re-use the cache from the evaluation.
```
위와 같이 저장후 사용한다.

## 복잡한 조회 Q objects

각각의 이중밑줄조건등을 and or연산을 할 수 있도록 해줌


or연산
```python
Q(question__startswith='Who') | Q(question__startswith='What')
```

and연산

```python
Q(question__startswith='Who') & Q(question__startswith='What')
```

## 모델 인스턴스 복사하기

pk를 None으로 만들기

```python
blog = Blog(name='My blog', tagline='Blogging is easy')
blog.save() # blog.pk == 1

blog.pk = None
blog.save() # blog.pk == 2
```

## 한꺼번에 여러 객체 업데이트 하기

```python
Entry.objects.filter(pub_date__year=2007).update(headline='Everything is the same')
```
위는 필터로 찾은 모든 객체에 대하여 헤드라인 변경


### 비관계 필드는 새 상수값을 update하면 되고

### ForeignKey는 새 모델 인스턴스를 인자로 할당해야 한다.

```python
>>> b = Blog.objects.get(pk=1)

# Change every Entry so that it belongs to this Blog.
>>> Entry.objects.all().update(blog=b)
```

**업데이트되는 QuerySet에 대한 유일한 제한은 모델의 주 테이블 인 하나의 데이터베이스 테이블에만 액세스 할 수 있다는 것.

update () 메서드는 SQL 문으로 직접 변환됨



## Related objects¶

### One-to-many relationships¶

캐시관계 살펴보기

```python
>>> e = Entry.objects.get(id=2)
>>> print(e.blog)  # Hits the database to retrieve the associated Blog.
>>> print(e.blog)  # Doesn't hit the database; uses cached version.
>>> 

>>> e = Entry.objects.select_related().get(id=2)
>>> print(e.blog)  # Doesn't hit the database; uses cached version.
>>> print(e.blog)  # Doesn't hit the database; uses cached version.
```
위와 같이 cach한다.


#### 역참조: Manager
this Manager is named FOO_set, FOO is the source model name, lowercased. 

이를 따로 related_name 변수를 통해서 다시 재정의 가능

Manager의 여러 기능들

* add(obj1, obj2, ...)
* create(**kwargs)
* remove(obj1, obj2, ...)
* clear()
* set(objs)

### Many-to-many relationships
역시  ManyToManyField_relatedManager를 가지고 있다. 이를 통해서 역참조 하면 됨.


#### 역방향으로 참조하는 것은 settings의 INSTALLED_APPS에서 정의된다. 따라서 여기에 추가해 주는 것이 중요하다.







