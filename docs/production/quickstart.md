# Quick Start Guide

Get up and running with Django Binder in minutes. This guide will walk you through creating a complete REST API for a blog application.

## Prerequisites

- Django Binder installed ([Installation Guide](installation.md))
- Basic Django knowledge
- Python 3.6+ environment

## Step 1: Create Your Models

Let's create a simple blog with authors, categories, and posts:

```python
# blog/models.py
from django.db import models
from django.contrib.auth.models import User
from binder.models import BinderModel, BinderImageField

class Category(BinderModel):
    name = models.CharField(max_length=100, unique=True)
    description = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        verbose_name_plural = "categories"
        ordering = ['name']
    
    def __str__(self):
        return self.name

class Author(BinderModel):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    bio = models.TextField(blank=True)
    avatar = BinderImageField(upload_to='avatars/', blank=True, null=True)
    website = models.URLField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.user.get_full_name() or self.user.username

class Post(BinderModel):
    title = models.CharField(max_length=200)
    slug = models.SlugField(unique=True)
    content = models.TextField()
    excerpt = models.TextField(blank=True)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name='posts')
    category = models.ForeignKey(Category, on_delete=models.CASCADE, related_name='posts')
    tags = models.ManyToManyField('Tag', blank=True, related_name='posts')
    featured_image = BinderImageField(upload_to='posts/', blank=True, null=True)
    published = models.BooleanField(default=False)
    published_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    # Enable history tracking
    class Binder:
        history = True
    
    class Meta:
        ordering = ['-published_at', '-created_at']
    
    def __str__(self):
        return self.title

class Tag(BinderModel):
    name = models.CharField(max_length=50, unique=True)
    color = models.CharField(max_length=7, default='#007bff')  # Hex color
    
    class Meta:
        ordering = ['name']
    
    def __str__(self):
        return self.name

class Comment(BinderModel):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author_name = models.CharField(max_length=100)
    author_email = models.EmailField()
    content = models.TextField()
    approved = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['-created_at']
    
    def __str__(self):
        return f'Comment by {self.author_name} on {self.post.title}'
```

## Step 2: Create Your Views

Create powerful API views with minimal code:

```python
# blog/views.py
from django.db.models import Q, Count
from binder.views import ModelView
from binder.router import list_route, detail_route
from django.http import JsonResponse
from .models import Category, Author, Post, Tag, Comment

class CategoryView(ModelView):
    model = Category
    
    # Enable searching
    searches = ['name__icontains', 'description__icontains']
    
    # Custom endpoint to get category statistics
    @detail_route(name='stats')
    def get_stats(self, request, pk):
        category = self.get_object(pk)
        stats = {
            'total_posts': category.posts.count(),
            'published_posts': category.posts.filter(published=True).count(),
            'total_authors': category.posts.values('author').distinct().count(),
        }
        return JsonResponse(stats)

class AuthorView(ModelView):
    model = Author
    file_fields = ['avatar']
    
    # Include related data
    shown_properties = ['post_count']
    
    # Custom property for post count
    def get_post_count(self, obj):
        return obj.posts.filter(published=True).count()
    
    # Search across user fields
    searches = ['user__username__icontains', 'user__first_name__icontains', 
                'user__last_name__icontains', 'bio__icontains']

class PostView(ModelView):
    model = Post
    file_fields = ['featured_image']
    m2m_fields = ['tags']
    
    # Advanced filtering options
    alternative_filters = {
        'content_search': ['title__icontains', 'content__icontains', 'excerpt__icontains'],
        'author_search': ['author__user__username', 'author__user__first_name', 'author__user__last_name'],
        'date_range': ['published_at__gte', 'published_at__lte'],
        'category_filter': ['category__name'],
    }
    
    # Custom queryset with optimizations
    def get_queryset(self, request):
        return super().get_queryset(request).select_related(
            'author', 'author__user', 'category'
        ).prefetch_related('tags', 'comments')
    
    # Custom endpoint for published posts only
    @list_route(name='published')
    def published_posts(self, request):
        queryset = self.get_queryset(request).filter(published=True)
        return self.get_list_response(request, queryset)
    
    # Custom endpoint to publish/unpublish a post
    @detail_route(name='toggle_publish', methods=['POST'])
    def toggle_publish(self, request, pk):
        post = self.get_object(pk)
        post.published = not post.published
        if post.published and not post.published_at:
            from django.utils import timezone
            post.published_at = timezone.now()
        post.save()
        return JsonResponse({'published': post.published})

class TagView(ModelView):
    model = Tag
    
    # Search by name
    searches = ['name__icontains']
    
    # Custom endpoint for popular tags
    @list_route(name='popular')
    def popular_tags(self, request):
        queryset = self.get_queryset(request).annotate(
            post_count=Count('posts')
        ).filter(post_count__gt=0).order_by('-post_count')
        return self.get_list_response(request, queryset)

class CommentView(ModelView):
    model = Comment
    
    # Filter options
    alternative_filters = {
        'post_filter': ['post__id', 'post__slug'],
        'approval_status': ['approved'],
        'author_search': ['author_name__icontains', 'author_email__icontains'],
    }
    
    # Custom endpoint to approve/reject comments
    @detail_route(name='approve', methods=['POST'])
    def approve_comment(self, request, pk):
        comment = self.get_object(pk)
        comment.approved = True
        comment.save()
        return JsonResponse({'approved': True})
    
    @detail_route(name='reject', methods=['POST'])
    def reject_comment(self, request, pk):
        comment = self.get_object(pk)
        comment.approved = False
        comment.save()
        return JsonResponse({'approved': False})
```

