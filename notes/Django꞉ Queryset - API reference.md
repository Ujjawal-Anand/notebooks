---
tags: [aggregation, django, queryset]
title: 'Django: Queryset - API reference'
created: '2020-04-23T07:17:42.626Z'
modified: '2020-04-26T16:57:04.600Z'
---

# Django: Queryset - API reference

## F() expression

An F() object makes it possible to refer to model field values and perform action on them without actually having pull them out of database in python memory.

Example :

Normal way
```python
book = Book.objects.get(name="Django QuerySet for Dummies")
book.price += 10.02
book.save()
```

Here the value of book.price is getting pulled from python database into python memory and being manipulated using python operator and saved back to database. 

Using F()

```python
from django.db.models import F

book = Book.objects.get(name="Django QuerySet for Dummies")
book.price = F('price') +1
book.save()

# last two queries can be reduced as one by
book.update(price=F('price')+1)

# update to increment the field value on multiple objects
Book.objects.all().update(price=F('price')+1)
```

When Django encounters an instance of F(), it overrides the standard Python operators to create an encapsulated SQL expression; in this case, one which instructs the database to increment the database field represented by reporter.stories_filed.

Whatever value is or was on reporter.stories_filed, Python never gets to know about it - it is dealt with entirely by the database. All Python does, through Django’s F() class, is create the SQL syntax to refer to the field and describe the operation.

To access the new value saved this way, the object must be reloaded:

```python
book = Book.objects.get(pk=book.pk)

# or more succinctly:
book.refresh_from_db()
```

F() therefore can offer performance advantages by:

- getting the database, rather than Python, to do work
- reducing the number of queries some operations require

### Race Condition
If two Python threads execute the code in the first example above, one thread could retrieve, increment, and save a field’s value after the other has retrieved it from the database. The value that the second thread saves will be based on the original value; the work of the first thread will be lost.

If the database is responsible for updating the field, the process is more robust: it will only ever update the field based on the value of the field in the database when the save() or update() is executed, rather than based on its value when the instance was retrieved.

### F() assignment persists after Model.save()

```python
book = Book.objects.get(name="Django QuerySet for Dummies")
book.price = F('price')+10
book.save()

book.name = "Django QuerySet for Smarts"
book.save()

```
price will be updated twice in this case. If it's initially 30, the final value will 50. This persistence can be avoided by reloading the model object just after saving it, by using refresh_from_db()

### Using F() in querysets

```python
# To find all the blog entries with more than twice as many comments as pingbacks,
>>> Entry.objects.filter(number_of_comments__gt=F('number_of_pingbacks')*2)

# find all the entries where the rating of the entry is less than the sum of 
# the pingback count and comment count
>>> Entry.objects.filter(ratings__lt=F('number_of_pingbacks') + F('number_of_comments'))

# An F() object with a double underscore will introduce any joins needed to access the related object. 
# retrieve all the entries where the author’s name is the same as the blog name

>>> Entry.objects.filter(authors__name=F('blog__name'))

#For date and date/time fields, you can add or subtract a timedelta object. 
# Return all entries that were modified more than 3 days after they were published
>>> Entry.objects.filter(mod_date__gt=F('pub_date') + timedelta(days=3))

# The F() objects support bitwise operations by .bitand(), .bitor(), 
# .bitrightshift(), and .bitleftshift(). For example:

>>> F('somefield').bitand(16)
```



## Aggregation

Sometimes you will need to retrieve values that are derived by summarizing or aggregating a collection of objects, that's where aggregation comes:

Example :

*models.py*
```python
from django.db import models

class Author(models.Model):
  name = models.CharField(max_lenght=100)
  age = models.IntegerField()

class Publisher(models.Model):
  name = models.CharField(max_length=100)

class Book(models.Model):
  name = models.CharField(max_lenght=100)
  pages = models.IntgerField()
  price = models.DecimalField(max_digits=10, decimal_places=2)
  ratings = models.FLoatField()
  authors = models.ManyToManyField(Author)
  publisher = models.ForeignKey(Publisher, on_delete=models.CASCADE)
  pub_date = models.DateField()

class Store(models.Model):
  name = models.CharField(max_length=100)
  books = models.ManyToManyField(Book)
```

