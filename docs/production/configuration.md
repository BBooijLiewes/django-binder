# Configuration

This guide covers all Django Binder configuration options for production environments.

## Basic Configuration

### Required Settings

```python
# settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # Django Binder - must be included
    'binder',
    
    # Your apps
    'your_app',
]

# Required for CSRF handling
CSRF_FAILURE_VIEW = 'binder.router.csrf_failure'
```

### URL Configuration

```python
# urls.py
from django.urls import re_path, include
import binder.router
import binder.views
import binder.models

# Create router and register views
router = binder.router.Router().register(binder.views.ModelView)

urlpatterns = [
    re_path(r'^api/', include(router.urls)),
    re_path(r'^api/', binder.views.api_catchall, name='api_catchall'),
]

# Install history signal handlers
binder.models.install_history_signal_handlers(binder.models.BinderModel)
```

## Permission Configuration

### Basic Permissions

```python
# permissions.py
permissions = {
    'default': [
        ('auth.view_user', 'own'),
        ('auth.change_user', 'own'),
        ('auth.login_user', None),
        ('auth.logout_user', None),
        ('auth.unmasquerade_user', None),
    ],
    'staff': [
        ('myapp.view_mymodel', 'all'),
        ('myapp.add_mymodel', 'all'),
        ('myapp.change_mymodel', 'all'),
        ('myapp.delete_mymodel', 'all'),
    ],
}

# settings.py
from .permissions import permissions
BINDER_PERMISSION = permissions
```

## Database Configuration

### PostgreSQL (Recommended)

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'your_database',
        'USER': 'your_user',
        'PASSWORD': 'your_password',
        'HOST': 'localhost',
        'PORT': '5432',
        'OPTIONS': {
            'connect_timeout': 10,
        },
    }
}
```

### MySQL (Limited Support)

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'your_database',
        'USER': 'your_user',
        'PASSWORD': 'your_password',
        'HOST': 'localhost',
        'PORT': '3306',
        'OPTIONS': {
            # Required for Django Binder
            'init_command': 'SET SESSION group_concat_max_len = 1000000',
        },
    }
}
```

## File Handling Configuration

### Media Files

```python
# Basic media configuration
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')

# File upload limits
FILE_UPLOAD_MAX_MEMORY_SIZE = 10 * 1024 * 1024  # 10MB
DATA_UPLOAD_MAX_MEMORY_SIZE = 10 * 1024 * 1024   # 10MB

# For production with cloud storage
# DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
# AWS_STORAGE_BUCKET_NAME = 'your-bucket'
# AWS_S3_REGION_NAME = 'us-east-1'
```

### Direct File Serving

```python
# For nginx X-Accel-Redirect
INTERNAL_MEDIA_HEADER = 'X-Accel-Redirect'
INTERNAL_MEDIA_LOCATION = '/internal/media/'

# For Apache X-Sendfile
# INTERNAL_MEDIA_HEADER = 'X-Sendfile'
# INTERNAL_MEDIA_LOCATION = '/path/to/media/'
```

## WebSocket Configuration

### Redis Channel Layer

```python
# Install: pip install channels-redis
INSTALLED_APPS += ['channels']

CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('127.0.0.1', 6379)],
        },
    },
}

# WebSocket routing
ASGI_APPLICATION = 'your_project.asgi.application'
```

### RabbitMQ Configuration

```python
# For RabbitMQ WebSocket support
RABBITMQ_CONFIG = {
    'host': 'localhost',
    'port': 5672,
    'username': 'guest',
    'password': 'guest',
    'virtual_host': '/',
}
```

## Security Configuration

### Production Security

```python
# Security settings
DEBUG = False
ALLOWED_HOSTS = ['yourdomain.com', 'www.yourdomain.com']

# Security headers
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'

# HTTPS settings
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# Session and CSRF security
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
CSRF_COOKIE_HTTPONLY = True
```

### CORS Configuration

