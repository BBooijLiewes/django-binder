# Performance Optimization

This guide covers comprehensive performance optimization strategies for Django Binder applications, from database queries to caching and scaling.

## Database Optimization

### Query Optimization

Optimize database queries for better performance:

```python
class ArticleView(ModelView):
    model = Article
    
    def get_queryset(self, request):
        """Optimized queryset with proper prefetching"""
        queryset = super().get_queryset(request)
        
        # Use select_related for foreign keys
        queryset = queryset.select_related(
            'author',
            'category',
            'author__profile'
        )
        
        # Use prefetch_related for many-to-many and reverse foreign keys
        queryset = queryset.prefetch_related(
            'tags',
            'comments__author',
            'attachments'
        )
        
        # Add useful annotations to avoid additional queries
        queryset = queryset.annotate(
            comment_count=Count('comments'),
            tag_count=Count('tags'),
            view_count_total=Sum('views__count')
        )
        
        return queryset
```

### Database Indexes

Optimize database performance with proper indexing:

```python
class Article(BinderModel):
    title = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(unique=True)  # Automatically indexed
    published = models.BooleanField(default=False, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    view_count = models.PositiveIntegerField(default=0, db_index=True)
    
    class Meta:
        # Composite indexes for common query patterns
        indexes = [
            models.Index(fields=['published', '-created_at']),
            models.Index(fields=['author', 'published']),
            models.Index(fields=['category', 'published', '-created_at']),
            models.Index(fields=['-view_count', 'published']),
        ]
        
        # Database constraints
        constraints = [
            models.CheckConstraint(
                check=models.Q(view_count__gte=0),
                name='positive_view_count'
            )
        ]
```

### Query Analysis

Analyze and optimize slow queries:

```python
from django.db import connection
from django.conf import settings
import logging

logger = logging.getLogger('performance')

class QueryAnalysisMixin:
    """Mixin to analyze query performance"""
    
    def get_queryset(self, request):
        queryset = super().get_queryset(request)
        
        if settings.DEBUG:
            # Log query analysis in development
            self.log_query_analysis(queryset)
        
        return queryset
    
    def log_query_analysis(self, queryset):
        """Log query analysis information"""
        # Get the SQL query
        sql_query = str(queryset.query)
        
        # Count queries before and after
        initial_queries = len(connection.queries)
        
        # Execute a sample to see actual query count
        list(queryset[:1])  # Execute one item
        
        final_queries = len(connection.queries)
        query_count = final_queries - initial_queries
        
        logger.info(f'Query analysis for {self.model.__name__}:')
        logger.info(f'SQL: {sql_query}')
        logger.info(f'Query count: {query_count}')
        
        # Log slow queries
        if query_count > 5:
            logger.warning(f'High query count ({query_count}) for {self.model.__name__}')

class ArticleView(QueryAnalysisMixin, ModelView):
    model = Article
```

### Bulk Operations

Use bulk operations for better performance:

```python
class ArticleView(ModelView):
    model = Article
    
    @list_route(name='bulk_update', methods=['POST'])
    def bulk_update(self, request):
        """Bulk update articles efficiently"""
        data = self._get_request_data(request)
        article_ids = data.get('article_ids', [])
        updates = data.get('updates', {})
        
        # Bulk update using queryset.update()
        updated_count = Article.objects.filter(
            id__in=article_ids
        ).update(**updates)
        
        return JsonResponse({
            'success': True,
            'updated_count': updated_count
        })
    
    @list_route(name='bulk_create', methods=['POST'])
    def bulk_create(self, request):
        """Bulk create articles efficiently"""
        data = self._get_request_data(request)
        articles_data = data.get('articles', [])
        
        # Prepare objects for bulk creation
        articles_to_create = []
        for article_data in articles_data:
            article = Article(**article_data)
            article.full_clean()  # Validate
            articles_to_create.append(article)
        
        # Bulk create
        created_articles = Article.objects.bulk_create(articles_to_create)
        
        return JsonResponse({
            'success': True,
            'created_count': len(created_articles)
        })
```

