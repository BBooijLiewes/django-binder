# Views & ViewSets

Django Binder's view system is the heart of API development, providing powerful, convention-based classes that automatically generate comprehensive REST APIs from your models.

## ModelView Basics

The `ModelView` class is the foundation of Django Binder's API system:

```python
from binder.views import ModelView
from .models import Article

class ArticleView(ModelView):
    model = Article
    # That's it! Full CRUD API is now available
```

This minimal configuration automatically provides:
- `GET /api/article/` - List all articles
- `POST /api/article/` - Create new article
- `GET /api/article/{id}/` - Get specific article
- `PUT /api/article/{id}/` - Update specific article
- `DELETE /api/article/{id}/` - Delete specific article
- `POST /api/article/{id}/` - Undelete specific article

## View Configuration

### Basic Configuration Options

```python
class ArticleView(ModelView):
    model = Article
    
    # Control which fields are included in responses
    shown_fields = ['id', 'title', 'content', 'published', 'created_at']
    
    # Include computed properties in responses
    shown_properties = ['word_count', 'reading_time']
    
    # Configure file upload fields
    file_fields = ['featured_image', 'attachment']
    
    # Configure many-to-many fields
    m2m_fields = ['tags', 'categories']
    
    # Enable searching
    searches = ['title__icontains', 'content__icontains']
    
    # Configure advanced filtering
    alternative_filters = {
        'author_search': ['author__name__icontains', 'author__email__icontains'],
        'date_range': ['created_at__gte', 'created_at__lte'],
        'status_filter': ['published', 'featured'],
    }
```

### Computed Properties

Add computed properties to your API responses:

```python
class ArticleView(ModelView):
    model = Article
    shown_properties = ['word_count', 'reading_time', 'author_name']
    
    def get_word_count(self, obj):
        """Calculate word count"""
        return len(obj.content.split())
    
    def get_reading_time(self, obj):
        """Estimate reading time in minutes"""
        word_count = self.get_word_count(obj)
        return max(1, word_count // 200)  # Assume 200 words per minute
    
    def get_author_name(self, obj):
        """Get author's full name"""
        return obj.author.get_full_name() or obj.author.username
```

## Query Customization

### Custom QuerySets

Override `get_queryset` to customize the base query:

```python
class ArticleView(ModelView):
    model = Article
    
    def get_queryset(self, request):
        """Custom queryset with optimizations"""
        queryset = super().get_queryset(request)
        
        # Add select_related for performance
        queryset = queryset.select_related('author', 'category')
        
        # Add prefetch_related for many-to-many fields
        queryset = queryset.prefetch_related('tags')
        
        # Filter based on user permissions
        if not request.user.is_staff:
            queryset = queryset.filter(published=True)
        
        return queryset
```

### Annotations and Aggregations

Add database-level calculations:

```python
from django.db.models import Count, Avg, Q

class ArticleView(ModelView):
    model = Article
    shown_properties = ['comment_count', 'average_rating']
    
    def get_queryset(self, request):
        return super().get_queryset(request).annotate(
            comment_count=Count('comments'),
            average_rating=Avg('ratings__score'),
            published_comment_count=Count(
                'comments', 
                filter=Q(comments__approved=True)
            )
        )
    
    def get_comment_count(self, obj):
        return obj.comment_count
    
    def get_average_rating(self, obj):
        return round(obj.average_rating or 0, 1)
```

## Filtering and Searching

### Basic Searching

Enable simple text search across multiple fields:

```python
class ArticleView(ModelView):
    model = Article
    searches = [
        'title__icontains',
        'content__icontains', 
        'author__name__icontains',
        'tags__name__icontains'
    ]
```

Usage: `GET /api/article/?search=django`

### Alternative Filters

Create named filter groups for complex filtering:

```python
class ArticleView(ModelView):
    model = Article
    alternative_filters = {
        # Content-based filtering
        'content_search': [
            'title__icontains',
            'content__icontains',
            'excerpt__icontains'
        ],
        
        # Author-based filtering
        'author_filter': [
            'author__username',
            'author__email',
            'author__first_name__icontains',
            'author__last_name__icontains'
        ],
        
        # Date-based filtering
        'date_range': [
            'created_at__gte',
            'created_at__lte',
            'published_at__gte',
            'published_at__lte'
        ],
        
        # Status filtering
        'status': ['published', 'featured', 'draft'],
        
        # Category filtering with relationships
        'category_filter': [
            'category__name',
            'category__slug',
            'category__parent__name'
        ]
    }
```