*querysets*
```python
# Total number of books
>>> Book.objects.count()
2452

# Total number of books from publisher "Harper Collins"
>>> Book.objects.filter(publisher__name="Harper Collins").count()
60

# Average price across all books
>>> from django.db.models import Avg
>>> Books.objects.all().aggregate(Avg('price))
{'price__avg': 30.34}

# Max price across all book
>>> from django.db.models import Max
>>> Books.objects.all().aggregate(Max('price'))
{'price__max': Decimal(60.34)}

# Difference between highest priced book and average price of all books
>>> from django.db.models import FloatField
>>> Books.objects.all.aggregate(price_diff = Max('price', output_field=FloatField) - Avg('price'))
{'price_diff': 29.66}
```

## Annotate

Annotate is used to generate per-object summeries (unlike aggreagate which calculates such reports over collection of objects) 

```python
# All the following queries involve traversing the Book <-> Publisher
# foreign key relationship backwards

# Each publisher, each with count of books as a "num_books" attribute.
>>> from django.db.models import Count
>>> pubs = Publisher.objects.annotate(num_books=Count('book'))
>>> pubs
<QuerySet [<Publisher: BaloneyPress>, <Publisher: SalonyPress>, ...]>
>>> pubs[0].num_books
73

>>> q = Blog.objects.annotate(Count('entry'))
>>> q[0].name
'Hello World'
# The number of entries on first blog
>>> q[0].entry__count # focus on name of generated alias 
42

# Each publisher, with a separate count of books with a rating above
# and below 5
>>> from django.db.models import Q
>>> above_5 = Count('book', filter=Q(book__rating__gt=5))
>>> below_5 = Count('book', filter=Q(book_rating__lte=5))
>>> pubs = Publisher.objects.annotate(below_5=below_5)
            .annotate(above_5=above_5)
>>> pubs[0].above_5
23
>>> pubs[0].below_5
12
```

Each argument to annotate() is an annotation that will be added to each object in the Queryset that is returned.

Annotations specified using keyword argument will use the keyword as the alias for the annotation.

Anonymous arguments will have an alias generated for them based upon the name of the aggregate function and the model field that is being generated (see the Blog example in above)

