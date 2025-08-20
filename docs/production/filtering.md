# Filtering & Searching

Django Binder provides powerful filtering and searching capabilities that allow clients to query data with complex conditions while maintaining performance and security.

## Basic Filtering

### Field-Based Filtering

Filter on any model field using URL parameters:

```python
# Simple equality filtering
GET /api/article/?published=true
GET /api/article/?author=5
GET /api/article/?category=technology

# Multiple filters (AND logic)
GET /api/article/?published=true&category=technology&author=5
```

### Django Field Lookups

Use Django's field lookup syntax for advanced filtering:

```python
# Text field lookups
GET /api/article/?title__icontains=django
GET /api/article/?title__startswith=How
GET /api/article/?title__endswith=Guide
GET /api/article/?title__exact=Django Binder Guide

# Numeric field lookups
GET /api/article/?view_count__gt=1000
GET /api/article/?view_count__gte=100
GET /api/article/?view_count__lt=50
GET /api/article/?rating__range=3,5

# Date field lookups
GET /api/article/?created_at__date=2023-12-01
GET /api/article/?created_at__year=2023
GET /api/article/?created_at__month=12
GET /api/article/?published_at__gte=2023-01-01
GET /api/article/?published_at__lte=2023-12-31

# Null filtering
GET /api/article/?featured_image__isnull=false
GET /api/article/?deleted_at__isnull=true
```

### Relationship Filtering

Filter across foreign key relationships:

```python
# Filter by related model fields
GET /api/article/?author__username=john
GET /api/article/?author__email__endswith=@company.com
GET /api/article/?category__name=Technology
GET /api/article/?category__parent__name=Programming

# Many-to-many filtering
GET /api/article/?tags__name=python
GET /api/article/?tags__name__in=python,django,web
```

## Search Functionality

### Basic Search Configuration

Enable search across multiple fields:

```python
class ArticleView(ModelView):
    model = Article
    searches = [
        'title__icontains',
        'content__icontains',
        'author__username__icontains',
        'tags__name__icontains'
    ]
```

Usage:
```bash
# Search across all configured fields
GET /api/article/?search=django

# Searches in title, content, author username, and tag names
```

### Advanced Search Configuration

Configure different search strategies:

```python
class ArticleView(ModelView):
    model = Article
    searches = [
        # Exact matches
        'id__exact',
        'slug__exact',
        
        # Case-insensitive partial matches
        'title__icontains',
        'content__icontains',
        
        # Relationship searches
        'author__username__icontains',
        'author__first_name__icontains',
        'author__last_name__icontains',
        
        # Many-to-many searches
        'tags__name__icontains',
        'categories__name__icontains',
    ]
```

### Fuzzy Search

Django Binder includes a fuzzy search lookup:

```python
class ArticleView(ModelView):
    model = Article
    searches = [
        'title__fuzzy',    # Fuzzy matching on title
        'content__fuzzy',  # Fuzzy matching on content
    ]
```

Usage:
```bash
# Fuzzy search matches partial words and handles typos
GET /api/article/?search=djnago  # Matches "django"
```

## Alternative Filters

### Basic Alternative Filters

Group related filters under named categories:

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
        'status': ['published', 'featured', 'archived'],
    }
```

Usage:
```bash
# Use alternative filters
GET /api/article/?content_search=django
GET /api/article/?author_filter=john
GET /api/article/?date_range.created_at__gte=2023-01-01
GET /api/article/?status=published
```

### Advanced Alternative Filters

Combine filters with complex logic:

```python
class ArticleView(ModelView):
    model = Article
    alternative_filters = {
        # Multi-field search
        'full_search': [
            'title__icontains',
            'content__icontains',
            'author__username__icontains',
            'tags__name__icontains'
        ],
        
        # Category hierarchy
        'category_tree': [
            'category__name',
            'category__slug',
            'category__parent__name',
            'category__parent__slug'
        ],
        
        # Engagement metrics
        'engagement': [
            'view_count__gte',
            'comment_count__gte',
            'like_count__gte',
            'share_count__gte'
        ],
    }
