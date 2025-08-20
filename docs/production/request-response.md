# Request/Response Handling

Django Binder provides comprehensive request and response handling with automatic serialization, validation, and error management for building robust APIs.

## Request Processing

### Request Data Parsing

Django Binder automatically parses different request formats:

```python
class ArticleView(ModelView):
    model = Article
    
    def _get_request_data(self, request):
        """Get parsed request data"""
        # Automatically handles:
        # - JSON requests (application/json)
        # - Form data (application/x-www-form-urlencoded)
        # - Multipart form data (multipart/form-data)
        # - File uploads with data
        
        data = super()._get_request_data(request)
        
        # Custom processing
        if 'title' in data:
            data['title'] = data['title'].strip()
        
        return data
```

### JSON Request Handling

Handle JSON requests with automatic parsing:

```bash
POST /api/article/
Content-Type: application/json

{
  "title": "New Article",
  "content": "Article content here",
  "published": true,
  "tags": [1, 2, 3],
  "metadata": {
    "source": "api",
    "priority": "high"
  }
}
```

### Multipart Form Data

Handle file uploads with form data:

```bash
POST /api/article/
Content-Type: multipart/form-data

data: {"title": "Article with Image", "content": "Content", "featured_image": null}
file:featured_image: [binary image data]
```

### Custom Request Processing

Implement custom request processing logic:

```python
class ArticleView(ModelView):
    model = Article
    
    def _get_request_data(self, request):
        """Custom request data processing"""
        data = super()._get_request_data(request)
        
        # Auto-generate slug from title
        if 'title' in data and 'slug' not in data:
            from django.utils.text import slugify
            data['slug'] = slugify(data['title'])
        
        # Set author to current user if not specified
        if 'author' not in data and request.user.is_authenticated:
            data['author'] = request.user.id
        
        # Process tags (convert names to IDs)
        if 'tag_names' in data:
            tag_names = data.pop('tag_names')
            tag_ids = []
            for name in tag_names:
                tag, created = Tag.objects.get_or_create(name=name)
                tag_ids.append(tag.id)
            data['tags'] = tag_ids
        
        # Validate business rules
        if data.get('published') and not data.get('content'):
            raise ValidationError('Published articles must have content')
        
        return data
```

## Response Serialization

### Automatic Serialization

Django Binder automatically serializes model instances to JSON:

```python
# GET /api/article/1/ returns:
{
  "id": 1,
  "title": "Sample Article",
  "content": "Article content",
  "published": true,
  "created_at": "2023-12-01T10:30:00Z",
  "updated_at": "2023-12-01T15:45:00Z",
  "author": 5,
  "category": 2,
  "tags": [1, 3, 5]
}
```

### Custom Serialization

Customize how objects are serialized:

```python
class ArticleView(ModelView):
    model = Article
    shown_fields = ['id', 'title', 'content', 'published', 'created_at']
    shown_properties = ['word_count', 'reading_time', 'author_name']
    
    def get_word_count(self, obj):
        """Calculate word count"""
        return len(obj.content.split())
    
    def get_reading_time(self, obj):
        """Estimate reading time"""
        word_count = self.get_word_count(obj)
        return max(1, word_count // 200)  # 200 words per minute
    
    def get_author_name(self, obj):
        """Get author's display name"""
        return obj.author.get_full_name() or obj.author.username
    
    def serialize_object(self, obj, request):
        """Custom object serialization"""
        data = super().serialize_object(obj, request)
        
        # Add computed fields
        data['is_recent'] = (timezone.now() - obj.created_at).days < 7
        data['can_edit'] = obj.author == request.user or request.user.is_staff
        
        # Conditional fields based on permissions
        if not request.user.is_staff:
            data.pop('internal_notes', None)
        
        return data
```

### Related Object Serialization

Control how related objects are serialized:

```python
class ArticleView(ModelView):
    model = Article
    
    def serialize_object(self, obj, request):
        """Serialize with related objects"""
        data = super().serialize_object(obj, request)
        
        # Include author details
        data['author'] = {
            'id': obj.author.id,
            'username': obj.author.username,
            'full_name': obj.author.get_full_name(),
            'avatar_url': obj.author.profile.avatar.url if obj.author.profile.avatar else None
        }
        
        # Include category details
        if obj.category:
            data['category'] = {
                'id': obj.category.id,
                'name': obj.category.name,
                'slug': obj.category.slug
            }
        
        # Include tag details
        data['tags'] = [
            {
                'id': tag.id,
                'name': tag.name,
                'color': tag.color
            }
            for tag in obj.tags.all()
        ]
        
        return data
```

