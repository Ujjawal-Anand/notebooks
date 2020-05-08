---
tags: [django, model, query]
title: 'Django: QuerySet - Making Queries'
created: '2020-04-23T11:30:52.536Z'
modified: '2020-04-25T23:22:04.274Z'
---

# Django: QuerySet - Making Queries

*models.py*

```python
from django.db import models

class Blog(models.Model):
  name = models.CharField(max_lenght=100)
  tagline = models.TextField()

  def __str__(self):
    return self.name

class Author(models.Model):
  name = models.CharField(max_length=100)
  email = models.EmailField()

  def __str__(self):
    return self.name()

class Entry(models.Model):
  blog = models.ForeignKey(Blog, on_delete=models.CASCADE)
  headline = models.CharField(max_length=255)
  body_text = models.TextField()
  pub_date = models.DateField()
  mod_date = models.DateField()
  authors = models.ManyToManyField(Author)
  number_of_comments = models.IntegerField()
  number_of_pingbacks = models.IntegerField()
  ratings = models.IntegerField()

  def __str__(self):
    self.headline
```

## Saving ForeignKey and ManyToManyField

```python

# saving ForeignKey
>>> from blog.models import Blog, Entry
>>> entry = Entry.objects.get(pk=1)
>>> cheese_blog = Blog.objects.get(name="cheese_blog")
>>> entry.blog = cheese_blog
>>> entry.save

# save ManyToManyField
>>> from blog.models import Author
>>> ujjawal = Author.object.create(name="Ujjawal", email="ujjawalanand1729@gmail.com")
>>> entry.authors.add(ujjawal)
```

## Retrieving specific object with filter

```python
# get a queryset of blog entry published in the year 2006
>>> Entry.objects.filter(pub_date__year=2006)

>>> Entry.objects.filter(
            headline__startswith="Hello")
            .exclude(pub_date__gte=datetime.date.today())
            .filter(pub_date__gte=datetime.date(2019,1,30))
            )
# return queryset that has alll entries with headline that 
# starts with "Hello”, that were published between January 30, 2019, and the current day
```

**Filtered QuerySets are unique**

Each time you refine a QuerySet, you get a brand-new QuerySet that is in no way bound to the previous QuerySet. Each refinement creates a separate and distinct QuerySet that can be stored, used and reused.

```python
q1 = Entry.objects.filter(headline__startswith="Hello")
q2 = q1.exclude(pub_date__gte=datetime.date.today())
q3 = q3.filter(pub_date__gte=datetime.date(2019,1,30))
```

**QuerySets are lazy**

The act of creating querysets doesn't involve any database operation until you evaluate the said queryset

```python
>>> q = Entry.objects.filter(name__startswith="what")
>>> q = q.exclude(pub_date__gte=datetime.date.today())
>>> q = q.filter(body_text__icontains="food")
>>> print(q) 
```
Though this looks like three database hits, in fact it hits the database only once, at the last line (print(q)).

## Retrieving a single object with get()

if you know there is only object that matches your query, you can use get() which returns the object directly instead of a queryset
```python
>>> Entry.objects.get(pk=1)
```

**Note** - get() raises *DoesNotExist* exception when no objects matches with paramether.
Similarly, it raises *MultipleObjectsReturned* when more than one objects matches the parameter. 

## Limiting Querysets
```python
# returns first 5 objects
>>> Entry.objects.all()[:5]

# returns sixth object through 10th object
>>> Entry.objects.all()[5:10]

# negative indexing is not supported

# Generally, slicing a queryset returns a new queryset.
# An exception is when you use step parameter with slice
# Will actually evaluate the query in order to return every
# 2nd object up to 10th object
>>> Entry.objects.all()[:10:2]

# furture filtering or ordering of sliced queryset is probhited
# due to the ambiguous nature of how they may work

# To retrieve single object rather than list, use index (SELECT foo FROM bar LIMIT 1)
>>> Entry.objects.order_by('headline')[0]
```

## Field Lookups
Field lookups are how you specify the meat of an SQL WHERE clause. They’re specified as keyword arguments to the QuerySet methods filter(), exclude() and get().

```python
>>> Entry.objects.filter(pub_date__lte='2019-01-01')
```
translates to
```sql
SELECT * from blog_entry WHERE pub_date <= '2019-01-01'
```

The field specified in a lookup has to be the name of a model field. There’s one exception though, in case of a ForeignKey you can specify the field name suffixed with _id

```python
>>> Entry.objects.filter(blog_id=1)
```
if you pass an invalid keyword argument, the lookup function will raise *TypeError*

