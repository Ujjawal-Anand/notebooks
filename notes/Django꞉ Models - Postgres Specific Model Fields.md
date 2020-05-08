---
tags: [django, model]
title: 'Django: Models - Postgres Specific Model Fields'
created: '2020-05-08T06:59:22.436Z'
modified: '2020-05-08T14:11:21.093Z'
---

# [Django: Models - Postgres Specific Model Fields](https://docs.djangoproject.com/en/3.0/ref/contrib/postgres/fields/)

## ArrayField

**class ArrayField(base_field, size=None, **options)**

**base_field**
This is required argument, can be IntegerField or ChrField. Most of fields are permitted with exceptions of those which handle relationships like (ForeignKey, OneToOneField, ManyToManyField) 

It is possible to nest array fields
example 

```python
from django.contrib.postgres.fields import ArrayField
from django.db import models

class chessBoard(models.Model):
  board = ArrayField(
            ArrayField(models.CharField(max_length=10, blank=True),
                      size=8,
                    ),
            size=8,       
  ) # size is optional, this will be passed to database 
    # although postgres at present doesn't enforce this restriction
```

### Querying ArrayField

Cosnider following model

```python
class Post(models.Model):
  name = models.CharField(max_lenght=255)
  tags = ArrayField(models.CharField(max_length=200), blank=True)

  def __str__(self):
    return self.name
```

**queries**

```python
>>> Post.objects.create(name='First Post', tags=['thoughts', 'django'])
>>> Post.objects.create(name='Second Post', tags=['thoughts'])
>>> Post.objects.create(name='Third Post', tags=['django', 'tutorial'])

# contains
>>> Post.objects.filter(tags__contains=['django'])
<QuerySet [<Post: First post>, <Post: Third post>]>

>>> Post.objects.filter(tags__contains=['django', 'thoughts'])
<QuerySet [<Post: First post>]>


# contained_by : the objects returned will be those where the data is a subset of the values passed
>>> Post.objects.filter(tags__contained_by=['django', 'thoughts'])
<QuerySet [<Post: First post>, <Post: Second post>]>

>>> Post.objects.filter(tags__contained_by=['django', 'thoughts', 'tutorial'])
<QuerySet [<Post: First post>, <Post: Second post>, <Post: Third post>]>


# overlap : Returns objects where the data shares any results with the values passed
>>> Post.objects.filter(tags__overlap=['thoughts'])
<QuerySet [<Post: First post>, <Post: Second post>]>

>>> Post.objects.filter(tags__overlap=['thoughts', 'tutorial'])
<QuerySet [<Post: First post>, <Post: Second post>, <Post: Third post>]>


# len : Returns the length of the array
>>> Post.objects.filter(tags__len=1)
<QuerySet [<Post: Second post>]>


# index transform : transforms index into the array. Any non-negative 
# integer can be used. There are no errors if it exceeds the size of the array.
>>> Post.objects.filter(tags__0='thoughts')
<QuerySet [<Post: First post>, <Post: Second post>]>

>>> Post.objects.filter(tags__1__iexact='Django')
<QuerySet [<Post: First post>]>

>>> Post.objects.filter(tags__276='javascript')
<QuerySet []>


# slice transform : Any two non-negative integers can be used, separated by a single underscore.
>>> Post.objects.filter(tags__0_1=['thoughts'])
<QuerySet [<Post: First post>, <Post: Second post>]>

>>> Post.objects.filter(tags__0_2__contains=['thoughts'])
<QuerySet [<Post: First post>, <Post: Second post>]>

```


## HStoreField

**class HStoreField(**options)**

A field of storing key-value pairs. The python data type used is dict. Keys must be string, and values can
be strings or null

To use this field, you will need to:

