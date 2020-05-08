---
title: 'Django : How Did I Solve'
created: '2020-05-01T04:10:31.935Z'
modified: '2020-05-07T14:32:02.438Z'
---

# Django : How Did I Solve

## ImproperlyConfigured: 

The included URLconf 'config.urls' does not appear to have any patterns in it. If you see valid patterns in the file then the issue is probably caused by a circular import

**Solution** - Although the exception happned in urls.py file but actual source of error was in views.py file. It was importing a model which I had removed from models.py file.

Another error was in urls.py itself.
I used something  
```python
path('download_paper/<string:username>/', views.download_answer_paper, name='paper save'),`
```
It should have been
```python
path('download_paper/<str:username>/', views.download_answer_paper, name='paper save'),`
```
notice thr string parameter of url.



**Lesson** - Always try to check your imports first while countering such exception.


## Static files don't load when debug=False

To load static file while debug=False use

```python
./manage.py runserver --insecure
```