## Step 3: Run Migrations

Create and apply database migrations:

```bash
# Create migrations
python manage.py makemigrations blog

# Apply migrations
python manage.py migrate

# Create some sample data (optional)
python manage.py shell
```

Create sample data in the Django shell:

```python
# In Django shell
from django.contrib.auth.models import User
from blog.models import Category, Author, Post, Tag

# Create a user and author
user = User.objects.create_user('john', 'john@example.com', 'password')
author = Author.objects.create(user=user, bio='A passionate writer')

# Create categories
tech = Category.objects.create(name='Technology', description='Tech-related posts')
lifestyle = Category.objects.create(name='Lifestyle', description='Lifestyle content')

# Create tags
python_tag = Tag.objects.create(name='Python', color='#3776ab')
django_tag = Tag.objects.create(name='Django', color='#092e20')

# Create a post
post = Post.objects.create(
    title='Getting Started with Django Binder',
    slug='getting-started-django-binder',
    content='Django Binder is an amazing framework...',
    excerpt='Learn how to build APIs quickly with Django Binder',
    author=author,
    category=tech,
    published=True
)
post.tags.add(python_tag, django_tag)
```

## Step 4: Test Your API

Start the development server:

```bash
python manage.py runserver
```

### Basic CRUD Operations

```bash
# List all posts
curl http://localhost:8000/api/post/

# Get a specific post
curl http://localhost:8000/api/post/1/

# Create a new post
curl -X POST http://localhost:8000/api/post/ \
  -H "Content-Type: application/json" \
  -d '{
    "title": "My New Post",
    "slug": "my-new-post",
    "content": "This is the content of my new post",
    "author": 1,
    "category": 1,
    "published": true
  }'

# Update a post
curl -X PUT http://localhost:8000/api/post/1/ \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Updated Post Title",
    "content": "Updated content"
  }'

# Delete a post
curl -X DELETE http://localhost:8000/api/post/1/
```

### Advanced Filtering

```bash
# Search in content
curl "http://localhost:8000/api/post/?content_search=django"

# Filter by category
curl "http://localhost:8000/api/post/?category_filter=Technology"

# Filter by author
curl "http://localhost:8000/api/post/?author_search=john"

# Date range filtering
curl "http://localhost:8000/api/post/?date_range.published_at__gte=2023-01-01"

# Multiple filters
curl "http://localhost:8000/api/post/?published=true&category_filter=Technology&order_by=-published_at"
```

### Custom Endpoints

```bash
# Get published posts only
curl http://localhost:8000/api/post/published/

# Get category statistics
curl http://localhost:8000/api/category/1/stats/

# Get popular tags
curl http://localhost:8000/api/tag/popular/

# Toggle post publication status
curl -X POST http://localhost:8000/api/post/1/toggle_publish/

# Approve a comment
curl -X POST http://localhost:8000/api/comment/1/approve/
```

### Working with Relations

```bash
# Get post with related data
curl "http://localhost:8000/api/post/1/?with=author,category,tags"

# Get author with their posts
curl "http://localhost:8000/api/author/1/?with=posts"

# Get category with posts and their authors
curl "http://localhost:8000/api/category/1/?with=posts.author"
```