## Response Formats

### Standard Response Format

Django Binder uses consistent response formats:

```json
// Single object response
{
  "id": 1,
  "title": "Article Title",
  "content": "Article content",
  // ... other fields
}

// List response
{
  "data": [
    {
      "id": 1,
      "title": "First Article"
    },
    {
      "id": 2,
      "title": "Second Article"
    }
  ],
  "meta": {
    "total_records": 25,
    "limit": 10,
    "offset": 0
  }
}
```

### Custom Response Formatting

Customize response format:

```python
class ArticleView(ModelView):
    model = Article
    
    def get_list_response(self, request, queryset):
        """Custom list response format"""
        # Get standard response
        response = super().get_list_response(request, queryset)
        
        # Add custom metadata
        if hasattr(response, 'data') and isinstance(response.data, dict):
            response.data['meta'].update({
                'published_count': queryset.filter(published=True).count(),
                'draft_count': queryset.filter(published=False).count(),
                'generated_at': timezone.now().isoformat(),
                'api_version': '1.0'
            })
        
        return response
    
    def get_object_response(self, request, obj):
        """Custom object response format"""
        response = super().get_object_response(request, obj)
        
        # Add related data
        if hasattr(response, 'data'):
            response.data['related'] = {
                'comments_count': obj.comments.count(),
                'views_count': obj.view_count,
                'related_articles': [
                    {'id': a.id, 'title': a.title}
                    for a in obj.get_related_articles()[:3]
                ]
            }
        
        return response
```

### Pagination Response

Handle paginated responses:

```python
class ArticleView(ModelView):
    model = Article
    
    def get_list_response(self, request, queryset):
        """Paginated list response"""
        # Apply pagination
        limit = int(request.GET.get('limit', 20))
        offset = int(request.GET.get('offset', 0))
        
        total_count = queryset.count()
        paginated_queryset = queryset[offset:offset + limit]
        
        # Serialize objects
        data = [self.serialize_object(obj, request) for obj in paginated_queryset]
        
        # Build response
        response_data = {
            'data': data,
            'meta': {
                'total_records': total_count,
                'limit': limit,
                'offset': offset,
                'has_next': offset + limit < total_count,
                'has_previous': offset > 0,
                'next_offset': offset + limit if offset + limit < total_count else None,
                'previous_offset': max(0, offset - limit) if offset > 0 else None
            }
        }
        
        return JsonResponse(response_data)
```

## Error Handling

### Validation Errors

Handle validation errors with detailed responses:

```python
class ArticleView(ModelView):
    model = Article
    
    def validate_request_data(self, request, data):
        """Custom validation with detailed errors"""
        errors = {}
        
        # Title validation
        if 'title' in data:
            title = data['title'].strip()
            if not title:
                errors['title'] = ['Title cannot be empty']
            elif len(title) > 200:
                errors['title'] = ['Title cannot exceed 200 characters']
            elif Article.objects.filter(title=title).exclude(id=data.get('id')).exists():
                errors['title'] = ['Article with this title already exists']
        
        # Content validation
        if 'content' in data:
            content = data['content'].strip()
            if data.get('published') and len(content) < 100:
                errors['content'] = ['Published articles must have at least 100 characters']
        
        # Category validation
        if 'category' in data:
            try:
                category = Category.objects.get(id=data['category'])
                if not category.active:
                    errors['category'] = ['Selected category is not active']
            except Category.DoesNotExist:
                errors['category'] = ['Invalid category selected']
        
        # Tags validation
        if 'tags' in data:
            tag_ids = data['tags']
            if len(tag_ids) > 10:
                errors['tags'] = ['Maximum 10 tags allowed']
            
            existing_tags = Tag.objects.filter(id__in=tag_ids).count()
            if existing_tags != len(tag_ids):
                errors['tags'] = ['One or more invalid tags selected']
        
        if errors:
            raise ValidationError(errors)
        
        return super().validate_request_data(request, data)
```

