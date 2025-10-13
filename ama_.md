# Django Framework Components Overview

## 1. User Authentication
Django’s `django.contrib.auth` manages user accounts, login, logout, and permissions. It uses the `User` model and supports custom backends.

```python
from django.contrib.auth import authenticate, login
def login_view(request):
    if request.method == 'POST':
        user = authenticate(request, username=request.POST['username'], password=request.POST['password'])
        if user:
            login(request, user)
            return redirect('home')
        return render(request, 'login.html', {'error': 'Invalid credentials'})
    return render(request, 'login.html')
```

## 2. Sessions and Cookies
Sessions store user data across requests, managed by `django.contrib.sessions`. Cookies track session state in the browser.

```python
# settings.py
INSTALLED_APPS = ['django.contrib.sessions']
MIDDLEWARE = ['django.contrib.sessions.middleware.SessionMiddleware']

# View
def session_view(request):
    request.session['key'] = 'value'
    return render(request, 'example.html', {'data': request.session.get('key')})
```

## 3. Models
Models define database schema using Django’s ORM. They are classes inheriting from `django.db.models.Model`.

```python
from django.db import models
class Article(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
```

## 4. Middleware
Middleware processes requests/responses globally, enabling tasks like authentication or logging.

```python
class CustomMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    def __call__(self, request):
        response = self.get_response(request)
        return response
```

## 5. Crispy Forms
`django-crispy-forms` simplifies form rendering with customizable templates and layouts.

```python
from crispy_forms.helper import FormHelper
from crispy_forms.layout import Submit
class MyForm(forms.Form):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.helper = FormHelper()
        self.helper.add_input(Submit('submit', 'Submit'))
```

## 6. Template Language
Django’s template language uses tags (`{% %}`) and filters (`{{ variable|filter }}`) for dynamic HTML rendering.

```html
{% for article in articles %}
    <h2>{{ article.title|upper }}</h2>
{% endfor %}
```

## 7. Sending Email
Django’s `django.core.mail` sends emails using configured backends like SMTP.

```python
from django.core.mail import send_mail
send_mail(
    'Subject', 'Message', 'from@example.com',
    ['to@example.com'], fail_silently=False
)
```

## 8. Views
Views handle HTTP requests and return responses, either as functions or class-based views.

```python
from django.http import HttpResponse
def my_view(request):
    return HttpResponse("Hello, Django!")
```
