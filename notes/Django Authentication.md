---
tags: [authentication, django]
title: Django Authentication
created: '2020-04-18T06:52:02.813Z'
modified: '2020-04-18T16:11:14.302Z'
---

# Django Authentication

## Authentication Backends
This should be a list of Python path names that point to Python classes that know how to authenticate.
*settings.py*
```python
AUTHENTICATION_BACKENDS = [
    "django.contrib.auth.backends.ModelBackend",
    "allauth.account.auth_backends.AuthenticationBackend",
]
```
- The order of AUTHENTICATION_BACKENDS matters, so if the same username and password is valid in multiple backends, Django will stop processing at the first positive match.

- If a backend raises a PermissionDenied exception, authentication will immediately fail. Django won’t check the backends that follow.



## Extending the existing User model
**(1) Via Proxy Model**

```python
from django.contrib.auth.models import User
from django.db import models

class Employee(models.Model):
  user = models.OneToOneField(User, on_delete=models.CASCADE)
  department = models.CharField(max_length=100)

```

Assuming an employee with name Ujjawal Anand who has both User and Employee model, you can access related information by following
```python
>> user = User.objects.get(username='ujjawal')
>> user_department = user.employee.department
```

<details><summary>Add Profile Model to User Page in Admin</summary>

```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin
from django.contrib.auth.model import User

from .models import Employee

# define an admin descriptor for Employee
class EmployeeInline(admin.StackedInline):
  model = Employee
  can_delete = false
  verbose_name_plural = 'employee'

# define a new user admin
class UserAdmin(BaseUserAdmin):
  inline = (EmployeeInline,)

# re-register UserAdmin
admin.site.unregister(User)
admin.site.register(User, UserAdmin)
```
</details>
<br />

**(2) By creating custom User model**

Django allows you to overrirde the default User model by providing a value to AUTH_USER_MODEL setting that referes custom user model
```python
AUTH_USER_MODEL = 'myapp.MyUser'
```

*models.py*
```python
from django.db.models import CharField
from django.contrib.auth.models import AbstractUser
from django.urls import reverse

class MyUser(AbstractUser):
  name = CharField("Name of user", max_length=100, blank=True)

  def get_absolute_url(self):
    return reverse('users:detail', kwargs={'username': self.username})
```

<details><summary>Register User in Admin</summary>

*admin.py*
```python
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as AbstractUserAdmin
from django.contrib.auth import get_user_model

from .forms import UserChangeForm, UserCreationForm

# Register your models here.

User = get_user_model()

@admin.register(User)
class UserAdmin(AbstractUserAdmin):
    form = UserChangeForm
    add_form = UserCreationForm
    fieldsets = ((User, {"fields": ('name',),}),) + AbstractUserAdmin.fieldsets
    list_display = ["username", "name", "is_superuser"]
    search_fields = ["name"]
```

*forms.py*
```python
from django.contrib.auth import forms, get_user_model
from django.core.exceptions import ValidationError

User = get_user_model()

class UserChangeForm(forms.UserChangeForm):
    class Meta(forms.UserChangeForm.Meta):
        model = User

class UserCreationForm(forms.UserCreationForm):
    error_message = forms.UserCreationForm.error_messages.update(
        {'duplicate_username': 'This username has already been taken'
    })

    class Meta(forms.UserCreationForm.Meta):
        model = User

    def clean_username(self):
        username = self.cleaned_data["username"]

        try:
            username = User.objects.get(username=username)
        except User.DoesNotExist:
            return username
        
        raise ValidationError(self.error_messages['duplicate_username'])
```
</details>

*Note:*
- Changing AUTH_USER_MODEL after you’ve created database tables is significantly more difficult since it affects foreign keys and many-to-many relationships.

- Due to limitations of Django’s dynamic dependency feature for swappable models, the model referenced by AUTH_USER_MODEL must be created in the first migration of its app (usually called 0001_initial); otherwise, you’ll have dependency issues.

**(3) By specifying a custom user model**

*models.py*
```python
from django.contrib.auth.models import AbstractBaseUser
from django.db import models
class MyUser(AbstractBaseUser):
  identifier = models.CharField(max_length=100, unique=True)
  date_of_birth = models.DateField()
  height =  models.FloatField()
  
  
  USERNAME_FIELD = 'identifier'
  REQUIRED_FIELD = ['date_of_birth', 'height'] # will be prompted for when creating a user via the createsuperuser 

```

## Examples
**Objective**: Use an email as the primary user identifier instead of username for authentication

### Difference between AbstractUser & AbstractBaseUser
**AbstractUser:** AbstractUser is complete User model class, complete with fields as an abstract class 
so that it can be inherited and custom fields can be added to it. 

**AbstractBaseUser:** AbstractBaseUser only contains authentication related functions and no actual
fields. You have to supply them wherever you subclass it.

### Custom Model Manager
*managers.py*
```python
from django.contrib.auth.base_user import BaseUserManager
from django.utils.translation import ugettext_lazy as _

class CustomUserManager(BaseUserManager):
  """
    custom user manager where email is unique identifier 
    instead of username
  """

  def create_user(self, email, password, **extra_fields):
    """
      create and save user with given email and password
    """
    now = timezone.now()
    email = self.normalize_email(email)
    user = self.model(email=email, 
                      is_staff=False, is_superuser=false,
                      is_active=True, last_login=now, 
                      date_joined=now, **extra_fields)
    user.set_password(password)
    user.save(using=self._db) # self._db : This is simply there to optionally 
                              # override it in case you have multiple databases
    return user

  def create_superuser(self, email, password, **extra_fields):
    """
      create a superuser
    """
    user = self.create_user(email, password, **extra_fields)
    user.is_staff = True
    user.is_superuser = True
    user.is_active = True
    user.save(using=self._db)
    return user
```