```

### Filter Operators

Use `:any` and `:all` operators for complex logic:

```python
# ANY logic (default) - matches if ANY condition is true
GET /api/article/?author_filter:any=john,jane

# ALL logic - matches if ALL conditions are true
GET /api/article/?tags:all=python,django

# Negation with :not
GET /api/article/?status:not=archived
GET /api/article/?author_filter:not:any=banned_user1,banned_user2

# Combination of operators
GET /api/article/?tags:not:all=spam,inappropriate
```

## Custom Filtering

### Override Filter Methods

Implement custom filtering logic:

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
        
        # Custom filter: reading difficulty
        difficulty = request.GET.get('difficulty')
        if difficulty == 'easy':
            queryset = queryset.extra(
                where=["LENGTH(content) < 1000"]
            )
        elif difficulty == 'hard':
            queryset = queryset.extra(
                where=["LENGTH(content) > 5000"]
            )
        
        # Custom filter: user's preferred language
        if hasattr(request.user, 'profile') and request.user.profile.language:
            lang = request.GET.get('user_language')
            if lang == 'preferred':
                queryset = queryset.filter(
                    language=request.user.profile.language
                )
        
        return queryset
```

### Custom Filter Fields

Add computed filter fields:

```python
class ArticleView(ModelView):
    model = Article
    
    def get_queryset(self, request):
        """Add annotations for filtering"""
        queryset = super().get_queryset(request)
        
        # Add computed fields for filtering
        queryset = queryset.annotate(
            word_count=Length('content') / 5,  # Approximate word count
            engagement_score=(
                F('view_count') + F('comment_count') * 10 + F('like_count') * 5
            ),
            is_recent=Case(
                When(created_at__gte=timezone.now() - timedelta(days=7), then=True),
                default=False,
                output_field=models.BooleanField()
            )
        )
        
        return queryset
    
    def filter_queryset(self, request, queryset):
        """Filter on computed fields"""
        queryset = super().filter_queryset(request, queryset)
        
        # Filter by word count
        min_words = request.GET.get('min_words')
        if min_words:
            queryset = queryset.filter(word_count__gte=int(min_words))
        
        # Filter by engagement score
        min_engagement = request.GET.get('min_engagement')
        if min_engagement:
            queryset = queryset.filter(engagement_score__gte=int(min_engagement))
        
        # Filter recent articles
        if request.GET.get('recent') == 'true':
            queryset = queryset.filter(is_recent=True)
        
        return queryset
```

## Geographic Filtering

### Location-Based Filtering

Filter by geographic data (requires PostGIS):

```python
from django.contrib.gis.db import models as gis_models
from django.contrib.gis.geos import Point
from django.contrib.gis.measure import Distance

class LocationView(ModelView):
    model = Location
    
    def filter_queryset(self, request, queryset):
        queryset = super().filter_queryset(request, queryset)
        
        # Filter by distance from point
        lat = request.GET.get('lat')
        lng = request.GET.get('lng')
        radius = request.GET.get('radius')
        
        if lat and lng and radius:
            point = Point(float(lng), float(lat), srid=4326)
            distance = Distance(km=float(radius))
            queryset = queryset.filter(
                location__distance_lte=(point, distance)
            )
        
        # Filter by bounding box
        bbox = request.GET.get('bbox')  # "min_lng,min_lat,max_lng,max_lat"
        if bbox:
            coords = [float(x) for x in bbox.split(',')]
            queryset = queryset.filter(
                location__longitude__gte=coords[0],
                location__latitude__gte=coords[1],
                location__longitude__lte=coords[2],
                location__latitude__lte=coords[3]
            )
        
        return queryset
```

## Performance Optimization

### Efficient Filtering

Optimize filter performance:

```python
class ArticleView(ModelView):
    model = Article
    
    def get_queryset(self, request):
        """Optimized queryset with proper indexing"""
        queryset = super().get_queryset(request)
        
        # Use select_related for foreign keys that will be filtered
        queryset = queryset.select_related('author', 'category')
        
        # Use prefetch_related for many-to-many fields
        queryset = queryset.prefetch_related('tags')
        
        return queryset
    
    def filter_queryset(self, request, queryset):
        """Apply filters in optimal order"""
        queryset = super().filter_queryset(request, queryset)
        
        # Apply most selective filters first
        if request.GET.get('published') == 'true':
            queryset = queryset.filter(published=True)
        
        # Use database functions for complex filtering
        search_term = request.GET.get('search')
        if search_term:
            queryset = queryset.extra(
                where=["to_tsvector('english', title || ' ' || content) @@ plainto_tsquery(%s)"],
                params=[search_term]
            )
        
        return queryset
```