### File Uploads

```bash
# Upload author avatar
curl -X POST http://localhost:8000/api/author/1/avatar/ \
  -F "file=@avatar.jpg"

# Upload post featured image
curl -X POST http://localhost:8000/api/post/1/featured_image/ \
  -F "file=@featured.jpg"
```

## Step 5: Frontend Integration

### JavaScript/React Example

```javascript
// API client setup
class BlogAPI {
  constructor(baseURL = 'http://localhost:8000/api') {
    this.baseURL = baseURL;
  }

  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    const response = await fetch(url, {
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
      ...options,
    });
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    
    return response.json();
  }

  // Get all posts with filtering
  async getPosts(filters = {}) {
    const params = new URLSearchParams(filters);
    return this.request(`/post/?${params}`);
  }

  // Get a single post with relations
  async getPost(id, withRelations = ['author', 'category', 'tags']) {
    const with_param = withRelations.join(',');
    return this.request(`/post/${id}/?with=${with_param}`);
  }

  // Create a new post
  async createPost(postData) {
    return this.request('/post/', {
      method: 'POST',
      body: JSON.stringify(postData),
    });
  }

  // Update a post
  async updatePost(id, postData) {
    return this.request(`/post/${id}/`, {
      method: 'PUT',
      body: JSON.stringify(postData),
    });
  }

  // Toggle post publication
  async togglePublish(id) {
    return this.request(`/post/${id}/toggle_publish/`, {
      method: 'POST',
    });
  }
}

// Usage example
const api = new BlogAPI();

// Load and display posts
async function loadPosts() {
  try {
    const response = await api.getPosts({
      published: true,
      order_by: '-published_at',
      limit: 10
    });
    
    console.log('Posts:', response.data);
    // Render posts in your UI
  } catch (error) {
    console.error('Error loading posts:', error);
  }
}

// Create a new post
async function createPost() {
  try {
    const newPost = await api.createPost({
      title: 'My New Post',
      slug: 'my-new-post',
      content: 'Post content here...',
      author: 1,
      category: 1,
      published: true
    });
    
    console.log('Created post:', newPost);
  } catch (error) {
    console.error('Error creating post:', error);
  }
}
```

## Next Steps

Congratulations! You now have a fully functional blog API with Django Binder. Here's what to explore next:

### Essential Reading
- [Models & Fields](models.md) - Learn about advanced model features
- [Views & ViewSets](views.md) - Master custom endpoint creation
- [Filtering & Searching](filtering.md) - Advanced querying techniques
- [Permissions & Security](permissions.md) - Secure your API

### Advanced Features
- [File Handling](file-handling.md) - Advanced file upload and processing
- [History & Auditing](history.md) - Track all changes to your models
- [Multi-PUT Operations](multi-put.md) - Batch operations for efficiency
- [WebSocket Support](websockets.md) - Real-time updates

### Production Deployment
- [Performance Optimization](performance.md) - Scale your application
- [Security Best Practices](security.md) - Production security
- [Monitoring & Logging](monitoring.md) - Monitor your API in production

## Common Patterns

### Soft Delete Pattern
```python
class SoftDeleteModel(BinderModel):
    deleted_at = models.DateTimeField(null=True, blank=True)
    
    def delete(self):
        self.deleted_at = timezone.now()
        self.save()
    
    def hard_delete(self):
        super().delete()

class SoftDeleteView(ModelView):
    def get_queryset(self, request):
        return super().get_queryset(request).filter(deleted_at__isnull=True)
```

### Versioned API Pattern
```python
class VersionedView(ModelView):
    def get_queryset(self, request):
        version = request.META.get('HTTP_API_VERSION', 'v1')
        if version == 'v2':
            # Return different data for v2
            pass
        return super().get_queryset(request)
```

### Cached Responses Pattern
```python
from django.core.cache import cache

class CachedView(ModelView):
    cache_timeout = 300  # 5 minutes
    
    def get_list_response(self, request, queryset):
        cache_key = f"api:{self.model.__name__}:{hash(str(queryset.query))}"
        cached_response = cache.get(cache_key)
        
        if cached_response:
            return cached_response
            
        response = super().get_list_response(request, queryset)
        cache.set(cache_key, response, self.cache_timeout)
        return response
```

You're now ready to build powerful, production-ready APIs with Django Binder!