Usage examples:
```bash
# Search in content
GET /api/article/?content_search=django

# Filter by author
GET /api/article/?author_filter=john

# Date range filtering
GET /api/article/?date_range.created_at__gte=2023-01-01&date_range.created_at__lte=2023-12-31

# Multiple filters
GET /api/article/?status=published&category_filter=technology&content_search=python
```

### Advanced Filter Operators

Use `:any` and `:all` operators for complex logic:

```python
# Requires ALL specified categories
GET /api/article/?category_filter:all=tech,python

# Requires ANY of the specified authors  
GET /api/article/?author_filter:any=john,jane

# Negation with :not
GET /api/article/?status:not=draft

# Combination of operators
GET /api/article/?category_filter:not:any=archived,deleted
```

### Custom Filtering Logic

Implement custom filtering methods:

```python
class ArticleView(ModelView):
    model = Article
    
    def filter_queryset(self, request, queryset):
        """Custom filtering logic"""
        queryset = super().filter_queryset(request, queryset)
        
        # Custom filter: articles from last N days
        days = request.GET.get('last_days')
        if days:
            try:
                days = int(days)
                cutoff_date = timezone.now() - timedelta(days=days)
                queryset = queryset.filter(created_at__gte=cutoff_date)
            except ValueError:
                pass
        
        # Custom filter: articles by reading difficulty
        difficulty = request.GET.get('difficulty')
        if difficulty == 'easy':
            queryset = queryset.extra(
                where=["LENGTH(content) < 1000"]
            )
        elif difficulty == 'hard':
            queryset = queryset.extra(
                where=["LENGTH(content) > 5000"]
            )
        
        return queryset
```

## Custom Endpoints

### List Routes

Add custom endpoints that operate on collections:

```python
from binder.router import list_route
from django.http import JsonResponse

class ArticleView(ModelView):
    model = Article
    
    @list_route(name='published')
    def published_articles(self, request):
        """Get only published articles"""
        queryset = self.get_queryset(request).filter(published=True)
        return self.get_list_response(request, queryset)
    
    @list_route(name='stats')
    def article_stats(self, request):
        """Get article statistics"""
        queryset = self.get_queryset(request)
        stats = {
            'total': queryset.count(),
            'published': queryset.filter(published=True).count(),
            'draft': queryset.filter(published=False).count(),
            'this_month': queryset.filter(
                created_at__gte=timezone.now().replace(day=1)
            ).count()
        }
        return JsonResponse(stats)
    
    @list_route(name='export', methods=['POST'])
    def export_articles(self, request):
        """Export articles to CSV"""
        import csv
        from django.http import HttpResponse
        
        response = HttpResponse(content_type='text/csv')
        response['Content-Disposition'] = 'attachment; filename="articles.csv"'
        
        writer = csv.writer(response)
        writer.writerow(['ID', 'Title', 'Author', 'Published', 'Created'])
        
        queryset = self.get_queryset(request)
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

### Detail Routes

Add custom endpoints that operate on specific instances:

```python
from binder.router import detail_route

class ArticleView(ModelView):
    model = Article
    
    @detail_route(name='publish', methods=['POST'])
    def publish_article(self, request, pk):
        """Publish an article"""
        article = self.get_object(pk)
        article.published = True
        article.published_at = timezone.now()
        article.save()
        
        return JsonResponse({
            'success': True,
            'message': 'Article published successfully',
            'published_at': article.published_at.isoformat()
        })
    
    @detail_route(name='duplicate', methods=['POST'])
    def duplicate_article(self, request, pk):
        """Create a duplicate of an article"""
        original = self.get_object(pk)
        
        # Create duplicate
        duplicate = Article.objects.create(
            title=f"Copy of {original.title}",
            content=original.content,
            author=original.author,
            category=original.category,
            published=False
        )
        
        # Copy many-to-many relationships
        duplicate.tags.set(original.tags.all())
        
        return JsonResponse({
            'success': True,
            'duplicate_id': duplicate.id,
            'message': 'Article duplicated successfully'
        })
    
    @detail_route(name='related')
    def get_related_articles(self, request, pk):
        """Get articles related to this one"""
        article = self.get_object(pk)
        
        # Find related articles by shared tags
        related = Article.objects.filter(
            tags__in=article.tags.all(),
            published=True
        ).exclude(
            id=article.id
        ).distinct()[:5]
        
        return self.get_list_response(request, related)