1. Add 'django.contrib.postgres' in your INSTALLED_APPS
2. [Setup the hstore extension](https://docs.djangoproject.com/en/3.0/ref/contrib/postgres/operations/#create-postgresql-extensions) in PostgresSql

You’ll see an error like can't adapt type 'dict' if you skip the first step, or type "hstore" does not exist if you skip the second.

Example

```python
from django.contrib.postgres.fields import HStoreField
from django.db import models

class Dog(models.Model):
  name = models.CharField(max_length=200)
  data = HStoreField()

  def __str__(self):
    return self.name
```

### Queries

```python
>>> Dog.objects.create(name='Rufus', data={'breed': 'labrador'})
>>> Dog.objects.create(name='Meg', data={'breed': 'collie'})

>>> Dog.objects.filter(data__breed='collie')
<QuerySet [<Dog: Meg>]>

# you can chain other lookups after key lookup
>>> Dog.objects.filter(data__breed__contains='l')
<QuerySet [<Dog: Rufus>, <Dog: Meg>]>

# has_key
# returns objects where the given key is in data
>>> Dog.objects.create(name='Rufus', data={'breed': 'labrador'})
>>> Dog.objects.create(name='Meg', data={'breed': 'collie', 'owner': 'Bob'})

>>> Dog.objects.filter(data__has_key='owner')
<QuerySet [<Dog: Meg>]>

# has_any_keys
# returns objects where any of the given keys is in data
>>> Dog.objects.filter(data__has_any_keys=['owner', 'breed'])
<QuerySet [<Dog: Rufus>, <Dog: Meg>]>


# has_keys
# returns objects where all of the given keys are in data
>>> Dog.objects.filter(data__has_keys=['breed', 'owner'])
<QuerySet [<Dog: Meg>]>


# keys
# returns objects where the array of the keys is the given value
# Order is not guaranteed to be reliable
>>> Dog.objects.create(name='Rufus', data={'toy': 'bone'})
>>> Dog.objects.create(name='Meg', data={'breed': 'collie', 'owner': 'Bob'})

>>> Dog.objects.filter(data__keys__overlap=['breed', 'toy'])
<QuerySet [<Dog: Rufus>, <Dog: Meg>]>

# values
>>> Dog.objects.create(name='Rufus', data={'breed': 'labrador'})
>>> Dog.objects.create(name='Meg', data={'breed': 'collie', 'owner': 'Bob'})

>>> Dog.objects.filter(data__values__contains=['collie])
<QuerySet [<Dog: Meg>]>

```


## JSONField

**class JSONField(encoder=None, **options)**

A field for storing JSON encoded data. In python the data is represented in its Python native format: dictionaries, lists, strings, numbers, booleans and None

**encoder**
  An optional JSON-encoding class to serialize data types not supported by the standard JSON seriealizer(datetime, uuid etc)

  see [more](https://docs.djangoproject.com/en/3.0/ref/contrib/postgres/fields/#jsonfield)


Example - 

```python
from django.contrib.postgres.fields import JSONField
from django.db import models

class Dog(models.Model):
  name = models.CharField(max_length=200)
  data = JSONField()

  def __str__(self):
    return self.name
```

### Querying JSONField

```python
>>> Dog.objects.create(name='Rufus', data= {
            'breed': 'labrador',
            'owner': {
              'name': 'Bob',
              'other_pets': [{
                'name': 'Fishy',
              }],
            },
})
>>> Dog.objects.create(name='Meg', data={'breed':'collie', 'owner': None})

>>> Dog.objects.filter(data__breed='collie')
<QuerySet [<Dog: Meg>]>

# multiple keys can be chained together to form a path lookup
>>> Dog.objects.filter(data__owner__name="Bob')
<QuerySet [<Dog: Rufus>]>

# if the key is an integer, it will be interpreted as an index lookup in an array
>>> Dog.objects.filter(data__owner__other_pets__0__name='Fishy')
<QuerySet [<Dog: Rufus>]>

# To query for missing keys, use the isnull lookup
>>> Dog.objects.filter(data__owner__isnull=True)
<QuerySet [<Dog: Meg>]>
```

### Containment and key operations

JSONField shares lookups relating to containment and keys with HStoreField
  - **contains** (accepts any JSON rather than a dictionary of strings)

  - **contained_by** (accepts any JSON rather than just a dictionary of strings)

  - **has_keys**

  - **has_any_keys**

  - **has_keys**






