# Routing System

Django Binder's routing system automatically discovers and registers API endpoints from your ModelView classes, providing a convention-over-configuration approach to URL management.

## Router Basics

### Automatic Router Setup

The simplest way to set up routing is using the bootstrap method:

```python
# urls.py
import binder.router

# Automatic discovery and registration
router = binder.router.Router.bootstrap()

urlpatterns = [
    re_path(r'^api/', include(router.urls)),
    re_path(r'^api/', binder.views.api_catchall, name='api_catchall'),
]
```

### Manual Router Setup

For more control, create and configure the router manually:

```python
# urls.py
import binder.router
import binder.views

# Create router and register base view
router = binder.router.Router().register(binder.views.ModelView)

urlpatterns = [
    re_path(r'^api/', include(router.urls)),
    re_path(r'^api/', binder.views.api_catchall, name='api_catchall'),
]
```

## URL Patterns

Django Binder automatically generates the following URL patterns for each registered model:

### Standard CRUD Endpoints

```python
# For a model named 'Article'
GET    /api/article/              # List all articles
POST   /api/article/              # Create new article
GET    /api/article/{id}/         # Get specific article
PUT    /api/article/{id}/         # Update specific article
DELETE /api/article/{id}/         # Delete specific article
POST   /api/article/{id}/         # Undelete specific article
```

### File Field Endpoints

```python
# For file fields on models
GET    /api/article/{id}/image/   # Download file
POST   /api/article/{id}/image/   # Upload file
```

### History Endpoints

```python
# For models with history enabled
GET    /api/article/{id}/history/       # View change history
GET    /api/article/{id}/history/debug/ # Debug history view
```

## Route Configuration

### Basic Route Configuration

Control how your models are routed:

```python
from binder.views import ModelView
from binder.router import Route

class ArticleView(ModelView):
    model = Article
    route = True  # Use default route (model name)

class CategoryView(ModelView):
    model = Category
    route = 'categories'  # Custom route name

class TagView(ModelView):
    model = Tag
    route = Route('tags', list_endpoint=True, detail_endpoint=False)
    # Only list endpoint, no detail endpoints
```

### Route Objects

Use Route objects for fine-grained control:

```python
from binder.router import Route

class SpecialView(ModelView):
    model = MyModel
    route = Route(
        route='special',
        list_endpoint=True,    # Enable GET /api/special/
        detail_endpoint=False  # Disable GET /api/special/{id}/
    )
```

### Disabling Routes

Disable automatic routing for base classes:

```python
class BaseView(ModelView):
    model = None  # No model = no automatic routing
    route = None  # Explicitly disable routing
    
    # Shared logic here
    def get_queryset(self, request):
        return self.model.objects.filter(active=True)

class ArticleView(BaseView):
    model = Article
    route = True  # Enable routing for this subclass
```

## Custom Endpoints

### List Routes

Add custom endpoints that operate on collections:

```python
from binder.router import list_route

class ArticleView(ModelView):
    model = Article
    
    @list_route(name='published')
    def published_articles(self, request):
        """GET /api/article/published/"""
        queryset = self.get_queryset(request).filter(published=True)
        return self.get_list_response(request, queryset)
    
    @list_route(name='stats', methods=['GET'])
    def article_stats(self, request):
        """GET /api/article/stats/"""
        return JsonResponse({
            'total': self.get_queryset(request).count(),
            'published': self.get_queryset(request).filter(published=True).count()
        })
```

### Detail Routes

Add custom endpoints that operate on specific instances:

```python
from binder.router import detail_route

class ArticleView(ModelView):
    model = Article
    
    @detail_route(name='publish', methods=['POST'])
    def publish_article(self, request, pk):
        """POST /api/article/{id}/publish/"""
        article = self.get_object(pk)
        article.published = True
        article.save()
        return JsonResponse({'success': True})
    
    @detail_route(name='related')
    def get_related(self, request, pk):
        """GET /api/article/{id}/related/"""
        article = self.get_object(pk)
        related = Article.objects.filter(
            category=article.category
        ).exclude(id=article.id)[:5]
        return self.get_list_response(request, related)
```

### Route Parameters

Add URL parameters to custom routes:

```python
class ArticleView(ModelView):
    model = Article
    
    @list_route(
        name='by_category',
        extra_route=r'(?P<category_slug>[^/]+)/'
    )
    def articles_by_category(self, request, category_slug):
        """GET /api/article/by_category/{category_slug}/"""
        queryset = self.get_queryset(request).filter(
            category__slug=category_slug
        )
        return self.get_list_response(request, queryset)
    
    @detail_route(
        name='comments',
        extra_route=r'(?P<status>approved|pending)/'
    )
    def get_comments(self, request, pk, status):
        """GET /api/article/{id}/comments/{status}/"""
        article = self.get_object(pk)
        comments = article.comments.filter(status=status)
        return self.get_list_response(request, comments)
```

### Unauthenticated Routes

Allow unauthenticated access to specific endpoints:

```python
class ArticleView(ModelView):
    model = Article
    
    @list_route(name='public', unauthenticated=True)
    def public_articles(self, request):
        """Public endpoint - no authentication required"""
        queryset = self.get_queryset(request).filter(
            published=True,
            public=True
        )
        return self.get_list_response(request, queryset)
```

## Route Discovery

### Automatic View Discovery

The router automatically discovers views in your Django apps:

```python
# Router.bootstrap() automatically imports views from:
# - myapp.views
# - myapp.api.views  
# - myapp.api

# Manual discovery
router = binder.router.Router()
for app in apps.get_app_configs():
    try:
        import_module('.views', app.name)
    except ImportError:
        pass  # No views module
router.register(binder.views.ModelView)
```

### Manual View Registration

Register views manually for more control:

```python
router = binder.router.Router()

# Register specific views
router.register(ArticleView)
router.register(CategoryView)
router.register(TagView)

# Register view hierarchies
router.register(BaseView)  # Registers all subclasses
```

## Advanced Routing Patterns

### Versioned APIs

Implement API versioning:

```python
# v1/views.py
class ArticleViewV1(ModelView):
    model = Article
    route = 'v1/article'

# v2/views.py  
class ArticleViewV2(ModelView):
    model = Article
    route = 'v2/article'
    
    def get_queryset(self, request):
        # V2 specific logic
        return super().get_queryset(request).select_related('author')

# urls.py
router = binder.router.Router()
router.register(ArticleViewV1)
router.register(ArticleViewV2)
```

### Nested Resources

Create nested resource patterns:

```python
class CommentView(ModelView):
    model = Comment
    route = None  # Disable default routing
    
    @list_route(name='article_comments', extra_route=r'article/(?P<article_id>\d+)/')
    def comments_for_article(self, request, article_id):
        """GET /api/comment/article/{article_id}/"""
        queryset = self.get_queryset(request).filter(article_id=article_id)
        return self.get_list_response(request, queryset)

# Or use detail routes on the parent
class ArticleView(ModelView):
    model = Article
    
    @detail_route(name='comments')
    def get_comments(self, request, pk):
        """GET /api/article/{id}/comments/"""
        article = self.get_object(pk)
        comments = Comment.objects.filter(article=article)
        return self.get_list_response(request, comments)
```

### Conditional Routing

Route based on conditions:

```python
class ConditionalView(ModelView):
    model = MyModel
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        
        # Conditional route registration
        if settings.FEATURE_ENABLED:
            self.route = 'mymodel'
        else:
            self.route = None  # Disable routing
```

## Route Debugging

### Debug Route Registration

Check which routes are registered:

```python
# In Django shell
from myproject.urls import router

# List all registered routes
for route, view in router.route_views.items():
    print(f"Route: {route.route} -> {view.__name__}")

# List all model mappings
for model, view in router.model_views.items():
    print(f"Model: {model.__name__} -> {view.__name__}")
```

### Route Inspection

Inspect generated URL patterns:

```python
# In Django shell
from django.urls import reverse
from myproject.urls import router

# Get all URL patterns
for pattern in router.urls:
    print(f"Pattern: {pattern.pattern}")
    print(f"Name: {pattern.name}")
    print(f"View: {pattern.callback}")
    print("---")
```

### Route Testing

Test route resolution:

```python
from django.test import TestCase
from django.urls import reverse, resolve

class RouteTest(TestCase):
    def test_article_list_route(self):
        """Test article list route resolves correctly"""
        url = reverse('Article')
        self.assertEqual(url, '/api/article/')
        
        resolver = resolve('/api/article/')
        self.assertEqual(resolver.view_name, 'Article')
    
    def test_custom_route(self):
        """Test custom route resolves correctly"""
        url = reverse('Article.published')
        self.assertEqual(url, '/api/article/published/')
```

## Error Handling

### Route Conflicts

Handle route conflicts:

```python
# This will raise ValueError
class ConflictView1(ModelView):
    model = Model1
    route = 'conflict'

class ConflictView2(ModelView):
    model = Model2
    route = 'conflict'  # Error: route already exists

# Solution: Use unique route names
class ConflictView2(ModelView):
    model = Model2
    route = 'model2'
```

### Missing Routes

Handle missing model views:

```python
# This will raise BinderRequestError when accessed
class MyModel(BinderModel):
    name = models.CharField(max_length=100)

# No corresponding view defined
# Accessing /api/mymodel/ will fail

# Solution: Define the view
class MyModelView(ModelView):
    model = MyModel
```

## Performance Considerations

### Route Caching

Routes are cached after initial registration:

```python
# Routes are built once during startup
router = binder.router.Router.bootstrap()

# URL patterns are cached
urlpatterns = [
    re_path(r'^api/', include(router.urls)),  # Cached
]
```

### Lazy Loading

Views are loaded lazily:

```python
# Views are only instantiated when accessed
class ExpensiveView(ModelView):
    model = MyModel
    
    def __init__(self, *args, **kwargs):
        # This only runs when the view is first accessed
        super().__init__(*args, **kwargs)
        self.expensive_setup()
```

This routing system provides powerful, flexible URL management while maintaining Django Binder's convention-over-configuration philosophy.