## Caching Strategies

### Response Caching

Implement response caching for frequently accessed data:

```python
from django.core.cache import cache
from django.utils.cache import make_template_fragment_key
import hashlib

class CachedModelView(ModelView):
    cache_timeout = 300  # 5 minutes
    cache_key_prefix = 'api_cache'
    
    def get_cache_key(self, request, *args, **kwargs):
        """Generate cache key for request"""
        # Include relevant request parameters
        cache_params = {
            'model': self.model.__name__,
            'user_id': request.user.id if request.user.is_authenticated else 'anonymous',
            'query_params': sorted(request.GET.items()),
            'method': request.method,
        }
        
        # Create hash of parameters
        cache_string = str(cache_params)
        cache_hash = hashlib.md5(cache_string.encode()).hexdigest()
        
        return f"{self.cache_key_prefix}:{cache_hash}"
    
    def get_list_response(self, request, queryset):
        """Cached list response"""
        cache_key = self.get_cache_key(request)
        
        # Try to get from cache
        cached_response = cache.get(cache_key)
        if cached_response:
            return cached_response
        
        # Generate response
        response = super().get_list_response(request, queryset)
        
        # Cache the response
        cache.set(cache_key, response, self.cache_timeout)
        
        return response
    
    def store(self, obj, fields, request):
        """Clear cache on updates"""
        result = super().store(obj, fields, request)
        
        # Clear related caches
        cache_pattern = f"{self.cache_key_prefix}:*"
        self.clear_cache_pattern(cache_pattern)
        
        return result
    
    def clear_cache_pattern(self, pattern):
        """Clear cache entries matching pattern"""
        # Implementation depends on cache backend
        # For Redis:
        from django.core.cache import cache
        if hasattr(cache, 'delete_pattern'):
            cache.delete_pattern(pattern)

class ArticleView(CachedModelView):
    model = Article
    cache_timeout = 600  # 10 minutes for articles
```

### Query Result Caching

Cache expensive database queries:

```python
class QueryCacheMixin:
    query_cache_timeout = 300
    
    def get_queryset(self, request):
        """Cached queryset"""
        # Create cache key from query parameters
        cache_key = self.get_queryset_cache_key(request)
        
        # Try to get cached result IDs
        cached_ids = cache.get(cache_key)
        if cached_ids is not None:
            # Return queryset filtered by cached IDs
            return self.model.objects.filter(id__in=cached_ids)
        
        # Generate queryset
        queryset = super().get_queryset(request)
        
        # Cache the result IDs
        result_ids = list(queryset.values_list('id', flat=True))
        cache.set(cache_key, result_ids, self.query_cache_timeout)
        
        return queryset
    
    def get_queryset_cache_key(self, request):
        """Generate cache key for queryset"""
        params = {
            'model': self.model.__name__,
            'filters': sorted(request.GET.items()),
            'user_permissions': self.get_user_permission_hash(request.user)
        }
        
        return f"queryset_cache:{hashlib.md5(str(params).encode()).hexdigest()}"
    
    def get_user_permission_hash(self, user):
        """Get hash of user permissions for cache key"""
        if user.is_anonymous:
            return 'anonymous'
        
        permissions = user.get_all_permissions()
        return hashlib.md5(str(sorted(permissions)).encode()).hexdigest()

class ArticleView(QueryCacheMixin, ModelView):
    model = Article
```

### Template Fragment Caching

Cache expensive template fragments:

```python
from django.core.cache import cache
from django.template.loader import render_to_string

class TemplateFragmentMixin:
    def get_list_response(self, request, queryset):
        """Response with cached template fragments"""
        response = super().get_list_response(request, queryset)
        
        # Add cached template fragments to response
        if hasattr(response, 'data') and isinstance(response.data, dict):
            # Cache expensive computed data
            cache_key = f"article_stats:{request.user.id}"
            stats = cache.get(cache_key)
            
            if stats is None:
                stats = self.compute_user_stats(request.user)
                cache.set(cache_key, stats, 300)  # 5 minutes
            
            response.data['user_stats'] = stats
        
        return response
    
    def compute_user_stats(self, user):
        """Compute expensive user statistics"""
        return {
            'total_articles': Article.objects.filter(author=user).count(),
            'published_articles': Article.objects.filter(author=user, published=True).count(),
            'total_views': Article.objects.filter(author=user).aggregate(
                total=Sum('view_count')
            )['total'] or 0,
        }
```