```python
# An exact match
>>> Entry.objects.filter(headline__excat="cats bite dog)
# will tanslate to
SELECT .... WHERE headline = "cats bite dog"

# case-insensitive match
>>> Entry.objects.filter(headline__iexact="cats bite dog")

>>> Entry.objects.filter(body_text__contains="Corona")

# if you are filtering across multiple relationships and one of the 
# intermediate models doesn’t have a value that meets the filter condition, 
# Django will treat it as if there is an empty (all values are NULL), but valid, object there
>>> Blog.objects.filter(entry__authors__name="Ujjawal")

# will return Blog objects that have an empty name on the author and also 
# those which have an empty author on the entry.
>>> Blog.objects.filter(entry__authors__name__isnull=True)

# if you don't want the latter to happen, you can write
>>> Blog.objects.filter(entry__authors__inull=False, entry__authors__name__isnull=True)
```

### Spanning multi-valued relationships

```python
# all blogs which have entries with both 'World' in headline and published in year 2008
>>> Blog.objects.filter(entry__headline__contains='World', entry__pub_date__year=2008)

# all blogs that contain an entry with “World” in the headline as well as 
# an entry that was published in 2008
>>> Blog.objects.filter(entry__headline__contains='World').filter(entry__pub_date__year=2008)

```

### Filters can reference fields on model (using f() lookup)

```python
#  all the entries where the rating of the entry is less than 
#  the sum of the pingback count and comment count
>>> Entry.objects.filter(rating__lt=F('number_of_comments') + F('number_of_pingbacks'))

# An F() object with a double underscore will introduce any 
# joins needed to access the related object

# retrieve all the entries where the author’s name is the same as the blog name
>>> Entry.objects.filter(authors__name=F('blog__name'))
```

### The pk lookup shortcut

```python
# following 3 are equivalent
>>> Blog.objects.get(id__exact=14)
>>> Blog.objects.get(pk=14)
>>> Blog.objects.get(id=14)

# any query term can be combined with pk to perform a query on the primary key of a model:
# get blogs with id 1, 4, 7
>>> Blog.objects.get(pk__in=[1,4,7])

# get all blogs entries > 14
>>> Blog.objects.get(pk__gt=14)

# pk also work across joins
# all following three are equivalent
>>> Entry.objects.filter(blog__pk=2)
>>> Entry.objects.filter(blog__id__exact=2)
>>> Entry.objects.filter(blog__id=2)
```

## Caching & Querysets

```python
# following will create two querysets, use them and then throw them away
# means, the same database query will be executed twice, effectively doubling your database load
# also two lists may not include the same database record
>>> print([e.headline for e in Entry.objects.all()])
>>> print([e.pub_date for e in Entry.objects.all()])


# To avoid this problem, save queryset and reuse it
>>> queryset = Entry.objects.all()
>>> print([p.headline for p in queryset])
>>> print([p.pub_date for p in queryset])
```

### When querysets are not cached
```python
>>> queryset = Entry.objects.all()

# limiting querysets using an array slice or index will not popilate cache
>>> print(queyset[5]) # Queries the database
>>> print(queryset[5]) # Queries the database, again

# if the entire database has already been evaluated. the cache will be checked
>>> [entry for entry in queryset] # queries the database
>>> print(queyset[5]) # Use cache
>>> print(queryset[5]) # Use cache
```

## Complex lookup with Q objects
A Q object(django.db.models.Q) is an object ued to encapsulate a collection of keyword arguments

Example:
```python
>>> from django.db.models import Q
>>> Q(question__startswith="What") | Q(question__startswith="Who")
# this is equivalent to
WHERE question LIKE 'What%' OR question LIKE 'Who%'

# You can compose statements of arbitrary complexity by combining Q 
# objects with the &, | and ~ operators and use parenthetical grouping
>>> Q(question__startswith='who') | ~Q(question__pub_date__year=2019)

# if you provide multiple Q argumets to a lookup function, the argument
# will be ANDed together
>>> Poll.objects.get(Q(question__startswith='who'), 
                        Q(pub_date=date(2019, 5, 2)) | Q(pub_date=date(2018, 2,1)))

# roughly translates to SQL
SELECT * FROM polls WHERE question LIKE 'Who%' AND (pub_date= '2019-05-02' OR pub_date= '2018-02-01')
```

## Comparing Objects
To compare two model instances, use the standard Python comparison operator, the double equals sign: ==. Behind the scenes, that compares the primary key values of two models.

```python
# two are equivalent
>>> some_entry == other_entry
>>> some_entry.id == other_entry.id
```