```python
# Install: pip install django-cors-headers
INSTALLED_APPS += ['corsheaders']

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    # ... other middleware
]

# CORS settings for API access
CORS_ALLOWED_ORIGINS = [
    "https://yourdomain.com",
    "https://www.yourdomain.com",
]

# For development only
# CORS_ALLOW_ALL_ORIGINS = True
```

## Logging Configuration

### Production Logging

```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
        'simple': {
            'format': '{levelname} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.FileHandler',
            'filename': '/var/log/django/binder.log',
            'formatter': 'verbose',
        },
        'console': {
            'level': 'DEBUG',
            'class': 'logging.StreamHandler',
            'formatter': 'simple',
        },
    },
    'loggers': {
        'binder': {
            'handlers': ['file', 'console'],
            'level': 'INFO',
            'propagate': True,
        },
        'binder.permissions': {
            'handlers': ['file'],
            'level': 'WARNING',
            'propagate': False,
        },
    },
}
```

## Performance Configuration

### Database Optimization

```python
# Connection pooling
DATABASES['default']['CONN_MAX_AGE'] = 600

# Query optimization
DATABASES['default']['OPTIONS'].update({
    'MAX_CONNS': 20,
    'OPTIONS': {
        'charset': 'utf8mb4',
        'use_unicode': True,
    },
})
```

### Caching Configuration

```python
# Redis cache
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        }
    }
}

# Session cache
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'
```

## Plugin Configuration

### Token Authentication

```python
INSTALLED_APPS += ['binder.plugins.token_auth']

# Token settings
TOKEN_EXPIRY_HOURS = 24 * 7  # 1 week
TOKEN_CLEANUP_DAYS = 30      # Clean up expired tokens after 30 days
```

### CSV Export Plugin

```python
INSTALLED_APPS += ['binder.plugins.views']

# CSV export settings
CSV_EXPORT_MAX_ROWS = 10000
CSV_EXPORT_TIMEOUT = 300  # 5 minutes
```

### My Filters Plugin

```python
INSTALLED_APPS += ['binder.plugins.my_filters']

# User filter settings
MAX_SAVED_FILTERS_PER_USER = 50
```

## Environment-Specific Configuration

### Development Settings

```python
# development.py
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1']

# Development database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# Disable security features for development
SECURE_SSL_REDIRECT = False
SESSION_COOKIE_SECURE = False
CSRF_COOKIE_SECURE = False
```

### Production Settings

```python
# production.py
from .base import *
import os

DEBUG = False
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

# Production database from environment
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST', 'localhost'),
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}

# Security settings
SECRET_KEY = os.environ.get('SECRET_KEY')
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

## Docker Configuration

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DEBUG=False
      - SECRET_KEY=${SECRET_KEY}
      - DB_NAME=binder_db
      - DB_USER=postgres
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_HOST=db
    depends_on:
      - db
      - redis

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: binder_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

### Environment Variables

```bash
# .env
SECRET_KEY=your-secret-key-here
DB_PASSWORD=your-db-password
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com
DEBUG=False
```

## Monitoring Configuration

### Health Check Endpoint

```python
# urls.py
from django.http import JsonResponse
from django.db import connection

def health_check(request):
    """Simple health check endpoint"""
    try:
        # Check database connection
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        
        return JsonResponse({
            'status': 'healthy',
            'database': 'connected'
        })
    except Exception as e:
        return JsonResponse({
            'status': 'unhealthy',
            'error': str(e)
        }, status=500)

urlpatterns += [
    re_path(r'^health/$', health_check, name='health_check'),
]
```

### Metrics Collection

```python
# Custom middleware for metrics
class MetricsMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        import time
        start_time = time.time()
        
        response = self.get_response(request)
        
        # Log request metrics
        duration = time.time() - start_time
        logger.info(f'Request: {request.method} {request.path} - '
                   f'Status: {response.status_code} - '
                   f'Duration: {duration:.3f}s')
        
        return response

MIDDLEWARE += ['your_app.middleware.MetricsMiddleware']
```

This configuration guide covers all essential Django Binder settings for production deployment.