## Memory Optimization

### Efficient Data Loading

Optimize memory usage when loading large datasets:

```python
class MemoryEfficientView(ModelView):
    def get_list_response(self, request, queryset):
        """Memory-efficient list response"""
        # Use iterator for large datasets
        limit = int(request.GET.get('limit', 20))
        offset = int(request.GET.get('offset', 0))
        
        if limit > 1000:  # Large dataset
            # Use iterator to avoid loading all objects into memory
            objects = []
            for obj in queryset[offset:offset + limit].iterator():
                objects.append(self.serialize_object(obj, request))
            
            return JsonResponse({
                'data': objects,
                'meta': {
                    'limit': limit,
                    'offset': offset,
                    'total_records': queryset.count()
                }
            })
        
        return super().get_list_response(request, queryset)
    
    def serialize_object(self, obj, request):
        """Memory-efficient object serialization"""
        # Only serialize required fields
        data = {}
        
        for field in self.get_shown_fields():
            if hasattr(obj, field):
                value = getattr(obj, field)
                
                # Handle different field types efficiently
                if isinstance(value, models.Model):
                    # Only include ID for related objects
                    data[field] = value.id
                elif hasattr(value, 'all'):  # Many-to-many
                    # Use values_list for efficiency
                    data[field] = list(value.values_list('id', flat=True))
                else:
                    data[field] = value
        
        return data
```

### Lazy Loading

Implement lazy loading for expensive operations:

```python
class LazyLoadingMixin:
    def serialize_object(self, obj, request):
        """Serialize with lazy loading"""
        data = super().serialize_object(obj, request)
        
        # Add lazy-loaded fields
        lazy_fields = self.get_lazy_fields()
        for field_name in lazy_fields:
            # Add placeholder for lazy-loaded data
            data[f'{field_name}_url'] = f'/api/{self.model.__name__.lower()}/{obj.id}/{field_name}/'
        
        return data
    
    def get_lazy_fields(self):
        """Get fields that should be lazy-loaded"""
        return getattr(self, 'lazy_fields', [])
    
    @detail_route(name='comments')
    def get_comments(self, request, pk):
        """Lazy-load comments"""
        article = self.get_object(pk)
        comments = article.comments.select_related('author')[:10]
        
        return JsonResponse({
            'comments': [
                {
                    'id': comment.id,
                    'content': comment.content,
                    'author': comment.author.username,
                    'created_at': comment.created_at.isoformat()
                }
                for comment in comments
            ]
        })

class ArticleView(LazyLoadingMixin, ModelView):
    model = Article
    lazy_fields = ['comments', 'related_articles']
```

## Connection Pooling

### Database Connection Optimization

Optimize database connections:

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'your_database',
        'USER': 'your_user',
        'PASSWORD': 'your_password',
        'HOST': 'localhost',
        'PORT': '5432',
        'OPTIONS': {
            'MAX_CONNS': 20,
            'MIN_CONNS': 5,
        },
        'CONN_MAX_AGE': 600,  # 10 minutes
    }
}

# For production with connection pooling
# pip install django-db-pool
DATABASES['default']['ENGINE'] = 'django_db_pool.backends.postgresql'
DATABASES['default']['POOL_OPTIONS'] = {
    'POOL_SIZE': 20,
    'MAX_OVERFLOW': 30,
    'RECYCLE': 24 * 60 * 60,  # 24 hours
}
```

### Redis Connection Pooling

Optimize Redis connections for caching:

```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'CONNECTION_POOL_KWARGS': {
                'max_connections': 50,
                'retry_on_timeout': True,
            }
        }
    }
}
```

## Async Support

### Async Views

Implement async views for I/O-bound operations:

```python
import asyncio
from django.http import JsonResponse
from asgiref.sync import sync_to_async

