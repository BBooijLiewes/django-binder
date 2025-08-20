# API Reference

This comprehensive reference covers all Django Binder classes, methods, and configuration options.

## Core Classes

### BinderModel

Base model class for Django Binder applications.

```python
from binder.models import BinderModel

class MyModel(BinderModel):
    # Your fields here
    
    class Binder:
        history = True  # Enable change tracking
```

**Methods:**

- `full_clean()` - Enhanced validation with detailed error reporting
- `save()` - Automatic validation and history tracking
- `serialize_for_history(field_name, value)` - Custom history serialization
- `format_instance_for_history(id)` - Custom history display format

**Configuration Options (Binder class):**

- `history: bool` - Enable change tracking (default: False)
- `history_exclude_fields: list` - Fields to exclude from history
- `soft_delete: bool` - Enable soft delete functionality
- `soft_delete_field: str` - Field name for soft delete timestamp

### ModelView

Base view class for API endpoints.

```python
from binder.views import ModelView

class MyModelView(ModelView):
    model = MyModel
    # Configuration options
```

**Configuration Attributes:**

- `model` - Django model class
- `shown_fields: list` - Fields to include in responses
- `shown_properties: list` - Computed properties to include
- `file_fields: list` - File upload fields
- `m2m_fields: list` - Many-to-many fields
- `searches: list` - Searchable fields
- `alternative_filters: dict` - Named filter groups
- `unupdatable_fields: list` - Fields that cannot be updated

**Methods:**

```python
def get_queryset(self, request):
    """Get base queryset for the view"""
    return self.model.objects.all()

def filter_queryset(self, request, queryset):
    """Apply filters to queryset"""
    return queryset

def serialize_object(self, obj, request):
    """Serialize model instance to dict"""
    return {}

def store(self, obj, fields, request):
    """Save model instance"""
    return obj

def validate_request_data(self, request, data):
    """Validate request data"""
    return data

def get_list_response(self, request, queryset):
    """Generate list response"""
    return JsonResponse({})

def get_object_response(self, request, obj):
    """Generate object response"""
    return JsonResponse({})
```

### Router

URL routing system for automatic endpoint registration.

```python
from binder.router import Router

# Automatic registration
router = Router.bootstrap()

# Manual registration
router = Router()
router.register(ModelView)
```

**Methods:**

- `bootstrap()` - Automatically discover and register views
- `register(view_class)` - Register a view class
- `model_view(model)` - Get view for a model
- `model_route(model, pk=None, field=None)` - Get URL for model

## Field Types

### BinderFileField

Enhanced file field with metadata and validation.

```python
from binder.models import BinderFileField

class Document(BinderModel):
    file = BinderFileField(
        upload_to='documents/',
        allowed_extensions=['pdf', 'doc', 'docx'],
        serve_directly=True,
        max_length=200
    )
```

**Parameters:**

- `upload_to: str` - Upload directory
- `allowed_extensions: list` - Allowed file extensions
- `serve_directly: bool` - Delegate serving to web server
- `max_length: int` - Maximum filename length

**Properties:**

- `hash` - SHA1 hash of file content
- `content_type` - MIME type of file
- `filename` - Original filename

### BinderImageField

Enhanced image field with processing capabilities.

```python
from binder.models import BinderImageField

class Photo(BinderModel):
    image = BinderImageField(
        upload_to='photos/',
        allowed_extensions=['jpg', 'jpeg', 'png'],
        max_length=200
    )
```

**Additional Features:**
- Image validation
- Format conversion support
- Automatic thumbnail generation (when configured)

### ChoiceEnum

Type-safe enum field helper.

```python
from binder.models import ChoiceEnum

class MyModel(BinderModel):
    STATUS = ChoiceEnum('draft', 'published', 'archived')
    
    status = models.CharField(max_length=20, choices=STATUS.choices())
```

**Methods:**

- `choices()` - Get Django choices tuple
- `get_display(value)` - Get display name for value

## Route Decorators

### @list_route

Decorator for custom list endpoints.

```python
from binder.router import list_route

class MyView(ModelView):
    @list_route(name='custom', methods=['GET', 'POST'], unauthenticated=False)
    def custom_endpoint(self, request):
        return JsonResponse({})
```

**Parameters:**

- `name: str` - Route name
- `methods: list` - Allowed HTTP methods
- `unauthenticated: bool` - Allow unauthenticated access
- `extra_route: str` - Additional URL pattern

### @detail_route

Decorator for custom detail endpoints.

```python
from binder.router import detail_route

class MyView(ModelView):
    @detail_route(name='action', methods=['POST'])
    def custom_action(self, request, pk):
        obj = self.get_object(pk)
        return JsonResponse({})
```

**Parameters:**
- Same as `@list_route`
- Method receives `pk` parameter

## Permission System

### PermissionView

Base view with permission checking.

```python
from binder.permissions.views import PermissionView

class MyView(PermissionView):
    model = MyModel
```

**Permission Methods:**

```python
def _require_model_perm(self, perm_type, request, pk=None):
    """Check model permissions"""
    return ['scope_name']

def _scope_view_custom(self, request):
    """Custom view scope"""
    return Q(owner=request.user)

def _scope_change_custom(self, request, obj, values):
    """Custom change scope"""
    return obj.owner == request.user
```

**Permission Types:**
- `view` - Read permissions
- `add` - Create permissions
- `change` - Update permissions
- `delete` - Delete permissions

### Permission Configuration

```python
# permissions.py
permissions = {
    'default': [
        ('auth.view_user', 'own'),
        ('auth.change_user', 'own'),
    ],
    'staff': [
        ('myapp.view_mymodel', 'all'),
        ('myapp.add_mymodel', 'all'),
    ],
}
```