```

### Route Configuration

Customize route behavior with parameters:

```python
class ArticleView(ModelView):
    model = Article
    
    @list_route(
        name='search',
        methods=['GET', 'POST'],
        unauthenticated=True,  # Allow unauthenticated access
        extra_route=r'(?P<query>[^/]+)/'  # Add URL parameter
    )
    def search_articles(self, request, query=None):
        """Advanced search endpoint"""
        if request.method == 'POST':
            # Handle complex search from POST body
            search_data = self._get_request_data(request)
            query = search_data.get('query', '')
        
        queryset = self.get_queryset(request).filter(
            Q(title__icontains=query) | Q(content__icontains=query)
        )
        
        return self.get_list_response(request, queryset)
```

## Request and Response Handling

### Custom Request Processing

Override request handling methods:

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
        
        return data
    
    def store(self, obj, fields, request):
        """Custom save logic"""
        # Pre-save processing
        if 'published' in fields and fields['published']:
            if not obj.published_at:
                obj.published_at = timezone.now()
        
        # Call parent save method
        result = super().store(obj, fields, request)
        
        # Post-save processing
        if 'published' in fields and fields['published']:
            # Send notification, update search index, etc.
            self.notify_publication(obj)
        
        return result
    
    def notify_publication(self, article):
        """Send notifications when article is published"""
        # Implementation depends on your notification system
        pass
```

### Custom Response Formatting

Customize response format:

```python
class ArticleView(ModelView):
    model = Article
    
    def get_list_response(self, request, queryset):
        """Custom list response format"""
        response = super().get_list_response(request, queryset)
        
        # Add metadata to response
        if hasattr(response, 'data'):
            response.data['meta'] = {
                'total_count': queryset.count(),
                'published_count': queryset.filter(published=True).count(),
                'generated_at': timezone.now().isoformat()
            }
        
        return response
    
    def get_object_response(self, request, obj):
        """Custom object response format"""
        response = super().get_object_response(request, obj)
        
        # Add related data
        if hasattr(response, 'data'):
            response.data['related_articles'] = [
                {'id': a.id, 'title': a.title}
                for a in obj.get_related_articles()[:3]
            ]
        
        return response
```

## File Handling

### File Upload Configuration

Configure file upload handling:

```python
class ArticleView(ModelView):
    model = Article
    file_fields = ['featured_image', 'attachment', 'gallery_images']
    
    # Configure image resizing
    image_resize_threshold = {
        'featured_image': 1200,  # Resize if larger than 1200px
        'gallery_images': 800,
    }
    
    # Configure image format conversion
    image_format_override = {
        'featured_image': 'jpeg',  # Convert to JPEG
    }
    
    def validate_file_upload(self, field_name, file_obj):
        """Custom file validation"""
        if field_name == 'featured_image':
            # Validate image dimensions
            from PIL import Image
            img = Image.open(file_obj)
            if img.width < 400 or img.height < 300:
                raise ValidationError('Featured image must be at least 400x300 pixels')
        
        elif field_name == 'attachment':
            # Validate file type
            allowed_types = ['application/pdf', 'application/msword']
            if file_obj.content_type not in allowed_types:
                raise ValidationError('Only PDF and Word documents are allowed')
        
        return super().validate_file_upload(field_name, file_obj)
```

### Custom File Processing

```python
class ArticleView(ModelView):
    model = Article
    file_fields = ['featured_image']
    
    def handle_file_upload(self, obj, field_name, file_obj):
        """Custom file processing"""
        if field_name == 'featured_image':
            # Generate thumbnail
            self.generate_thumbnail(obj, file_obj)
            
            # Extract image metadata
            self.extract_image_metadata(obj, file_obj)
        
        return super().handle_file_upload(obj, field_name, file_obj)
    
    def generate_thumbnail(self, obj, image_file):
        """Generate thumbnail for uploaded image"""
        from PIL import Image
        import io
        
        # Open and resize image
        img = Image.open(image_file)
        img.thumbnail((300, 300), Image.Resampling.LANCZOS)
        
        # Save thumbnail
        thumb_io = io.BytesIO()
        img.save(thumb_io, format='JPEG', quality=85)
        thumb_io.seek(0)
        
        # Save to model (assuming you have a thumbnail field)
        if hasattr(obj, 'thumbnail'):
            obj.thumbnail.save(
                f'thumb_{obj.id}.jpg',
                ContentFile(thumb_io.read()),
                save=False
            )
```