### Error Response Format

Standardized error responses:

```json
// Validation error response
{
  "code": "ValidationError",
  "error": {
    "validation_errors": {
      "title": [
        {
          "code": "This field cannot be blank."
        }
      ],
      "email": [
        {
          "code": "Enter a valid email address."
        }
      ]
    }
  }
}

// Permission error response
{
  "code": "PermissionDenied",
  "error": {
    "message": "You do not have permission to perform this action.",
    "required_permission": "article.change_article"
  }
}

// Not found error response
{
  "code": "NotFound",
  "error": {
    "message": "Article with id 999 not found."
  }
}
```

### Custom Error Handling

Implement custom error handling:

```python
from binder.exceptions import BinderException

class CustomValidationError(BinderException):
    http_code = 400
    
    def __init__(self, field_errors, message="Validation failed"):
        self.field_errors = field_errors
        super().__init__(message)
    
    def response_data(self):
        return {
            'code': 'CustomValidationError',
            'message': str(self),
            'field_errors': self.field_errors,
            'timestamp': timezone.now().isoformat()
        }

class ArticleView(ModelView):
    model = Article
    
    def validate_business_rules(self, obj, data, request):
        """Custom business rule validation"""
        errors = {}
        
        # Check publication rules
        if data.get('published'):
            if not obj.content or len(obj.content.strip()) < 500:
                errors['content'] = 'Published articles must have at least 500 characters'
            
            if not obj.featured_image:
                errors['featured_image'] = 'Published articles must have a featured image'
            
            if obj.category and obj.category.requires_approval and not request.user.is_staff:
                errors['category'] = 'This category requires staff approval'
        
        # Check author permissions
        if 'author' in data and data['author'] != request.user.id:
            if not request.user.has_perm('article.change_author'):
                errors['author'] = 'You cannot change the article author'
        
        if errors:
            raise CustomValidationError(errors)
    
    def store(self, obj, fields, request):
        """Override store with custom validation"""
        # Validate business rules before saving
        self.validate_business_rules(obj, fields, request)
        
        return super().store(obj, fields, request)
```

## Content Negotiation

### Accept Header Handling

Handle different response formats based on Accept header:

```python
class ArticleView(ModelView):
    model = Article
    
    def get_response_format(self, request):
        """Determine response format from Accept header"""
        accept = request.META.get('HTTP_ACCEPT', 'application/json')
        
        if 'application/xml' in accept:
            return 'xml'
        elif 'text/csv' in accept:
            return 'csv'
        elif 'text/html' in accept:
            return 'html'
        else:
            return 'json'
    
    def get_list_response(self, request, queryset):
        """Return response in requested format"""
        format_type = self.get_response_format(request)
        
        if format_type == 'csv':
            return self.get_csv_response(queryset)
        elif format_type == 'xml':
            return self.get_xml_response(queryset)
        else:
            return super().get_list_response(request, queryset)
    
    def get_csv_response(self, queryset):
        """Return CSV response"""
        import csv
        from django.http import HttpResponse
        
        response = HttpResponse(content_type='text/csv')
        response['Content-Disposition'] = 'attachment; filename="articles.csv"'
        
        writer = csv.writer(response)
        writer.writerow(['ID', 'Title', 'Author', 'Published', 'Created'])
        
        for article in queryset:
            writer.writerow([
                article.id,
                article.title,
                article.author.username,
                article.published,
                article.created_at.strftime('%Y-%m-%d')
            ])
        
        return response
```

### Custom Content Types

Support custom content types:

```python
class ArticleView(ModelView):
    model = Article
    
    def dispatch(self, request, *args, **kwargs):
        """Handle custom content types"""
        content_type = request.META.get('CONTENT_TYPE', '')
        
        if content_type.startswith('application/vnd.api+json'):
            # JSON API format
            request._custom_format = 'jsonapi'
        elif content_type.startswith('application/vnd.article+json'):
            # Custom article format
            request._custom_format = 'article'
        
        return super().dispatch(request, *args, **kwargs)
    
    def serialize_object(self, obj, request):
        """Serialize based on custom format"""
        if hasattr(request, '_custom_format'):
            if request._custom_format == 'jsonapi':
                return self.serialize_jsonapi(obj, request)
            elif request._custom_format == 'article':
                return self.serialize_article_format(obj, request)
        
        return super().serialize_object(obj, request)
    
    def serialize_jsonapi(self, obj, request):
        """JSON API format serialization"""
        return {
            'type': 'article',
            'id': str(obj.id),
            'attributes': {
                'title': obj.title,
                'content': obj.content,
                'published': obj.published,
                'created-at': obj.created_at.isoformat(),
            },
            'relationships': {
                'author': {
                    'data': {'type': 'user', 'id': str(obj.author.id)}
                },
                'category': {
                    'data': {'type': 'category', 'id': str(obj.category.id)} if obj.category else None
                }
            }
        }
```

## Response Caching

### HTTP Caching Headers

Set appropriate caching headers:

```python
from django.views.decorators.cache import cache_control
from django.utils.decorators import method_decorator

class ArticleView(ModelView):
    model = Article
    
    @method_decorator(cache_control(max_age=300))  # 5 minutes
    def get_list_response(self, request, queryset):
        """Cached list response"""
        response = super().get_list_response(request, queryset)
        
        # Set ETag for cache validation
        import hashlib
        content_hash = hashlib.md5(str(queryset.query).encode()).hexdigest()
        response['ETag'] = f'"{content_hash}"'
        
        return response
    
    def get_object_response(self, request, obj):
        """Cached object response"""
        response = super().get_object_response(request, obj)
        
        # Set Last-Modified header
        response['Last-Modified'] = obj.updated_at.strftime('%a, %d %b %Y %H:%M:%S GMT')
        
        # Set ETag based on object version
        response['ETag'] = f'"{obj.id}-{obj.updated_at.timestamp()}"'
        
        return response
```

### Conditional Requests

Handle conditional requests:

```python
class ArticleView(ModelView):
    model = Article
    
    def get_object_response(self, request, obj):
        """Handle conditional requests"""
        # Check If-Modified-Since header
        if_modified_since = request.META.get('HTTP_IF_MODIFIED_SINCE')
        if if_modified_since:
            from django.utils.http import parse_http_date
            try:
                modified_since = parse_http_date(if_modified_since)
                if obj.updated_at.timestamp() <= modified_since:
                    return HttpResponse(status=304)  # Not Modified
            except ValueError:
                pass
        
        # Check If-None-Match header (ETag)
        if_none_match = request.META.get('HTTP_IF_NONE_MATCH')
        if if_none_match:
            current_etag = f'"{obj.id}-{obj.updated_at.timestamp()}"'
            if if_none_match == current_etag:
                return HttpResponse(status=304)  # Not Modified
        
        return super().get_object_response(request, obj)
```

## Testing Request/Response

### Request/Response Tests

Test request and response handling:

```python
from django.test import TestCase
from django.contrib.auth.models import User
import json

class RequestResponseTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        self.client.force_login(self.user)
    
    def test_json_request(self):
        """Test JSON request handling"""
        data = {
            'title': 'Test Article',
            'content': 'Test content',
            'published': True
        }
        
        response = self.client.post(
            '/api/article/',
            json.dumps(data),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, 200)
        response_data = response.json()
        self.assertEqual(response_data['title'], 'Test Article')
    
    def test_validation_error_response(self):
        """Test validation error response format"""
        data = {
            'title': '',  # Empty title should fail
            'content': 'Test content'
        }
        
        response = self.client.post(
            '/api/article/',
            json.dumps(data),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, 400)
        response_data = response.json()
        self.assertEqual(response_data['code'], 'ValidationError')
        self.assertIn('validation_errors', response_data['error'])
    
    def test_custom_serialization(self):
        """Test custom field serialization"""
        article = Article.objects.create(
            title='Test Article',
            content='Test content with many words here',
            author=self.user
        )
        
        response = self.client.get(f'/api/article/{article.id}/')
        self.assertEqual(response.status_code, 200)
        
        response_data = response.json()
        self.assertIn('word_count', response_data)
        self.assertGreater(response_data['word_count'], 0)
```

This comprehensive request/response system provides robust data handling with extensive customization options.
