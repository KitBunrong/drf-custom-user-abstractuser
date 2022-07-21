# Custom User Model in Django
Why would you need to tweak some of the functinalities of the default `user` model?
* Maybe you want to make the `email` field the primayr unique identifier of users instead of `username`
* Maybe you want to include other unique fields for authentication like phone number
* Maybe you want to stack all user information, be it auth related or non-auth fields all in the `User` model

Before get into coding part, let's understand the 2 different ways we can create our own custom `User` model by extending from either `AbstractUser` or `AbstractBaseUser`.
1. `AbstractUser`: are you satisfied with the existing fields in the built-in User model but you want to use email as the primary unique identifier of your users or perhaps remove the username field? If yes, AbstractUser is the right option for you. 
2. `AbstractBaseUser`: There are two things to consider whenever starting a new project :
    * `User` model focused on authentication, and another model like `Profile` that keeps app-related information of the user. Here you may use `AbstractBaseUser` if you have additional auth-related attributes that you want to include in your User model.
    * `User` model that stores all information (auth related or non-auth attributes) all in one. You may want to do this to avoid using additional database queries to retrieve related model. If you want to take this approach, `AbstractBaseUser` is the right option.

Let's start with `Abstractuser` and next time will explore with `AbstractBaseUser`.


## Requirements
Create a python virtual environment and activate it.
```bash
pip install django djangorestframework
```
```bash
django-admin startproject custom_user_abstractuser .
```
```bash
python manage.py startapp users
```
Create a folder name `apps` and move `users` app inside it and modified it as below:
```python
# apps/users/apps.py

class UsersConfig(AppConfig):
    ....        
    name = "apps.users"
```
Add apps inside `settings.py`
```python
# custom_user_abstractuser/settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # thrid-party apps
    'rest_framework',

    # local apps
    'apps.users',
]
```
## 1. Custom Model Manager
`Manager` is a class that provides an interface through which database query operations are provided to Django models. You can have more than one manager for your model.

For our custom user model, we need to define a custom manager class because we are going to modify the initial `QuerySet` that the default Manager class returns. We do this by extending from `BaseUserManager` and providing two additional methods `create_user` and `create_superuser`.
```python
# apps/users/managers.py

from django.contrib.auth.base_user import BaseUserManager
from django.utils.translation import gettext as _


class CustomUserManager(BaseUserManager):
    """
    Custom user model manager where email is the unique identifier
    for authetication instead of username.
    """

    def create_user(self, email, password, **extra_fields):
        if not email:
            raise ValueError(_("Users must have an email address"))
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save()
        return user

    def create_superuser(self, email, password, **extra_fields):
        extra_fields.setdefault("is_staff", True)
        extra_fields.setdefault("is_superuser", True)
        extra_fields.setdefault("is_active", True)

        if extra_fields.get("is_staff") is not True:
            raise ValueError(_("Superuser must have is_staff=True."))
        if extra_fields.get("is_superuser") is not True:
            raise ValueError(_("Superuser must have is_superuser=True."))
        return self.create_user(email, password, **extra_fields)

```

## 2. The User Model
```python
# apps/users/models.py

from django.contrib.auth.models import AbstractUser
from django.db import models
from django.utils.translation import gettext as _

from .managers import CustomUserManager


class User(AbstractUser):
    email = models.EmailField(_("email address"), unique=True)

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ("username",)

    objects = CustomUserManager()

    def __str__(self):
        return self.email

```
* `USERNAME_FIELD` specifies the name of the field on the user model that is used as the unique identifier. In our case it’s email.
* `REQUIRED_FIELDS` A list of the field names that will be prompted for when creating a superuser via the `createsuperuser` management command. This doesn’t have any effect in other parts of Django like when creating a user in the admin panel.

## 3. The Settings
We now have to tell Django our new model that should be used to represent a User. This is done as follows.
```python
# custom_user_abstractuser/settings.py

AUTH_USER_MODEL = "users.User"
```
Now we can create and apply migrations.
```bash
python manage.py makemigrations
python manage.py migrate
```

## 4. Forms
Django’s built-in `UserCreationForm` and `UserChangeForm` forms must be extended to let them know the new `user` model that we are working with.

Create a file named _forms.py_ inside users app and add the following:
```python
# apps/users/forms.py

from django.contrib.auth.forms import UserChangeForm, UserCreationForm

from .models import User


class CustomUserCreationForm(UserCreationForm):
    class Meta:
        model = User
        fields = ("email",)


class CutomUserChangeForm(UserChangeForm):
    class Meta:
        model = User
        fields = ("email",)

```

## 5. Admin
Tell the `admin` panel to use these forms by extending from UserAdmin.
```python
# apps/users/admin.py

from django.contrib import admin
from django.contrib.auth.admin import UserAdmin as BaseUserAdmin
from django.utils.translation import gettext_lazy as _

from .forms import CustomUserCreationForm, CutomUserChangeForm
from .models import User


class UserAdmin(BaseUserAdmin):
    add_form = CustomUserCreationForm
    form = CutomUserChangeForm

    model = User

    list_display = (
        "username",
        "email",
        "is_active",
        "is_staff",
        "is_superuser",
        "last_login",
    )
    list_filter = ("is_active", "is_staff", "is_superuser")
    fieldsets = (
        (
            _("Login Credentials"),
            {
                "fields": (
                    "email",
                    "password",
                )
            },
        ),
        (
            _("Personal Information"),
            {
                "fields": (
                    "username",
                    "first_name",
                    "last_name",
                )
            },
        ),
        (
            _("Permissions and Groups"),
            {
                "fields": (
                    "is_staff",
                    "is_active",
                    "is_superuser",
                    "groups",
                    "user_permissions",
                )
            },
        ),
        ("Dates", {"fields": ("last_login", "date_joined")}),
    )
    add_fieldsets = (
        (
            None,
            {
                "classes": ("wide",),
                "fields": (
                    "username",
                    "email",
                    "password1",
                    "password2",
                    "is_staff",
                    "is_active",
                ),
            },
        ),
    )
    search_fields = ("email",)
    ordering = ("email",)


admin.site.register(User, UserAdmin)
```
* `add_form` and `form` specifies the forms to add and change user instances.
* `fieldsets` specifies the fields to be used in editing users and `add_fieldsets` specifies fields to be used when creating a user.