## Permissions and Security

### Permission Integration

Integrate with Django Binder's permission system:

```python
from binder.permissions.views import PermissionView

class ArticleView(PermissionView):
    model = Article
    
    def _require_model_perm(self, perm_type, request, pk=None):
        """Custom permission checking"""
        # Call parent method for basic permission check
        scopes = super()._require_model_perm(perm_type, request, pk)
        
        # Add custom logic
        if perm_type == 'view' and not request.user.is_staff:
            # Non-staff users can only see published articles
            return ['published_only']
        
        return scopes
    
    def _scope_view_published_only(self, request):
        """Custom view scope for published articles only"""
        return Q(published=True)
    
    def _scope_change_own(self, request, obj, values):
        """Users can only edit their own articles"""
        return obj.author == request.user
```

### Input Validation and Sanitization

```python
from django.core.exceptions import ValidationError
import bleach

class ArticleView(ModelView):
    model = Article
    
    def validate_request_data(self, request, data):
        """Custom request validation"""
        # Sanitize HTML content
        if 'content' in data:
            allowed_tags = ['p', 'br', 'strong', 'em', 'ul', 'ol', 'li', 'a']
            allowed_attributes = {'a': ['href', 'title']}
            data['content'] = bleach.clean(
                data['content'],
                tags=allowed_tags,
                attributes=allowed_attributes,
                strip=True
            )
        
        # Validate title length
        if 'title' in data and len(data['title']) > 200:
            raise ValidationError({'title': 'Title cannot exceed 200 characters'})
        
        # Custom business logic validation
        if 'published' in data and data['published']:
            if 'content' not in data or len(data['content']) < 100:
                raise ValidationError({
                    'content': 'Published articles must have at least 100 characters'
                })
        
        return super().validate_request_data(request, data)
```

## Performance Optimization

### Query Optimization

```python
class ArticleView(ModelView):
    model = Article
    
    def get_queryset(self, request):
        """Optimized queryset"""
        queryset = super().get_queryset(request)
        
        # Always include related data to avoid N+1 queries
        queryset = queryset.select_related(
            'author',
            'category',
            'author__profile'
        ).prefetch_related(
            'tags',
            'comments__author'
        )
        
        # Add useful annotations
        queryset = queryset.annotate(
            comment_count=Count('comments'),
            tag_count=Count('tags')
        )
        
        return queryset
```

### Caching

```python
from django.core.cache import cache
from django.utils.cache import make_template_fragment_key

class ArticleView(ModelView):
    model = Article
    cache_timeout = 300  # 5 minutes
    
    def get_list_response(self, request, queryset):
        """Cached list response"""
        # Create cache key based on query parameters
        cache_key = f"article_list:{hash(str(queryset.query))}"
        
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
        cache.delete_many([
            f"article_list:*",
            f"article_detail:{obj.id}",
        ])
        
        return result
```

## Testing Views

### View Testing

```python
from django.test import TestCase
from django.contrib.auth.models import User
from rest_framework.test import APIClient
from .models import Article, Author

class ArticleViewTest(TestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        self.author = Author.objects.create(user=self.user, name='Test Author')
        
    def test_list_articles(self):
        """Test article list endpoint"""
        # Create test data
        Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.author,
            published=True
        )
        
        response = self.client.get('/api/article/')
        self.assertEqual(response.status_code, 200)
        self.assertEqual(len(response.json()['data']), 1)
    
    def test_create_article(self):
        """Test article creation"""
        self.client.force_authenticate(user=self.user)
        
        data = {
            'title': 'New Article',
            'content': 'New content',
            'author': self.author.id,
            'published': False
        }
        
        response = self.client.post('/api/article/', data)
        self.assertEqual(response.status_code, 200)
        self.assertTrue(Article.objects.filter(title='New Article').exists())
    
    def test_custom_endpoint(self):
        """Test custom endpoint"""
        article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.author,
            published=False
        )
        
        response = self.client.post(f'/api/article/{article.id}/publish/')
        self.assertEqual(response.status_code, 200)
        
        article.refresh_from_db()
        self.assertTrue(article.published)
```

This comprehensive guide covers all aspects of Django Binder's view system, from basic configuration to advanced customization patterns.
