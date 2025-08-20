# Plugin System

Django Binder provides a comprehensive plugin system that extends core functionality with reusable components for authentication, data export, user management, and more.

## Built-in Plugins

### Token Authentication Plugin

Provides token-based authentication with expiration support:

```python
# settings.py
INSTALLED_APPS += ['binder.plugins.token_auth']

# Token configuration
TOKEN_EXPIRY_HOURS = 24 * 7  # 1 week
TOKEN_CLEANUP_DAYS = 30      # Clean up expired tokens after 30 days
```

**Usage:**

```python
# Create user token
from binder.plugins.token_auth.models import Token

token = Token.objects.create(user=user)
print(f"Token: {token.key}")

# Use token in API requests
headers = {'Authorization': f'Token {token.key}'}
response = requests.get('/api/article/', headers=headers)
```

**Management Commands:**

```bash
# Create token for user
python manage.py create_user_token username

# Delete user token
python manage.py delete_user_token username

# Clean up expired tokens
python manage.py cleanup_expired_tokens
```

### User View Plugin

Provides comprehensive user management endpoints:

```python
# settings.py
INSTALLED_APPS += ['binder.plugins.views']

# In your views.py
from binder.plugins.views.userview import UserView

class MyUserView(UserView):
    # Inherits login, logout, password change, etc.
    pass
```

**Available Endpoints:**

```bash
POST /api/user/login/           # User login
POST /api/user/logout/          # User logout
POST /api/user/change_password/ # Change password
POST /api/user/masquerade/      # Masquerade as another user
POST /api/user/unmasquerade/    # Stop masquerading
```

**Login Example:**

```bash
POST /api/user/login/
Content-Type: application/json

{
  "username": "john",
  "password": "secret123"
}

# Response:
{
  "success": true,
  "user": {
    "id": 1,
    "username": "john",
    "email": "john@example.com"
  },
  "token": "abc123def456"
}
```

### CSV Export Plugin

Export model data to CSV format:

```python
from binder.plugins.views.csvexport import CsvExportView

class ArticleView(CsvExportView, ModelView):
    model = Article
    
    # Configure CSV export
    csv_fields = ['id', 'title', 'author__username', 'published', 'created_at']
    csv_filename = 'articles_export.csv'
```

**Usage:**

```bash
GET /api/article/?format=csv
# Returns CSV file download

# Or use dedicated endpoint
GET /api/article/export/csv/
```

### Multi-Request Plugin

Execute multiple API requests in a single call:

```python
from binder.plugins.views.multi_request import MultiRequestView

# Register the view
class MyMultiRequestView(MultiRequestView):
    pass
```

**Usage:**

```bash
POST /api/multi/
Content-Type: application/json

{
  "requests": [
    {
      "method": "GET",
      "url": "/api/article/",
      "id": "articles"
    },
    {
      "method": "POST", 
      "url": "/api/article/",
      "data": {
        "title": "New Article",
        "content": "Content"
      },
      "id": "new_article"
    }
  ]
}

# Response:
{
  "responses": {
    "articles": {
      "status": 200,
      "data": [/* article list */]
    },
    "new_article": {
      "status": 200,
      "data": {/* created article */}
    }
  }
}
```

### Image Processing Plugin

Advanced image manipulation capabilities:

```python
from binder.plugins.views.image import ImageView

class PhotoView(ImageView, ModelView):
    model = Photo
    file_fields = ['image']
```

**Image Operations:**

```bash
# Rotate image
POST /api/photo/1/rotate/
{"angle": 90}

# Crop image
POST /api/photo/1/crop/
{"x": 10, "y": 10, "width": 200, "height": 200}

# Reset image to original
POST /api/photo/1/reset/
```

### My Filters Plugin

User-specific saved filters:

```python
# settings.py
INSTALLED_APPS += ['binder.plugins.my_filters']

# Run migrations
python manage.py migrate
```

**Usage:**

```bash
# Save a filter
POST /api/my_filters/
{
  "name": "Published Articles",
  "model": "article",
  "filters": {"published": true, "category": "tech"}
}

# List saved filters
GET /api/my_filters/

# Apply saved filter
GET /api/article/?my_filter=1
```

### HTML Field Plugin

Secure HTML input validation:

```python
from binder.plugins.models.html_field import HtmlField

class Article(BinderModel):
    title = models.CharField(max_length=200)
    content = HtmlField(
        allowed_tags=['p', 'br', 'strong', 'em', 'ul', 'ol', 'li', 'a'],
        allowed_attributes={'a': ['href', 'title']},
        strip_disallowed=True
    )
```

### Combined View Plugin

Query multiple models in a single request:

```python
from binder.plugins.views.combined import CombinedView

# Register in URLs
urlpatterns += [
    re_path(r'^api/combined/(?P<names>.+)/$', CombinedView.as_view(), {'router': router}),
]
```

**Usage:**