### Filter Caching

Cache expensive filter operations:

```python
from django.core.cache import cache

class ArticleView(ModelView):
    model = Article
    
    def filter_queryset(self, request, queryset):
        # Create cache key from query parameters
        cache_key = f"filtered_articles:{hash(str(sorted(request.GET.items())))}"
        
        # Try to get cached result
        cached_ids = cache.get(cache_key)
        if cached_ids is not None:
            return queryset.filter(id__in=cached_ids)
        
        # Apply filters
        filtered_queryset = super().filter_queryset(request, queryset)
        
        # Cache the result IDs
        result_ids = list(filtered_queryset.values_list('id', flat=True))
        cache.set(cache_key, result_ids, timeout=300)  # 5 minutes
        
        return filtered_queryset
```

## Filter Validation

### Input Validation

Validate filter parameters:

```python
from django.core.exceptions import ValidationError

class ArticleView(ModelView):
    model = Article
    
    def filter_queryset(self, request, queryset):
        # Validate date parameters
        start_date = request.GET.get('start_date')
        if start_date:
            try:
                datetime.strptime(start_date, '%Y-%m-%d')
            except ValueError:
                raise ValidationError('Invalid date format. Use YYYY-MM-DD.')
        
        # Validate numeric parameters
        min_rating = request.GET.get('min_rating')
        if min_rating:
            try:
                rating = float(min_rating)
                if not 0 <= rating <= 5:
                    raise ValidationError('Rating must be between 0 and 5.')
            except ValueError:
                raise ValidationError('Invalid rating format.')
        
        # Validate choice parameters
        status = request.GET.get('status')
        if status and status not in ['draft', 'published', 'archived']:
            raise ValidationError('Invalid status. Must be draft, published, or archived.')
        
        return super().filter_queryset(request, queryset)
```

### Security Filtering

Prevent unauthorized data access:

```python
class ArticleView(ModelView):
    model = Article
    
    def filter_queryset(self, request, queryset):
        queryset = super().filter_queryset(request, queryset)
        
        # Security: Users can only see their own drafts
        if not request.user.is_staff:
            queryset = queryset.filter(
                Q(published=True) | Q(author=request.user)
            )
        
        # Security: Filter by user's department
        if hasattr(request.user, 'profile') and request.user.profile.department:
            dept_filter = request.GET.get('department')
            if dept_filter and dept_filter != request.user.profile.department:
                # User trying to access other department's data
                return queryset.none()
        
        return queryset
```

## Testing Filters

### Filter Testing

Test your filtering logic:

```python
from django.test import TestCase
from django.contrib.auth.models import User

class FilterTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        self.article1 = Article.objects.create(
            title='Django Tutorial',
            content='Learn Django',
            author=self.user,
            published=True
        )
        self.article2 = Article.objects.create(
            title='Python Guide',
            content='Learn Python',
            author=self.user,
            published=False
        )
    
    def test_published_filter(self):
        """Test published filter"""
        response = self.client.get('/api/article/?published=true')
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertEqual(len(data['data']), 1)
        self.assertEqual(data['data'][0]['title'], 'Django Tutorial')
    
    def test_search_filter(self):
        """Test search functionality"""
        response = self.client.get('/api/article/?search=Django')
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertEqual(len(data['data']), 1)
        self.assertEqual(data['data'][0]['title'], 'Django Tutorial')
    
    def test_alternative_filter(self):
        """Test alternative filters"""
        response = self.client.get('/api/article/?content_search=Python')
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertEqual(len(data['data']), 1)
        self.assertEqual(data['data'][0]['title'], 'Python Guide')
```

This comprehensive filtering system provides powerful querying capabilities while maintaining performance and security.