[see](https://docs.djangoproject.com/en/3.0/topics/db/aggregation/) for more detail on aggregation.

## order_by()
```python
# will be ordered by pub_date descending then by headline ascending
# The negative sign in front of "-pub_date" indicates descending order
>>> Entry.objects.filter(pub_date__year=2019).order_by('-pub_date', 'headline')

# To order randomly, use "?"
>>> Entry.objects.order_by("?")
# this can be expensive and slow, depending on database backend you're using

# order by a field in a different model
>>> Entry.objects.order_by('blog__name', 'headline')

# django will use default ordering on the related model
>>> Entry.objects.order_by('blog')
# identical to
>>> Entry.objects.order_by('blog__id')

# You can also order by query expressions by calling asc() or desc() on
# the expression
>>> Entry.objects.order_by(Coalesce('summary', 'headline').desc())
# asc() and desc() have arguments (null_first and null_last) that controls
# how null values are sorted

>>> Entry.objcts.order_by(Lower('headline').desc())

# each order_by() call will clear any previous ordering
# ordered by pub_date and not headline
>>> Entry.objects.order_by('headline').order_by('pub_date')
```

## reverse()

Use the reverse() method to reverse the order in which a queryset's element are returned.
```python
# to retrieve "last" five items in a queryset
>>> my_queryset.reverse()[:5]
```

## distinct()

If your query spans multiple tables, it's possible to get duplicate results when a QuerySet is evaluated. That's when you'd use distinct()

Example (those afte the first will only work on PostgresSQL)
```python
>>> Author.objects.distinct()

>>> Entry.objects.order_by('pub_date').distinct('pub_date')

>>> Entry.objects.order_by('blog').distinct('blog')
```

## values()

Returns a QuerSet that returns dictionaries, rather than model instances, when used as an iterable.

```python
# This list contains a Blog object
>>> Blog.objects.filter(name__startswith='Beatles')
<QuerySet [,blog: Beatles Blog]>

# This list contains a dictionary
>>> Blog.objects.filter(name__startwith='Beatles').values()
<QuerySet [{'id': 1, 'name': 'Beatles Blog', 'tagline': 'All the latest Beatles news.'}]>

>>> Blog.objects.values('id', 'name')
<QuerySet [{'id': 1, 'name': 'Beatles Blog'}]>
```

## values_list()

Instead of returning dictionaries, it returns tuples when iterated over.

```python
>>> Entry.objects.values_list('id', 'headline')
<QuerySet [(1, 'First entry'), ...]>

>>> from django.db.models.funtions import Lower
>>> Entry.objects.values_list('id', Lower('headline'))
<QuerySet [(1, 'first entry'), ...]>

# passing flat=True will return results as single values
>>> Entry.objects.value_list('id').order_by('id')
<QuerySet[(1,), (2,), (3,), ...]>
>>> Entry.objects.values_list('id', flat=True).order_by('id')
<QuerySet [1, 2, 3, 4, ...]>
# it is an error to pass flat when there is more than one field

# you can pass named=True to get results as namedtuple():
>>> Entry.objects.values_list('id', 'headline', named=True)
<QuerySet [Row(id=1, headline='First entry'), ...]>
```

## dates()

```python
# "year" returns a list of all distinct year values for the field.
>>> Entry.objects.dates('pub_date', 'year')
[datetime.date(2005, 1, 1)]

# "month" returns a list of all distinct year/month values for the field.
>>> Entry.objects.dates('pub_date', 'month')
[datetime.date(2005, 2, 1), datetime.date(2005, 3, 1)]

#"week" returns a list of all distinct year/week values for 
# the field. All dates will be a Monday.
>>> Entry.objects.dates('pub_date', 'week')
[datetime.date(2005, 2, 14), datetime.date(2005, 3, 14)]

# "day" returns a list of all distinct year/month/day values for the field.
>>> Entry.objects.dates('pub_date', 'day')
[datetime.date(2005, 2, 20), datetime.date(2005, 3, 20)]

>>> Entry.objects.dates('pub_date', 'day', order='DESC')
[datetime.date(2005, 3, 20), datetime.date(2005, 2, 20)]
>>> Entry.objects.filter(headline__contains='Lennon').dates('pub_date', 'day')
[datetime.date(2005, 3, 20)]
```

## union(), intersection(), difference()

union() - uses SQL's UNION operator to combine the results of two or more QuerySets

intersection() - uses SQL's INTERSECTION operator to return shared elements of two or more querysets

difference() - usess SQL's EXCEPT operator to keep only elements present in the QuerySet but not in some other queryset

```python
>>> qs1 = Author.objects.values_list('name')
>>> qs2 = Entry.objects.values_list('headline')

>>> qs1.union(qs2).order_by('name')
>>> qs1.intersection(qs2, qs3)
>>> qs1.difference(qs2, qs3)
```

## select_related()

Returns a QuerySet that wull follow foreign-key relationship, sleecting additional related-object data when it executes its query. This is a performance booster which results in a single more complex query but means that later use of foreign-key relationship won't require database queries.

```python
>>> e = Entry.objects.get(id=5) # Hits the database
>>> b = e.blog # Hits the database again

>>> e = Entry.objects.select_related('blog').get(id=5) # Hits the database
>>> b = e.blog # doesn't hits the database, because e.blog has been 
               # prepoplated in the previous query.

# find all the blogs with entries scheduled to be published in the future
>>> blogs = set()
>>> for e in Entry.objects.filter(pub_date__gt=timezone.now())
              .select_related('blog'):
              # without select_related(), this would make a database query 
              # for each loop iteration in order to fetch the related blog
              # for each entry
              blog.add(e.blog) 

# the order of filter() and select_related() chaining isn't important
# the followong two are equivalent
>>> Entry.objects.filter(pub_date__gt=timezone.now()).select_related('blog')
>>> Entry.objects.select_related('blog').filter(pub_date__gt=timezone.now())
```

## [prefetch_related()](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#prefetch-related)

Similar to select_related but the startegy is quite different!

select_related works by creating an SQL join and including the fields of the related object in the SELECT statement. For this reason, select_related gets the related objects in the same database query. However, to avoid the much larger set that would result from joining across a 'many' relationship, select_relationship is limited to single-valued relationship - foreign key and one-to-one

prefetch_related does a separate lookup for each relationship, and does the joining in python. This allows it to prefetch many-to-many and many-to-one objects which can not be done using select_related

## using()

This method is for controlling which database the QuerySet will be evaluated against if you are using more than one database.

```python
# queries the database with the 'default' alias
>>> Entry.objects.all()

# queries the database with the 'backup' alias
>>> Entry.objects.using('backup')
```

## select_for_update(nowait=False, skip_locked=False, of=())

Returns a queryset that will lock rows untill the end of the transaction, generating a SELECT ... FOR UPDATE SQL statement on supported databases.

For example:

```python
from django.db import transaction

entries = Entry.objects.select_for_update().filter(author=request.user)
with tranasaction.atomic():
  for entry in entries:
    ...
```

When the queryset is evaluated (for entry in entries in this case), all matched entries will be locked until the end of the transaction block, meaning that other transactions will be prevented from changing or acquiring locks on them.

## raw(raw_query, params=None, translations=None)

Takes a raw SQL query, executes it, and returns a django.db.models.query.RawQuerySet instance.

## Operators that return new QuerySets

Combined querysets must use the same model.

### AND (&)

```python
# the followings are equivalent
>>> Model.objects.filter(x=1) & Model.objects.filter(y=2)
>>> Model.objects.filter(x=1, y=2)
>>> from django.db.models import Q
>>> Model.objects.filter(Q(x-1) & Q(y=2))

# SQL equivalent
SELECT ... WHWERE x=1 AND y=2
```

### OR (|)

Similarly

## Methods that do not return QuerySets

### get()

Returns the object matching the given lookup otherwise DoesNotExist exception and MultipleObjectsReturned exception if more than one object is found.

### create(**kwargs)

A convenience method for creating an object and saving it all in one step:
```python
>>> p = Person.objects.create(first_name="Ujjawal", last_name="Anand")
# is equivalent to
>>> p = Person(first_name="Ujjawal", last_name="Anand")
>>> p.save(force_insert=True)
```

### [get_or_create(defaults=None, **kwargs)](https://docs.djangoproject.com/en/3.0/ref/models/querysets/#get-or-create)

A convenience method for looking up an object with given kwargs, creating one if necessary.

Returns a tuple of **(object, created)**, where **object** is the retrieved or **created** is a boolean specifying whether a new object was created.

```python
try:
  obj = Person.objects.get(first_name='Ujjawal', last_name='Anand')
except Person.DoesNotExist:
  obj = Person(first_name='Ujjawal', last_name='Anand', birthday=date(1996, 2, 12) )
  obj.save()

# is quivalent to
obj, created = Person.objects.get_or_create(first_name='Ujjawal',
                                            last_name='Annad',
                                            defaults={'birthday': date(1996, 2, 12)})
```

You can specify more complex conditions for the retrieved object by chaining get_or_create() with filter() and using Q objects. For example, to retrieve Robert or Bob Marley if either exists, and create the latter otherwise:
```python
obj.created = Person.objects.filter(Q(first_name='Bob') | Q(first_name='Robert'))
                            .get_or_create(last_name='Marley', defaults={'first_name': 'Bob'})
```

### update_or_create(defaults=None, **kwargs)

```python
defaults = {'first_name': 'Bob'}
try:
    obj = Person.objects.get(first_name='John', last_name='Lennon')
    for key, value in defaults.items():
        setattr(obj, key, value)
    obj.save()
except Person.DoesNotExist:
    new_values = {'first_name': 'John', 'last_name': 'Lennon'}
    new_values.update(defaults)
    obj = Person(**new_values)
    obj.save()

# is quivalent to
obj, created = Person.objects.update_or_create(
    first_name='John', last_name='Lennon',
    defaults={'first_name': 'Bob'},
)
```

### bilk_create()
```python
>>> Entry.objects.bulk_create([
      Entry(headline='This is a test'),
      Entry(headline='This is only a test),
])
```