```bash
# Query articles and categories together
GET /api/combined/article/category/

# With filtering
GET /api/combined/article/category/?name__icontains=tech
```

### File Hash View Plugin

File integrity verification:

```python
from binder.plugins.views.file_hash_view import FileHashView

class DocumentView(FileHashView, ModelView):
    model = Document
    file_fields = ['file']
```

**Usage:**

```bash
# Get file hash
GET /api/document/1/file/hash/

# Response:
{
  "hash": "sha1:abc123def456",
  "algorithm": "sha1",
  "filename": "document.pdf",
  "size": 1024000
}
```

## Creating Custom Plugins

### Plugin Structure

Create a custom plugin following Django Binder conventions:

```
my_plugin/
├── __init__.py
├── models.py
├── views.py
├── migrations/
│   └── __init__.py
└── management/
    └── commands/
        └── my_command.py
```

### Custom Plugin Example

Create a notification plugin:

```python
# my_plugin/models.py
from binder.models import BinderModel
from django.contrib.auth.models import User

class Notification(BinderModel):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    message = models.TextField()
    read = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Binder:
        history = True
    
    class Meta:
        ordering = ['-created_at']

# my_plugin/views.py
from binder.views import ModelView
from binder.router import list_route, detail_route
from django.http import JsonResponse
from .models import Notification

class NotificationView(ModelView):
    model = Notification
    
    def get_queryset(self, request):
        # Users can only see their own notifications
        return super().get_queryset(request).filter(user=request.user)
    
    @list_route(name='unread')
    def unread_notifications(self, request):
        """Get unread notifications"""
        queryset = self.get_queryset(request).filter(read=False)
        return self.get_list_response(request, queryset)
    
    @list_route(name='mark_all_read', methods=['POST'])
    def mark_all_read(self, request):
        """Mark all notifications as read"""
        count = self.get_queryset(request).filter(read=False).update(read=True)
        return JsonResponse({
            'success': True,
            'marked_read': count
        })
    
    @detail_route(name='mark_read', methods=['POST'])
    def mark_read(self, request, pk):
        """Mark specific notification as read"""
        notification = self.get_object(pk)
        notification.read = True
        notification.save()
        
        return JsonResponse({
            'success': True,
            'notification_id': notification.id
        })
    
    def store(self, obj, fields, request):
        """Auto-set user on creation"""
        if not obj.pk:  # New object
            obj.user = request.user
        
        return super().store(obj, fields, request)

# my_plugin/utils.py
from .models import Notification

class NotificationService:
    @staticmethod
    def send_notification(user, title, message):
        """Send notification to user"""
        notification = Notification.objects.create(
            user=user,
            title=title,
            message=message
        )
        
        # Send real-time notification via WebSocket
        from binder.websocket import send_to_room
        send_to_room(f'user_{user.id}', {
            'type': 'notification',
            'id': notification.id,
            'title': title,
            'message': message
        })
        
        return notification
    
    @staticmethod
    def send_bulk_notification(users, title, message):
        """Send notification to multiple users"""
        notifications = []
        for user in users:
            notification = NotificationService.send_notification(user, title, message)
            notifications.append(notification)
        
        return notifications
```

### Plugin Configuration

Configure your custom plugin:

```python
# my_plugin/__init__.py
default_app_config = 'my_plugin.apps.MyPluginConfig'

# my_plugin/apps.py
from django.apps import AppConfig

class MyPluginConfig(AppConfig):
    name = 'my_plugin'
    verbose_name = 'My Plugin'
    
    def ready(self):
        # Import signal handlers
        from . import signals

# my_plugin/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User
from .utils import NotificationService

@receiver(post_save, sender=User)
def welcome_notification(sender, instance, created, **kwargs):
    """Send welcome notification to new users"""
    if created:
        NotificationService.send_notification(
            instance,
            "Welcome!",
            "Welcome to our platform. Get started by exploring the features."
        )
```

### Plugin Installation

Install and configure your plugin:

```python
# settings.py
INSTALLED_APPS += ['my_plugin']

# Run migrations
python manage.py makemigrations my_plugin
python manage.py migrate
```

## Plugin Integration Patterns

### Mixin Pattern

Create reusable mixins for common functionality:

```python
# plugins/mixins.py
from django.http import JsonResponse
from django.utils import timezone

class TimestampMixin:
    """Add timestamp information to responses"""
    
    def get_list_response(self, request, queryset):
        response = super().get_list_response(request, queryset)
        
        if hasattr(response, 'data'):
            response.data['meta']['generated_at'] = timezone.now().isoformat()
        
        return response

class AuditMixin:
    """Add audit logging to model operations"""
    
    def store(self, obj, fields, request):
        # Log the operation
        import logging
        logger = logging.getLogger('audit')
        
        action = 'created' if not obj.pk else 'updated'
        logger.info(f'{action} {self.model.__name__} by user {request.user.id}')
        
        return super().store(obj, fields, request)

# Usage
class ArticleView(TimestampMixin, AuditMixin, ModelView):
    model = Article
```