### Custom User Model

*models.py*

```python
from django.db import models
from django.contrib.auth.models import AbstractBaseUser, PermissionsMixin
from django.utils.translation import gettext_lazy as _
from django.utils import timezone

from .manager import CustomUserManager

class CustomUser(AbstractBaseUser, PermissionsMixin):
  email = models.EmailField(_('email_address'), unique=True)
  is_staff = models.BooleanField(default=False)
  is_active = models.BooleanField(default=True)
  date_joined = models.DateTimeField(default=timezone.now())

  USERNAME_FIELD = 'email'
  REQUIRED_FIELDS = []

  objects = CustomUserManager()

  def __str__(self):
    return self.email


```

In, *settings.py*
```python
AUTH_USER_MODEL = 'users.CustomUser'
```

*forms.py*

```python
from django import forms
from django.contrib.auth.forms import ReadOnlyPasswordHashField
from django.contrib.auth import get_user_model

User = get_user_model()

class UserCreationForm(forms.ModelForm):
  """
    A form for creating 
  """
```

<details> <summary>Full AbstractUser with methods</summary>

```python
class AbstractUser(auth_models.AbstractBaseUser,
                   auth_models.PermissionsMixin):
    """
    An abstract base user suitable for use in Oscar projects.

    This is basically a copy of the core AbstractUser model but without a
    username field - taken from django oscar
    """
    email = models.EmailField(_('email address'), unique=True)
    first_name = models.CharField(
        _('First name'), max_length=255, blank=True)
    last_name = models.CharField(
        _('Last name'), max_length=255, blank=True)
    is_staff = models.BooleanField(
        _('Staff status'), default=False,
        help_text=_('Designates whether the user can log into this admin '
                    'site.'))
    is_active = models.BooleanField(
        _('Active'), default=True,
        help_text=_('Designates whether this user should be treated as '
                    'active. Unselect this instead of deleting accounts.'))
    date_joined = models.DateTimeField(_('date joined'),
                                       default=timezone.now)

    objects = UserManager()

    USERNAME_FIELD = 'email'

    class Meta:
        abstract = True
        verbose_name = _('User')
        verbose_name_plural = _('Users')

    def clean(self):
        super().clean()
        self.email = self.__class__.objects.normalize_email(self.email)

    def get_full_name(self):
        """
        Return the first_name plus the last_name, with a space in between.
        """
        full_name = '%s %s' % (self.first_name, self.last_name)
        return full_name.strip()

    def get_short_name(self):
        """
        Return the short name for the user.
        """
        return self.first_name

    def email_user(self, subject, message, from_email=None, **kwargs):
        """
        Send an email to this user.
        """
        send_mail(subject, message, from_email, [self.email], **kwargs)

    def _migrate_alerts_to_user(self):
        """
        Transfer any active alerts linked to a user's email address to the
        newly registered user.
        """
        ProductAlert = self.alerts.model
        alerts = ProductAlert.objects.filter(
            email=self.email, status=ProductAlert.ACTIVE)
        alerts.update(user=self, key='', email='')

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        # Migrate any "anonymous" product alerts to the registered user
        # Ideally, this would be done via a post-save signal. But we can't
        # use get_user_model to wire up signals to custom user models
        # see Oscar ticket #1127, Django ticket #19218
        self._migrate_alerts_to_user()
```

</details>

<details><summary>Test For Custom UserManager</summary>

```python
from django.test import TestCase
from django.contrib.auth import get_user_model


class UsersManagersTests(TestCase):

    def test_create_user(self):
        User = get_user_model()
        user = User.objects.create_user(email='normal@user.com', password='foo')
        self.assertEqual(user.email, 'normal@user.com')
        self.assertTrue(user.is_active)
        self.assertFalse(user.is_staff)
        self.assertFalse(user.is_superuser)
        try:
            # username is None for the AbstractUser option
            # username does not exist for the AbstractBaseUser option
            self.assertIsNone(user.username)
        except AttributeError:
            pass
        with self.assertRaises(TypeError):
            User.objects.create_user()
        with self.assertRaises(TypeError):
            User.objects.create_user(email='')
        with self.assertRaises(ValueError):
            User.objects.create_user(email='', password="foo")

    def test_create_superuser(self):
        User = get_user_model()
        admin_user = User.objects.create_superuser('super@user.com', 'foo')
        self.assertEqual(admin_user.email, 'super@user.com')
        self.assertTrue(admin_user.is_active)
        self.assertTrue(admin_user.is_staff)
        self.assertTrue(admin_user.is_superuser)
        try:
            # username is None for the AbstractUser option
            # username does not exist for the AbstractBaseUser option
            self.assertIsNone(admin_user.username)
        except AttributeError:
            pass
        with self.assertRaises(ValueError):
            User.objects.create_superuser(
                email='super@user.com', password='foo', is_superuser=False)
```

</details>






















**Sources**
[Django Auth Documentation @django_documentation](https://docs.djangoproject.com/en/3.0/topics/auth/customizing/)
[Writing an authentication backend @django_documentation](https://docs.djangoproject.com/en/3.0/topics/auth/customizing/#writing-an-authentication-backend)
[Specifying a custom user model @django_documentation](https://docs.djangoproject.com/en/3.0/topics/auth/customizing/#specifying-a-custom-user-model)
[multiple database @django_documentation](https://docs.djangoproject.com/en/3.0/topics/db/multi-db/)
[Create Custom User Model @testdriven.io](https://testdriven.io/blog/django-custom-user-model/)


