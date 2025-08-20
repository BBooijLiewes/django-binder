# Debugging

This guide covers comprehensive debugging strategies for Django Binder applications, from development debugging to production troubleshooting.

## Development Debugging

### Debug Settings

Configure debugging for development:

```python
# settings/development.py
DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1', '0.0.0.0']

# Enhanced debug toolbar
INSTALLED_APPS += [
    'debug_toolbar',
]

MIDDLEWARE += [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
]

# Debug toolbar configuration
DEBUG_TOOLBAR_CONFIG = {
    'SHOW_TOOLBAR_CALLBACK': lambda request: DEBUG,
    'SHOW_COLLAPSED': True,
}

# Detailed logging
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
        'console': {
            'level': 'DEBUG',
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
        'file': {
            'level': 'DEBUG',
            'class': 'logging.FileHandler',
            'filename': 'debug.log',
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
            'propagate': True,
        },
        'binder': {
            'handlers': ['console', 'file'],
            'level': 'DEBUG',
            'propagate': True,
        },
        'django.db.backends': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
}
```

### Debug Middleware

Create custom debug middleware:

```python
# debug/middleware.py
import time
import logging
from django.db import connection
from django.utils.deprecation import MiddlewareMixin

logger = logging.getLogger('debug')

class DebugMiddleware(MiddlewareMixin):
    def process_request(self, request):
        """Log request details"""
        request._debug_start_time = time.time()
        request._debug_queries_before = len(connection.queries)
        
        logger.debug(f"Request: {request.method} {request.path}")
        logger.debug(f"User: {request.user}")
        logger.debug(f"GET params: {dict(request.GET)}")
        
        if request.method in ['POST', 'PUT', 'PATCH']:
            if hasattr(request, 'body'):
                logger.debug(f"Request body: {request.body[:1000]}")  # First 1000 chars
    
    def process_response(self, request, response):
        """Log response details"""
        if hasattr(request, '_debug_start_time'):
            duration = time.time() - request._debug_start_time
            query_count = len(connection.queries) - request._debug_queries_before
            
            logger.debug(f"Response: {response.status_code}")
            logger.debug(f"Duration: {duration:.3f}s")
            logger.debug(f"Queries: {query_count}")
            
            # Log slow requests
            if duration > 1.0:
                logger.warning(f"Slow request: {request.path} took {duration:.3f}s")
            
            # Log queries for slow requests
            if query_count > 10:
                logger.warning(f"High query count: {request.path} used {query_count} queries")
                for query in connection.queries[-query_count:]:
                    logger.debug(f"Query ({query['time']}s): {query['sql'][:200]}...")
        
        return response

# Add to MIDDLEWARE in development
MIDDLEWARE += ['debug.middleware.DebugMiddleware']
```

### Debug Views

Create debug views for development:

```python
# debug/views.py
from django.http import JsonResponse
from django.conf import settings
from django.db import connection
from binder.views import ModelView
from binder.router import list_route

class DebugMixin:
    """Mixin to add debug information to views"""
    
    @list_route(name='debug', unauthenticated=True)
    def debug_info(self, request):
        """Debug information endpoint"""
        if not settings.DEBUG:
            return JsonResponse({'error': 'Debug mode disabled'}, status=403)
        
        debug_info = {
            'model': self.model.__name__,
            'view_class': self.__class__.__name__,
            'total_objects': self.model.objects.count(),
            'database_queries': len(connection.queries),
            'user': {
                'authenticated': request.user.is_authenticated,
                'username': request.user.username if request.user.is_authenticated else None,
                'is_staff': request.user.is_staff if request.user.is_authenticated else False,
            },
            'request_info': {
                'method': request.method,
                'path': request.path,
                'get_params': dict(request.GET),
                'content_type': request.content_type,
            }
        }
        
        # Add queryset information
        try:
            queryset = self.get_queryset(request)
            debug_info['queryset'] = {
                'count': queryset.count(),
                'sql': str(queryset.query),
                'filters_applied': bool(request.GET),
            }
        except Exception as e:
            debug_info['queryset_error'] = str(e)
        
        return JsonResponse(debug_info)

class ArticleView(DebugMixin, ModelView):
    model = Article
```

## Query Debugging

### SQL Query Analysis

Debug database queries:

```python
from django.db import connection
from django.conf import settings
import logging

logger = logging.getLogger('query_debug')

class QueryDebugMixin:
    """Mixin to debug database queries"""
    
    def dispatch(self, request, *args, **kwargs):
        if settings.DEBUG:
            # Reset query log
            connection.queries_log.clear()
        
        response = super().dispatch(request, *args, **kwargs)
        
        if settings.DEBUG:
            self.log_queries(request)
        
        return response
    
    def log_queries(self, request):
        """Log all queries executed during request"""
        queries = connection.queries
        total_time = sum(float(query['time']) for query in queries)
        
        logger.info(f"Query Debug for {self.__class__.__name__}:")
        logger.info(f"Total queries: {len(queries)}")
        logger.info(f"Total time: {total_time:.3f}s")
        
        # Log individual queries
        for i, query in enumerate(queries, 1):
            query_time = float(query['time'])
            sql = query['sql']
            
            # Highlight slow queries
            if query_time > 0.1:
                logger.warning(f"SLOW Query {i} ({query_time:.3f}s): {sql}")
            else:
                logger.debug(f"Query {i} ({query_time:.3f}s): {sql}")
        
        # Detect N+1 queries
        self.detect_n_plus_one_queries(queries)
    
    def detect_n_plus_one_queries(self, queries):
        """Detect potential N+1 query problems"""
        similar_queries = {}
        
        for query in queries:
            # Normalize SQL by removing specific values
            normalized_sql = self.normalize_sql(query['sql'])
            
            if normalized_sql in similar_queries:
                similar_queries[normalized_sql] += 1
            else:
                similar_queries[normalized_sql] = 1
        
        # Report potential N+1 queries
        for sql, count in similar_queries.items():
            if count > 5:  # More than 5 similar queries
                logger.warning(f"Potential N+1 query detected ({count} times): {sql[:100]}...")
    
    def normalize_sql(self, sql):
        """Normalize SQL query for comparison"""
        import re
        # Replace numbers and quoted strings with placeholders
        normalized = re.sub(r'\d+', 'N', sql)
        normalized = re.sub(r"'[^']*'", "'X'", normalized)
        normalized = re.sub(r'"[^"]*"', '"X"', normalized)
        return normalized

class ArticleView(QueryDebugMixin, ModelView):
    model = Article
```

### Query Optimization Debug

Debug query optimization:

```python
class QueryOptimizationDebugMixin:
    """Debug query optimization"""
    
    def get_queryset(self, request):
        """Debug queryset optimization"""
        queryset = super().get_queryset(request)
        
        if settings.DEBUG:
            self.debug_queryset_optimization(queryset, request)
        
        return queryset
    
    def debug_queryset_optimization(self, queryset, request):
        """Analyze queryset for optimization opportunities"""
        logger = logging.getLogger('query_optimization')
        
        # Check for select_related opportunities
        model_fields = [f.name for f in self.model._meta.fields if f.is_relation]
        if model_fields and not queryset.query.select_related:
            logger.warning(f"Consider using select_related for: {model_fields}")
        
        # Check for prefetch_related opportunities
        m2m_fields = [f.name for f in self.model._meta.many_to_many]
        reverse_fk_fields = [f.get_accessor_name() for f in self.model._meta.get_fields() 
                           if f.is_relation and f.one_to_many]
        
        prefetch_candidates = m2m_fields + reverse_fk_fields
        if prefetch_candidates and not queryset._prefetch_related_lookups:
            logger.warning(f"Consider using prefetch_related for: {prefetch_candidates}")
        
        # Analyze filters
        filters = dict(request.GET)
        if filters:
            logger.info(f"Applied filters: {filters}")
            
            # Check for indexed fields
            indexed_fields = [f.name for f in self.model._meta.fields if f.db_index]
            for filter_key in filters.keys():
                field_name = filter_key.split('__')[0]
                if field_name not in indexed_fields:
                    logger.warning(f"Filter on non-indexed field: {field_name}")
```

## Error Debugging

### Exception Handling

Enhanced exception handling for debugging:

```python
from binder.exceptions import BinderException
import traceback
import logging

logger = logging.getLogger('error_debug')

class ErrorDebugMixin:
    """Enhanced error debugging"""
    
    def dispatch(self, request, *args, **kwargs):
        try:
            return super().dispatch(request, *args, **kwargs)
        except Exception as e:
            self.log_exception(e, request, *args, **kwargs)
            raise
    
    def log_exception(self, exception, request, *args, **kwargs):
        """Log detailed exception information"""
        error_info = {
            'exception_type': type(exception).__name__,
            'exception_message': str(exception),
            'view_class': self.__class__.__name__,
            'request_method': request.method,
            'request_path': request.path,
            'request_user': request.user.username if request.user.is_authenticated else 'anonymous',
            'request_data': self.get_safe_request_data(request),
            'args': args,
            'kwargs': kwargs,
            'traceback': traceback.format_exc(),
        }
        
        logger.error(f"Exception in {self.__class__.__name__}: {error_info}")
    
    def get_safe_request_data(self, request):
        """Get request data safely (without sensitive information)"""
        try:
            if request.method in ['POST', 'PUT', 'PATCH']:
                data = self._get_request_data(request)
                # Remove sensitive fields
                sensitive_fields = ['password', 'token', 'secret']
                safe_data = {}
                for key, value in data.items():
                    if any(sensitive in key.lower() for sensitive in sensitive_fields):
                        safe_data[key] = '[REDACTED]'
                    else:
                        safe_data[key] = value
                return safe_data
        except Exception:
            return '[Could not parse request data]'
        
        return dict(request.GET)

class ArticleView(ErrorDebugMixin, ModelView):
    model = Article
```

### Validation Error Debugging

Debug validation errors:

```python
from django.core.exceptions import ValidationError

class ValidationDebugMixin:
    """Debug validation errors"""
    
    def validate_request_data(self, request, data):
        """Enhanced validation with debugging"""
        try:
            return super().validate_request_data(request, data)
        except ValidationError as e:
            self.log_validation_error(e, data, request)
            raise
    
    def log_validation_error(self, error, data, request):
        """Log detailed validation error information"""
        logger = logging.getLogger('validation_debug')
        
        validation_info = {
            'error_message': str(error),
            'error_dict': error.message_dict if hasattr(error, 'message_dict') else None,
            'validated_data': data,
            'model': self.model.__name__,
            'user': request.user.username if request.user.is_authenticated else 'anonymous',
        }
        
        logger.warning(f"Validation error in {self.__class__.__name__}: {validation_info}")
        
        # Analyze common validation issues
        if hasattr(error, 'message_dict'):
            for field, messages in error.message_dict.items():
                field_obj = self.model._meta.get_field(field)
                logger.debug(f"Field {field} ({field_obj.__class__.__name__}): {messages}")

class ArticleView(ValidationDebugMixin, ModelView):
    model = Article
```

## Performance Debugging

### Performance Profiling

Profile view performance:

```python
import cProfile
import pstats
import io
from django.http import JsonResponse

class PerformanceDebugMixin:
    """Performance profiling for views"""
    
    def dispatch(self, request, *args, **kwargs):
        if settings.DEBUG and request.GET.get('profile'):
            return self.profile_request(request, *args, **kwargs)
        
        return super().dispatch(request, *args, **kwargs)
    
    def profile_request(self, request, *args, **kwargs):
        """Profile request execution"""
        profiler = cProfile.Profile()
        profiler.enable()
        
        try:
            response = super().dispatch(request, *args, **kwargs)
        finally:
            profiler.disable()
        
        # Generate profile report
        s = io.StringIO()
        ps = pstats.Stats(profiler, stream=s)
        ps.sort_stats('cumulative')
        ps.print_stats(20)  # Top 20 functions
        
        profile_output = s.getvalue()
        
        # Log profile results
        logger = logging.getLogger('performance_debug')
        logger.info(f"Profile for {request.path}:\n{profile_output}")
        
        # Add profile info to response headers
        if hasattr(response, '__setitem__'):
            response['X-Profile-Available'] = 'true'
        
        return response

class ArticleView(PerformanceDebugMixin, ModelView):
    model = Article
```

### Memory Usage Debugging

Debug memory usage:

```python
import psutil
import os
import gc

class MemoryDebugMixin:
    """Debug memory usage"""
    
    def dispatch(self, request, *args, **kwargs):
        if settings.DEBUG:
            initial_memory = self.get_memory_usage()
        
        response = super().dispatch(request, *args, **kwargs)
        
        if settings.DEBUG:
            final_memory = self.get_memory_usage()
            memory_diff = final_memory - initial_memory
            
            if memory_diff > 10:  # More than 10MB increase
                logger = logging.getLogger('memory_debug')
                logger.warning(
                    f"High memory usage in {self.__class__.__name__}: "
                    f"{memory_diff:.2f}MB increase"
                )
                
                # Force garbage collection and log again
                gc.collect()
                after_gc_memory = self.get_memory_usage()
                logger.info(f"Memory after GC: {after_gc_memory:.2f}MB")
        
        return response
    
    def get_memory_usage(self):
        """Get current memory usage in MB"""
        process = psutil.Process(os.getpid())
        return process.memory_info().rss / 1024 / 1024

class ArticleView(MemoryDebugMixin, ModelView):
    model = Article
```

## Production Debugging

### Production Error Logging

Configure production error logging:

```python
# settings/production.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
        'json': {
            'format': '{"level": "{levelname}", "time": "{asctime}", "module": "{module}", "message": "{message}"}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'ERROR',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': '/var/log/django/error.log',
            'maxBytes': 1024*1024*10,  # 10MB
            'backupCount': 5,
            'formatter': 'json',
        },
        'mail_admins': {
            'level': 'ERROR',
            'class': 'django.utils.log.AdminEmailHandler',
            'formatter': 'verbose',
        },
        'sentry': {
            'level': 'ERROR',
            'class': 'sentry_sdk.integrations.logging.EventHandler',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['file', 'mail_admins'],
            'level': 'ERROR',
            'propagate': True,
        },
        'binder': {
            'handlers': ['file', 'sentry'],
            'level': 'ERROR',
            'propagate': True,
        },
    },
}
```

### Health Check Debugging

Create health check endpoints:

```python
from django.http import JsonResponse
from django.db import connection
from django.core.cache import cache
import time

def health_check(request):
    """Comprehensive health check"""
    health_status = {
        'status': 'healthy',
        'timestamp': time.time(),
        'checks': {}
    }
    
    # Database check
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        health_status['checks']['database'] = 'ok'
    except Exception as e:
        health_status['checks']['database'] = f'error: {str(e)}'
        health_status['status'] = 'unhealthy'
    
    # Cache check
    try:
        cache.set('health_check', 'ok', 10)
        if cache.get('health_check') == 'ok':
            health_status['checks']['cache'] = 'ok'
        else:
            health_status['checks']['cache'] = 'error: cache not working'
            health_status['status'] = 'unhealthy'
    except Exception as e:
        health_status['checks']['cache'] = f'error: {str(e)}'
        health_status['status'] = 'unhealthy'
    
    # Memory check
    try:
        process = psutil.Process(os.getpid())
        memory_mb = process.memory_info().rss / 1024 / 1024
        health_status['checks']['memory'] = f'{memory_mb:.2f}MB'
        
        if memory_mb > 1000:  # More than 1GB
            health_status['checks']['memory'] += ' (high)'
    except Exception as e:
        health_status['checks']['memory'] = f'error: {str(e)}'
    
    status_code = 200 if health_status['status'] == 'healthy' else 503
    return JsonResponse(health_status, status=status_code)

# Add to urls.py
urlpatterns += [
    re_path(r'^health/$', health_check, name='health_check'),
]
```

## Debug Tools

### Django Debug Toolbar

Configure Django Debug Toolbar:

```python
# settings/development.py
if DEBUG:
    INSTALLED_APPS += ['debug_toolbar']
    MIDDLEWARE += ['debug_toolbar.middleware.DebugToolbarMiddleware']
    
    DEBUG_TOOLBAR_CONFIG = {
        'SHOW_TOOLBAR_CALLBACK': lambda request: True,
        'SHOW_COLLAPSED': True,
        'INTERCEPT_REDIRECTS': False,
    }
    
    DEBUG_TOOLBAR_PANELS = [
        'debug_toolbar.panels.versions.VersionsPanel',
        'debug_toolbar.panels.timer.TimerPanel',
        'debug_toolbar.panels.settings.SettingsPanel',
        'debug_toolbar.panels.headers.HeadersPanel',
        'debug_toolbar.panels.request.RequestPanel',
        'debug_toolbar.panels.sql.SQLPanel',
        'debug_toolbar.panels.staticfiles.StaticFilesPanel',
        'debug_toolbar.panels.templates.TemplatesPanel',
        'debug_toolbar.panels.cache.CachePanel',
        'debug_toolbar.panels.signals.SignalsPanel',
        'debug_toolbar.panels.logging.LoggingPanel',
        'debug_toolbar.panels.redirects.RedirectsPanel',
        'debug_toolbar.panels.profiling.ProfilingPanel',
    ]
```