### Decorator Pattern

Create decorators for plugin functionality:

```python
# plugins/decorators.py
from functools import wraps
from django.http import JsonResponse

def rate_limit(max_requests=100, window_seconds=3600):
    """Rate limiting decorator"""
    def decorator(view_func):
        @wraps(view_func)
        def wrapper(self, request, *args, **kwargs):
            # Implement rate limiting logic
            user_key = f"rate_limit:{request.user.id}"
            
            # Check rate limit (implementation depends on cache backend)
            from django.core.cache import cache
            current_requests = cache.get(user_key, 0)
            
            if current_requests >= max_requests:
                return JsonResponse({
                    'error': 'Rate limit exceeded',
                    'retry_after': window_seconds
                }, status=429)
            
            # Increment counter
            cache.set(user_key, current_requests + 1, window_seconds)
            
            return view_func(self, request, *args, **kwargs)
        return wrapper
    return decorator

# Usage
class ArticleView(ModelView):
    model = Article
    
    @rate_limit(max_requests=50, window_seconds=3600)
    @list_route(name='search')
    def search_articles(self, request):
        # Search implementation
        pass
```

### Plugin Registry

Create a plugin registry system:

```python
# plugins/registry.py
class PluginRegistry:
    def __init__(self):
        self._plugins = {}
    
    def register(self, name, plugin_class):
        """Register a plugin"""
        self._plugins[name] = plugin_class
    
    def get_plugin(self, name):
        """Get a registered plugin"""
        return self._plugins.get(name)
    
    def get_all_plugins(self):
        """Get all registered plugins"""
        return self._plugins.copy()
    
    def apply_plugins(self, view_class):
        """Apply all registered plugins to a view class"""
        for plugin_name, plugin_class in self._plugins.items():
            if hasattr(plugin_class, 'apply_to_view'):
                view_class = plugin_class.apply_to_view(view_class)
        
        return view_class

# Global registry instance
plugin_registry = PluginRegistry()

# Plugin base class
class BasePlugin:
    @classmethod
    def apply_to_view(cls, view_class):
        """Apply plugin to view class"""
        # Create new class with plugin functionality
        class PluginView(cls, view_class):
            pass
        
        return PluginView

# Example plugin
class CachingPlugin(BasePlugin):
    cache_timeout = 300
    
    def get_list_response(self, request, queryset):
        from django.core.cache import cache
        
        cache_key = f"list_cache:{self.model.__name__}:{hash(str(queryset.query))}"
        cached_response = cache.get(cache_key)
        
        if cached_response:
            return cached_response
        
        response = super().get_list_response(request, queryset)
        cache.set(cache_key, response, self.cache_timeout)
        
        return response

# Register plugin
plugin_registry.register('caching', CachingPlugin)

# Apply to view
@plugin_registry.apply_plugins
class ArticleView(ModelView):
    model = Article
```

## Plugin Testing

### Plugin Test Framework

Test your custom plugins:

```python
# tests/test_notification_plugin.py
from django.test import TestCase
from django.contrib.auth.models import User
from my_plugin.models import Notification
from my_plugin.utils import NotificationService

class NotificationPluginTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        self.client.force_login(self.user)
    
    def test_send_notification(self):
        """Test sending notification"""
        notification = NotificationService.send_notification(
            self.user,
            "Test Title",
            "Test message"
        )
        
        self.assertEqual(notification.user, self.user)
        self.assertEqual(notification.title, "Test Title")
        self.assertFalse(notification.read)
    
    def test_notification_api(self):
        """Test notification API endpoints"""
        # Create notification
        Notification.objects.create(
            user=self.user,
            title="Test Notification",
            message="Test message"
        )
        
        # Test list endpoint
        response = self.client.get('/api/notification/')
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertEqual(len(data['data']), 1)
    
    def test_mark_read(self):
        """Test marking notification as read"""
        notification = Notification.objects.create(
            user=self.user,
            title="Test Notification",
            message="Test message"
        )
        
        # Mark as read
        response = self.client.post(f'/api/notification/{notification.id}/mark_read/')
        self.assertEqual(response.status_code, 200)
        
        # Verify it's marked as read
        notification.refresh_from_db()
        self.assertTrue(notification.read)
```

### Plugin Integration Tests

Test plugin integration with existing functionality:

```python
class PluginIntegrationTest(TestCase):
    def test_plugin_with_permissions(self):
        """Test plugin works with permission system"""
        # Test that plugin respects user permissions
        pass
    
    def test_plugin_with_history(self):
        """Test plugin works with history tracking"""
        # Test that plugin operations are tracked in history
        pass
    
    def test_plugin_with_websockets(self):
        """Test plugin works with WebSocket notifications"""
        # Test that plugin sends WebSocket messages
        pass
```

This comprehensive plugin system allows you to extend Django Binder with reusable, modular functionality while maintaining clean separation of concerns.