class AsyncModelView(ModelView):
    async def get_list_response_async(self, request, queryset):
        """Async list response"""
        # Convert sync operations to async
        count = await sync_to_async(queryset.count)()
        objects = await sync_to_async(list)(queryset[:20])
        
        # Parallel processing of objects
        tasks = [
            self.serialize_object_async(obj, request)
            for obj in objects
        ]
        
        serialized_objects = await asyncio.gather(*tasks)
        
        return JsonResponse({
            'data': serialized_objects,
            'meta': {
                'total_records': count
            }
        })
    
    async def serialize_object_async(self, obj, request):
        """Async object serialization"""
        # Perform async operations
        data = await sync_to_async(self.serialize_object)(obj, request)
        
        # Add async-computed fields
        if hasattr(self, 'get_async_fields'):
            async_data = await self.get_async_fields(obj, request)
            data.update(async_data)
        
        return data
    
    async def get_async_fields(self, obj, request):
        """Get fields that require async computation"""
        # Example: fetch data from external API
        async with aiohttp.ClientSession() as session:
            async with session.get(f'https://api.example.com/data/{obj.id}') as response:
                external_data = await response.json()
        
        return {
            'external_rating': external_data.get('rating', 0)
        }
```

## Monitoring and Profiling

### Performance Monitoring

Monitor API performance:

```python
import time
import logging
from django.utils.deprecation import MiddlewareMixin

logger = logging.getLogger('performance')

class PerformanceMonitoringMiddleware(MiddlewareMixin):
    def process_request(self, request):
        request._start_time = time.time()
        request._queries_before = len(connection.queries)
    
    def process_response(self, request, response):
        if hasattr(request, '_start_time'):
            duration = time.time() - request._start_time
            query_count = len(connection.queries) - request._queries_before
            
            # Log slow requests
            if duration > 1.0:  # Slower than 1 second
                logger.warning(
                    f'Slow request: {request.method} {request.path} - '
                    f'Duration: {duration:.3f}s, Queries: {query_count}'
                )
            
            # Add performance headers
            response['X-Response-Time'] = f'{duration:.3f}'
            response['X-Query-Count'] = str(query_count)
        
        return response

# Add to MIDDLEWARE
MIDDLEWARE += ['your_app.middleware.PerformanceMonitoringMiddleware']
```

### Query Profiling

Profile database queries:

```python
from django.db import connection
from django.conf import settings
import logging

class QueryProfilerMixin:
    def dispatch(self, request, *args, **kwargs):
        if settings.DEBUG:
            # Reset query log
            connection.queries_log.clear()
        
        response = super().dispatch(request, *args, **kwargs)
        
        if settings.DEBUG:
            self.log_query_profile()
        
        return response
    
    def log_query_profile(self):
        """Log query profiling information"""
        queries = connection.queries
        total_time = sum(float(query['time']) for query in queries)
        
        logger = logging.getLogger('query_profile')
        logger.info(f'Query Profile for {self.__class__.__name__}:')
        logger.info(f'Total queries: {len(queries)}')
        logger.info(f'Total time: {total_time:.3f}s')
        
        # Log slow queries
        slow_queries = [q for q in queries if float(q['time']) > 0.1]
        for query in slow_queries:
            logger.warning(f'Slow query ({query["time"]}s): {query["sql"][:200]}...')

class ArticleView(QueryProfilerMixin, ModelView):
    model = Article
```

## Load Testing

### API Load Testing

Test API performance under load:

```python
# load_test.py
import asyncio
import aiohttp
import time
from statistics import mean, median