### Custom Debug Commands

Create custom management commands for debugging:

```python
# management/commands/debug_queries.py
from django.core.management.base import BaseCommand
from django.db import connection
from myapp.models import Article

class Command(BaseCommand):
    help = 'Debug database queries for specific operations'
    
    def add_arguments(self, parser):
        parser.add_argument('--operation', type=str, help='Operation to debug')
        parser.add_argument('--count', type=int, default=10, help='Number of objects to test')
    
    def handle(self, *args, **options):
        operation = options['operation']
        count = options['count']
        
        # Reset query log
        connection.queries_log.clear()
        
        if operation == 'list':
            self.debug_list_operation(count)
        elif operation == 'detail':
            self.debug_detail_operation()
        elif operation == 'create':
            self.debug_create_operation(count)
        
        # Report queries
        queries = connection.queries
        total_time = sum(float(q['time']) for q in queries)
        
        self.stdout.write(f"Operation: {operation}")
        self.stdout.write(f"Queries executed: {len(queries)}")
        self.stdout.write(f"Total time: {total_time:.3f}s")
        
        for i, query in enumerate(queries, 1):
            self.stdout.write(f"Query {i} ({query['time']}s): {query['sql'][:100]}...")
    
    def debug_list_operation(self, count):
        """Debug list operation"""
        articles = list(Article.objects.all()[:count])
        for article in articles:
            # Access related fields to trigger queries
            _ = article.author.username
            _ = article.category.name
    
    def debug_detail_operation(self):
        """Debug detail operation"""
        article = Article.objects.first()
        if article:
            _ = article.author.username
            _ = article.category.name
            _ = list(article.tags.all())
    
    def debug_create_operation(self, count):
        """Debug create operation"""
        from django.contrib.auth.models import User
        user = User.objects.first()
        category = Category.objects.first()
        
        for i in range(count):
            Article.objects.create(
                title=f'Debug Article {i}',
                content='Debug content',
                author=user,
                category=category
            )
```

### API Debug Endpoints

Create debug endpoints for API testing:

```python
from binder.router import list_route

class DebugView(ModelView):
    model = Article
    
    @list_route(name='debug_sql', unauthenticated=True)
    def debug_sql(self, request):
        """Debug SQL queries for this view"""
        if not settings.DEBUG:
            return JsonResponse({'error': 'Debug mode disabled'}, status=403)
        
        # Reset query log
        connection.queries_log.clear()
        
        # Execute normal list operation
        queryset = self.get_queryset(request)
        data = [self.serialize_object(obj, request) for obj in queryset[:10]]
        
        # Get query information
        queries = connection.queries
        query_info = [
            {
                'sql': query['sql'],
                'time': query['time'],
            }
            for query in queries
        ]
        
        return JsonResponse({
            'data_count': len(data),
            'query_count': len(queries),
            'total_time': sum(float(q['time']) for q in queries),
            'queries': query_info,
        })
    
    @list_route(name='debug_permissions', unauthenticated=True)
    def debug_permissions(self, request):
        """Debug permission system"""
        if not settings.DEBUG:
            return JsonResponse({'error': 'Debug mode disabled'}, status=403)
        
        permission_info = {
            'user': {
                'authenticated': request.user.is_authenticated,
                'username': request.user.username if request.user.is_authenticated else None,
                'is_staff': request.user.is_staff if request.user.is_authenticated else False,
                'permissions': list(request.user.get_all_permissions()) if request.user.is_authenticated else [],
            },
            'view_permissions': {},
        }
        
        # Check each permission type
        for perm_type in ['view', 'add', 'change', 'delete']:
            try:
                scopes = self._require_model_perm(perm_type, request)
                permission_info['view_permissions'][perm_type] = scopes
            except Exception as e:
                permission_info['view_permissions'][perm_type] = f'Error: {str(e)}'
        
        return JsonResponse(permission_info)
```

This comprehensive debugging guide provides tools and techniques for identifying and resolving issues in Django Binder applications at all stages of development and deployment.