## Deliting Objects
The delete method, conveniently, is named delete(). This method immediately deletes the object and returns the number of objects deleted and a dictionary with the number of deletions per object type. Example:

```python
>>> e.delete()
(1, {'weblog.Entry': 1})

# delete in bulk, this deletes all Entry objects with a pub_date year of 2005
>>> Entry.object.
```

## Updating multiple object at once
```python
>>> Entry.objects.filter(pub_date__year=2019).update(headline="Hello World")
```

You can only update non-relational and ForeignKey using this method. To update a relational field provide the new value as constant
```python
>>> blog = Blog.objects.get(pk=1)
>>> Entry.objects.all().update(Blog=b)

# the only restriction on the QuerySet is you have only access to one table - the main table
# You can filter based on related fields
>>> Entry.objects.select_related().filter(Blog=b).update(headline="Nothing New")
```

Be aware that update() method doesn't run any save() methods on your model. You willhave to call save method manually like:
```python
for item in my_queryset:
  item.save()
```

Calls to update also can use F() expressions to update one file based on the value of another filed in the model. This is especially useful for incrementing counters based on curren value.
```python
>>> Entry.objects.all().update(number_of_pingbacks=F('number_of_pingbacks')+1)

# You can only reference fields local to the model being updated 
# unlike F() objects in filter and exclude clauses

# This will raise FieldError
>>> Entry.objects.update(headline=F('blog__name'))
```

## Related Objects
When you define a relationship in a model (i.e, a ForeignKey, OneToOneField or ManyToManyField) instances of that model will have a convenient API to access related objects.


### One-to-Many realtionship

**Forward Realtinship**

```python
>>> e = Entry.objects.get(pk=1)
>>> e.blog # returns the related Blog object

# if a ForeignKey field has null=True set you cnan assign None to 
# remove the relation
>>> e.blog =  None
>>> e.save() # "UPDATE blog_entry SET blog_id = NULL ...;"

# forward access to one-to-many relationship is cached the first time 
# the related object is accessed.
>>> e = Entry.objects.get(id=2)
>>> print(e.blog) # Hits the database to retrieve the associated Blog
>>> print(e.blog) # doesn't hit the database, uses cached version

# note that select_related() QuerySet method recusively prepolates
# the cache of all one-to-many relationship ahead of time
>>> e = Entry.objects.select_related().get(id=2)
>>> print(e.blog) # Doesn't hot the database; use cached version
>>> print(e.blog) # Doesn't hot the database; use cached version
```

**Backward Relationship**
The instance of foreign-key model has access to a Manager (named FOO_set, where FOO is the source model name) that returns all instances of the first model.

```python
>>> b = Blog.objects.get(id=1)
>>> b.entry_set.all() # returns all Entry objects related to Blog

# b.entry_set is a Manager that return QuerySet
>>> b.entry_set.filter(headline__contains='Lennon')
>>> b.entry_set.count()

# You can override the FOO_set name by setting the related_name 
# parameter in the ForeignKey definition. Like
>>> blog = ForeignKey(Blog, on_delete=models.CASCADE, related_name='entries')
>>> b = Blog.objects.get(id=1)
>>> b.entries.all() # returns all Entry obejcts related to Blog 

```

### Many-to_many relationships

```python
>>> e = Entry.objects.get(id=2)
>>> e.authors.all() # returns all Author objects for this Entry
>>> e.authors.count()
>>> e.authors.filter(name__contains="Ujjawal")

# backward relationship is just like ForeignKey
>>> a = Author.objects.get(id=5)
>>> a.entry_set.all() # returns all Entry objects for this Author

```

### One-to-one relationship

Example Model
```python
class EntryDetail(models.Model):
  entry = models.OneToOneField(Entry, on_delete=models.CASCADE)
  details = models.TextField()

ed = EntryDetail.objects.get(id=2)
ed.entry # returns the related Entry object

# Backward relatinship
e = Entry.objects.get(id=2)
# notice the difference here - hint: focus on return value
e.entrydetail # returns the related EntryDetail object

# if no object has been assigned to this relationship, 
# Django will raise a DoesNotExist exception

```

### How are the backward relationship possible?
 
The answer lies in app_registry. When Django starts, it imports each application listed in INSTALLED_APP, and then the model module inside each application. Whenever a new models class is created, Django adds backward-relationships to any related model. If the related models haven't been imported yet. Django keeps tracks of the relationship and adds them when the related models eventually are imported.

For this reason, it's particularly important that all the models you are using be defined in applications listed in INSTALLED_APPS. Otherwise, backwards relations may not work properly.  