async def make_request(session, url, headers=None):
    """Make a single API request"""
    start_time = time.time()
    try:
        async with session.get(url, headers=headers) as response:
            await response.text()
            return {
                'status': response.status,
                'duration': time.time() - start_time,
                'success': response.status == 200
            }
    except Exception as e:
        return {
            'status': 0,
            'duration': time.time() - start_time,
            'success': False,
            'error': str(e)
        }

async def load_test(url, concurrent_requests=10, total_requests=100):
    """Run load test against API endpoint"""
    headers = {'Authorization': 'Token your-token-here'}
    
    async with aiohttp.ClientSession() as session:
        # Create semaphore to limit concurrent requests
        semaphore = asyncio.Semaphore(concurrent_requests)
        
        async def bounded_request():
            async with semaphore:
                return await make_request(session, url, headers)
        
        # Run all requests
        tasks = [bounded_request() for _ in range(total_requests)]
        results = await asyncio.gather(*tasks)
    
    # Analyze results
    successful_requests = [r for r in results if r['success']]
    failed_requests = [r for r in results if not r['success']]
    
    durations = [r['duration'] for r in successful_requests]
    
    print(f"Load Test Results for {url}:")
    print(f"Total requests: {total_requests}")
    print(f"Successful requests: {len(successful_requests)}")
    print(f"Failed requests: {len(failed_requests)}")
    print(f"Success rate: {len(successful_requests)/total_requests*100:.1f}%")
    
    if durations:
        print(f"Average response time: {mean(durations):.3f}s")
        print(f"Median response time: {median(durations):.3f}s")
        print(f"Min response time: {min(durations):.3f}s")
        print(f"Max response time: {max(durations):.3f}s")

# Run load test
if __name__ == '__main__':
    asyncio.run(load_test('http://localhost:8000/api/article/', 20, 200))
```

### Performance Benchmarking

Benchmark different optimization strategies:

```python
import time
import statistics
from django.test import TestCase
from django.test.utils import override_settings

class PerformanceBenchmarkTest(TestCase):
    def setUp(self):
        # Create test data
        self.create_test_data()
    
    def create_test_data(self):
        """Create test data for benchmarking"""
        users = [User.objects.create_user(f'user{i}', f'user{i}@test.com', 'password') 
                for i in range(10)]
        
        categories = [Category.objects.create(name=f'Category {i}') 
                     for i in range(5)]
        
        for i in range(100):
            article = Article.objects.create(
                title=f'Article {i}',
                content=f'Content for article {i}' * 50,
                author=users[i % len(users)],
                category=categories[i % len(categories)],
                published=i % 2 == 0
            )
    
    def benchmark_query(self, query_func, iterations=10):
        """Benchmark a query function"""
        times = []
        
        for _ in range(iterations):
            start_time = time.time()
            result = query_func()
            end_time = time.time()
            
            times.append(end_time - start_time)
        
        return {
            'mean': statistics.mean(times),
            'median': statistics.median(times),
            'min': min(times),
            'max': max(times),
            'std_dev': statistics.stdev(times) if len(times) > 1 else 0
        }
    
    def test_query_optimization(self):
        """Benchmark query optimization strategies"""
        
        # Unoptimized query
        def unoptimized_query():
            articles = list(Article.objects.all()[:20])
            for article in articles:
                _ = article.author.username  # N+1 query
                _ = article.category.name    # N+1 query
            return articles
        
        # Optimized query
        def optimized_query():
            return list(Article.objects.select_related(
                'author', 'category'
            ).all()[:20])
        
        unoptimized_stats = self.benchmark_query(unoptimized_query)
        optimized_stats = self.benchmark_query(optimized_query)
        
        print(f"Unoptimized query: {unoptimized_stats['mean']:.3f}s")
        print(f"Optimized query: {optimized_stats['mean']:.3f}s")
        print(f"Improvement: {unoptimized_stats['mean']/optimized_stats['mean']:.1f}x faster")
        
        # Assert optimization is significantly better
        self.assertLess(optimized_stats['mean'], unoptimized_stats['mean'] * 0.5)
```

This comprehensive performance optimization guide provides strategies for scaling Django Binder applications from development to production environments.
