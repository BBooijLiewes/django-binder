# Installation & Setup

This guide covers the complete installation and setup process for Django Binder in production environments.

## Prerequisites

### System Requirements
- Python 3.6 or higher
- Django 3.0 - 5.0
- PostgreSQL 10+ (recommended) or MySQL 5.7+
- Redis (optional, for WebSocket support)

### Development Tools
- Git
- Virtual environment manager (venv, virtualenv, or conda)
- Database client tools

## Installation Methods

### 1. Install from Git Repository (Recommended)

```bash
# Install the latest stable version
pip install git+https://github.com/CodeYellowBV/django-binder.git@1.7.0

# Or install the latest development version
pip install git+https://github.com/CodeYellowBV/django-binder.git@master
```

### 2. Install from Local Clone

```bash
# Clone the repository
git clone https://github.com/CodeYellowBV/django-binder.git
cd django-binder

# Install in development mode
pip install -e .
```

## Project Setup

### 1. Django Settings Configuration

Add Django Binder to your `settings.py`:

```python
# settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # Django Binder
    'binder',
    
    # Your apps
    'your_app',
]

# Required for CSRF handling
CSRF_FAILURE_VIEW = 'binder.router.csrf_failure'

# Database configuration (PostgreSQL recommended)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'your_database',
        'USER': 'your_user',
        'PASSWORD': 'your_password',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}

# For MySQL (with limitations)
# DATABASES = {
#     'default': {
#         'ENGINE': 'django.db.backends.mysql',
#         'NAME': 'your_database',
#         'USER': 'your_user',
#         'PASSWORD': 'your_password',
#         'HOST': 'localhost',
#         'PORT': '3306',
#         'OPTIONS': {
#             'init_command': 'SET SESSION group_concat_max_len = 1000000',
#         },
#     }
# }

# Media files configuration
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')

# Optional: WebSocket support
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('127.0.0.1', 6379)],
        },
    },
}
```

### 2. URL Configuration

Configure your main `urls.py`:

```python
# urls.py
from django.contrib import admin
from django.urls import re_path, include
from django.conf import settings
from django.conf.urls.static import static

import binder.router
import binder.views
import binder.models

# Create and configure the router
router = binder.router.Router().register(binder.views.ModelView)

urlpatterns = [
    re_path(r'^admin/', admin.site.urls),
    
    # Django Binder API routes
    re_path(r'^api/', include(router.urls)),
    re_path(r'^api/', binder.views.api_catchall, name='api_catchall'),
]

# Serve media files in development
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)

# Install history signal handlers
binder.models.install_history_signal_handlers(binder.models.BinderModel)
```

### 3. Create Your First Model

Create a model in your app's `models.py`:

```python
# your_app/models.py
from django.db import models
from binder.models import BinderModel

class Article(BinderModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-created_at']
    
    def __str__(self):
        return self.title
```

### 4. Create Your First View

Create a view in your app's `views.py`:

```python
# your_app/views.py
from binder.views import ModelView
from .models import Article

class ArticleView(ModelView):
    model = Article
    
    # Optional: Configure searchable fields
    searches = ['title__icontains', 'content__icontains']
    
    # Optional: Configure filterable fields
    alternative_filters = {
        'published_status': ['published'],
        'date_range': ['created_at__gte', 'created_at__lte'],
    }
```

### 5. Database Migration

Run Django migrations to create the database tables:

```bash
# Create migrations for your models
python manage.py makemigrations

# Apply migrations
python manage.py migrate

# Create a superuser (optional)
python manage.py createsuperuser
```

### 6. Test Your Installation

Start the development server:

```bash
python manage.py runserver
```

Test your API endpoints:

```bash
# List all articles
curl http://localhost:8000/api/article/

# Create a new article
curl -X POST http://localhost:8000/api/article/ \
  -H "Content-Type: application/json" \
  -d '{"title": "My First Article", "content": "Hello, World!", "published": true}'

# Get a specific article
curl http://localhost:8000/api/article/1/
```

## Production Configuration

### Environment Variables

Create a `.env` file for environment-specific settings:

```bash
# .env
DEBUG=False
SECRET_KEY=your-secret-key-here
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com
```

### Security Settings

```python
# settings.py (production)
DEBUG = False
ALLOWED_HOSTS = ['yourdomain.com', 'www.yourdomain.com']

# Security headers
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'

# HTTPS settings (if using HTTPS)
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Session security
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

### Static Files Configuration

```python
# settings.py
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

# For production with CDN
# DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
# STATICFILES_STORAGE = 'storages.backends.s3boto3.StaticS3Boto3Storage'
```

## Docker Setup (Optional)

Create a `Dockerfile`:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "your_project.wsgi:application"]
```

Create a `docker-compose.yml`:

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DEBUG=False
      - DATABASE_URL=postgresql://postgres:password@db:5432/your_db
    depends_on:
      - db
      - redis

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: your_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:6-alpine

volumes:
  postgres_data:
```

## Troubleshooting

### Common Issues

1. **Import Error**: Ensure Django Binder is properly installed and in your Python path
2. **Database Connection**: Verify database credentials and connectivity
3. **CSRF Errors**: Ensure `CSRF_FAILURE_VIEW` is set correctly
4. **Media Files**: Configure `MEDIA_URL` and `MEDIA_ROOT` for file uploads

### Verification Commands

```bash
# Check Django Binder installation
python -c "import binder; print(binder.__file__)"

# Verify database connection
python manage.py dbshell

# Check for migration issues
python manage.py showmigrations

# Test API endpoints
python manage.py shell
>>> from django.test import Client
>>> client = Client()
>>> response = client.get('/api/article/')
>>> print(response.status_code)
```

## Next Steps

- Read the [Quick Start Guide](quickstart.md) to build your first API
- Learn about [Models & Fields](models.md) for advanced model configuration
- Explore [Views & ViewSets](views.md) for custom endpoint creation
- Review [Security Best Practices](security.md) for production deployment