**Built-in Scopes:**
- `all` - Access to all objects
- `own` - Access to owned objects only
- `none` - No access
- `None` - No permission check

## WebSocket Support

### WebSocket Functions

```python
from binder.websocket import send_to_room, register_user_room

# Send message to room
send_to_room('room_name', {
    'type': 'notification',
    'message': 'Hello World'
})

# Register user for room
register_user_room(user, 'room_name')
```

**Functions:**

- `send_to_room(room, message)` - Send message to room
- `register_user_room(user, room)` - Add user to room
- `unregister_user_room(user, room)` - Remove user from room
- `get_user_rooms(user)` - Get user's rooms

## Exception Classes

### BinderException

Base exception class for Django Binder.

```python
from binder.exceptions import BinderException

class CustomException(BinderException):
    http_code = 400
    
    def response_data(self):
        return {'error': 'Custom error'}
```

**Built-in Exceptions:**

- `BinderValidationError` - Validation errors (400)
- `BinderNotFound` - Object not found (404)
- `BinderForbidden` - Permission denied (403)
- `BinderMethodNotAllowed` - Method not allowed (405)
- `BinderRequestError` - General request error (400)

## Utility Functions

### JSON Utilities

```python
from binder.json import JsonResponse, jsonloads, jsondumps

# Enhanced JSON response
response = JsonResponse({'data': 'value'})

# JSON parsing with Django types
data = jsonloads('{"date": "2023-01-01"}')

# JSON serialization with Django types
json_string = jsondumps({'date': datetime.date(2023, 1, 1)})
```

### History Utilities

```python
from binder.history import ChangeSet, Change

# Get changes for object
changes = Change.objects.filter(
    model_name='MyModel',
    object_id=1
).select_related('changeset')

# Get recent changes
recent_changes = ChangeSet.objects.filter(
    timestamp__gte=timezone.now() - timedelta(days=7)
)
```

## Plugin System

### Built-in Plugins

#### Token Authentication

```python
# settings.py
INSTALLED_APPS += ['binder.plugins.token_auth']

# Usage
from binder.plugins.token_auth.models import Token
token = Token.objects.create(user=user)
```

#### User View

```python
from binder.plugins.views.userview import UserView

class MyUserView(UserView):
    # Inherits login, logout, password change endpoints
    pass
```

#### CSV Export

```python
from binder.plugins.views.csvexport import CsvExportView

class MyView(CsvExportView, ModelView):
    model = MyModel
    csv_fields = ['id', 'name', 'created_at']
```

#### Multi-Request

```python
from binder.plugins.views.multi_request import MultiRequestView

# Allows batching multiple API requests
```

#### Image Processing

```python
from binder.plugins.views.image import ImageView

class PhotoView(ImageView, ModelView):
    model = Photo
    # Adds rotate, crop, reset endpoints
```

## Configuration Reference

### Settings

```python
# Required settings
INSTALLED_APPS += ['binder']
CSRF_FAILURE_VIEW = 'binder.router.csrf_failure'

# Optional settings
BINDER_PERMISSION = permissions  # Permission configuration
TOKEN_EXPIRY_HOURS = 24         # Token expiration
TOKEN_CLEANUP_DAYS = 30         # Token cleanup interval
INTERNAL_MEDIA_HEADER = 'X-Accel-Redirect'  # File serving
INTERNAL_MEDIA_LOCATION = '/internal/media/' # File serving path
```

### Database Configuration

```python
# PostgreSQL (recommended)
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

# MySQL (limited support)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'OPTIONS': {
            'init_command': 'SET SESSION group_concat_max_len = 1000000',
        },
    }
}
```

## API Endpoints

### Standard Endpoints

For each registered model, Django Binder automatically creates:

```
GET    /api/model/              # List objects
POST   /api/model/              # Create object
GET    /api/model/{id}/         # Get object
PUT    /api/model/{id}/         # Update object
DELETE /api/model/{id}/         # Delete object
POST   /api/model/{id}/         # Undelete object
```

### File Endpoints

```
GET    /api/model/{id}/field/   # Download file
POST   /api/model/{id}/field/   # Upload file
```

### History Endpoints

```
GET    /api/model/{id}/history/       # View history
GET    /api/model/{id}/history/debug/ # Debug history
```

### Query Parameters

**Filtering:**
- `field=value` - Exact match
- `field__lookup=value` - Django field lookups
- `search=term` - Search across configured fields

**Ordering:**
- `order_by=field` - Ascending order
- `order_by=-field` - Descending order
- `order_by=field1,field2` - Multiple fields

**Pagination:**
- `limit=N` - Limit results
- `offset=N` - Skip results
- `after=id` - Results after specific ID

**Relations:**
- `with=relation` - Include related data
- `with=relation.nested` - Include nested relations

## Error Codes

### HTTP Status Codes

- `200` - Success
- `400` - Bad Request / Validation Error
- `401` - Unauthorized
- `403` - Forbidden
- `404` - Not Found
- `405` - Method Not Allowed
- `429` - Too Many Requests
- `500` - Internal Server Error

### Error Response Format

```json
{
  "code": "ValidationError",
  "error": {
    "validation_errors": {
      "field_name": [
        {
          "code": "Error message"
        }
      ]
    }
  }
}
```

## Management Commands

### Built-in Commands

```bash
# Token management
python manage.py create_user_token username
python manage.py delete_user_token username
python manage.py cleanup_expired_tokens

# History management
python manage.py binder_history_remove_reverse_spam

# Permission management
python manage.py define_groups
```

This API reference provides comprehensive documentation for all Django Binder features and functionality.
