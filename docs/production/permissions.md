# Permissions & Security

Django Binder provides a comprehensive permission system that integrates seamlessly with Django's authentication framework while offering fine-grained access control for API endpoints.

## Permission System Overview

Django Binder's permission system operates on four main permission types:
- **view**: Controls who can read/retrieve data
- **add**: Controls who can create new records
- **change**: Controls who can modify existing records  
- **delete**: Controls who can delete records

Each permission can be scoped to provide granular access control.

## Basic Setup

### 1. Create Permissions Configuration

Create a `permissions.py` file in your project:

```python
# permissions.py
permissions = {
    'default': [
        # Basic authentication permissions
        ('auth.view_user', 'own'),
        ('auth.change_user', 'own'),
        ('auth.login_user', None),
        ('auth.logout_user', None),
        ('auth.unmasquerade_user', None),
    ],
    
    'staff': [
        # Staff permissions
        ('blog.view_article', 'all'),
        ('blog.add_article', 'all'),
        ('blog.change_article', 'all'),
        ('blog.delete_article', 'all'),
        ('blog.view_category', 'all'),
        ('blog.add_category', 'all'),
    ],
    
    'author': [
        # Author permissions
        ('blog.view_article', 'all'),
        ('blog.add_article', 'all'),
        ('blog.change_article', 'own'),
        ('blog.delete_article', 'own'),
        ('blog.view_category', 'all'),
    ],
    
    'reader': [
        # Reader permissions
        ('blog.view_article', 'published'),
        ('blog.view_category', 'all'),
    ],
}
```

### 2. Configure Settings

Add the permissions to your Django settings:

```python
# settings.py
from .permissions import permissions

BINDER_PERMISSION = permissions

# Optional: Configure permission backend
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'binder.permissions.backends.BinderPermissionBackend',
]
```

### 3. Create Permission-Enabled Views

Use `PermissionView` instead of `ModelView`:

```python
# views.py
from binder.permissions.views import PermissionView
from .models import Article

class ArticleView(PermissionView):
    model = Article
    
    # View configuration remains the same
    searches = ['title__icontains', 'content__icontains']
    file_fields = ['featured_image']
```

## Built-in Scopes

Django Binder provides several built-in permission scopes:

### `all` Scope
Grants access to all records:
```python
('blog.view_article', 'all')  # Can view all articles
```

### `own` Scope  
Grants access only to records owned by the user:
```python
('blog.change_article', 'own')  # Can only edit own articles
```

### `none` Scope
Explicitly denies access:
```python
('blog.delete_article', 'none')  # Cannot delete any articles
```

### `None` Permission
No permission check (use carefully):
```python
('auth.login_user', None)  # No permission check for login
```

## Custom Scopes

Create custom scopes by implementing scope methods in your views:

### View Scopes

Control which records users can see:

```python
from django.db.models import Q
from binder.views import FilterDescription

class ArticleView(PermissionView):
    model = Article
    
    def _scope_view_published(self, request):
        """Users can only view published articles"""
        return Q(published=True)
    
    def _scope_view_own_or_published(self, request):
        """Users can view their own articles or published ones"""
        return Q(
            Q(author=request.user) | Q(published=True)
        )
    
    def _scope_view_department(self, request):
        """Users can view articles from their department"""
        if hasattr(request.user, 'profile'):
            return FilterDescription(
                Q(author__profile__department=request.user.profile.department),
                need_distinct=True  # Needed when filtering across relationships
            )
        return Q(pk__in=[])  # No access if no profile
    
    def _scope_view_recent(self, request):
        """Users can only view articles from last 30 days"""
        from datetime import timedelta
        cutoff_date = timezone.now() - timedelta(days=30)
        return Q(created_at__gte=cutoff_date)
```

### Add/Change/Delete Scopes

Control create, update, and delete operations:

```python
class ArticleView(PermissionView):
    model = Article
    
    def _scope_add_department_only(self, request, obj, values):
        """Users can only create articles in their department"""
        if hasattr(request.user, 'profile'):
            # Check if the article's category belongs to user's department
            category_id = values.get('category')
            if category_id:
                category = Category.objects.get(id=category_id)
                return category.department == request.user.profile.department
        return False
    
    def _scope_change_own(self, request, obj, values):
        """Users can only edit their own articles"""
        return obj.author == request.user
    
    def _scope_change_before_published(self, request, obj, values):
        """Users can only edit articles before they're published"""
        return not obj.published
    
    def _scope_delete_own_unpublished(self, request, obj, values):
        """Users can only delete their own unpublished articles"""
        return obj.author == request.user and not obj.published
```

## Advanced Permission Patterns

### Role-Based Permissions

Implement role-based access control:

```python
# models.py
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    role = models.CharField(max_length=20, choices=[
        ('admin', 'Administrator'),
        ('editor', 'Editor'),
        ('author', 'Author'),
        ('reader', 'Reader'),
    ])
    department = models.CharField(max_length=50)

# views.py
class ArticleView(PermissionView):
    model = Article
    
    def _scope_view_by_role(self, request):
        """Different view permissions based on user role"""
        if not hasattr(request.user, 'profile'):
            return Q(pk__in=[])  # No access without profile
        
        role = request.user.profile.role
        
        if role == 'admin':
            return Q()  # Can see everything
        elif role == 'editor':
            return Q(category__department=request.user.profile.department)
        elif role == 'author':
            return Q(
                Q(author=request.user) | 
                Q(published=True, category__department=request.user.profile.department)
            )
        else:  # reader
            return Q(published=True)
    
    def _scope_change_by_role(self, request, obj, values):
        """Different change permissions based on role"""
        if not hasattr(request.user, 'profile'):
            return False
        
        role = request.user.profile.role
        
        if role == 'admin':
            return True
        elif role == 'editor':
            return obj.category.department == request.user.profile.department
        elif role == 'author':
            return obj.author == request.user and not obj.published
        else:
            return False
```

### Time-Based Permissions

Implement time-sensitive permissions:

```python
class ArticleView(PermissionView):
    model = Article
    
    def _scope_change_editing_window(self, request, obj, values):
        """Users can only edit articles within 24 hours of creation"""
        from datetime import timedelta
        
        if obj.author != request.user:
            return False
        
        editing_deadline = obj.created_at + timedelta(hours=24)
        return timezone.now() <= editing_deadline
    
    def _scope_view_embargo(self, request):
        """Hide articles that are under embargo"""
        return Q(
            Q(embargo_until__isnull=True) | 
            Q(embargo_until__lte=timezone.now())
        )
```

### Hierarchical Permissions

Implement hierarchical access control:

```python
class CategoryView(PermissionView):
    model = Category
    
    def _scope_view_hierarchy(self, request):
        """Users can view categories they have access to and their children"""
        if not hasattr(request.user, 'profile'):
            return Q(pk__in=[])
        
        # Get user's accessible categories
        accessible_categories = self.get_user_categories(request.user)
        
        # Include all descendants of accessible categories
        descendant_ids = []
        for category in accessible_categories:
            descendant_ids.extend(category.get_descendant_ids())
        
        return Q(id__in=descendant_ids)
    
    def get_user_categories(self, user):
        """Get categories user has access to"""
        # Implementation depends on your hierarchy model
        pass
```

### Content-Based Permissions

Implement permissions based on content attributes:

```python
class ArticleView(PermissionView):
    model = Article
    
    def _scope_view_content_rating(self, request):
        """Filter content based on user's content rating preference"""
        if hasattr(request.user, 'profile'):
            max_rating = request.user.profile.max_content_rating
            return Q(content_rating__lte=max_rating)
        return Q(content_rating='G')  # Default to general audience
    
    def _scope_view_language(self, request):
        """Show content in user's preferred languages"""
        if hasattr(request.user, 'profile'):
            languages = request.user.profile.preferred_languages.all()
            return Q(language__in=languages)
        return Q(language='en')  # Default to English
```

## Permission Debugging

### Debug Permission Checks

Enable permission debugging in development:

```python
# settings.py
BINDER_DEBUG_PERMISSIONS = True

# This will log all permission checks
LOGGING = {
    'version': 1,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'loggers': {
        'binder.permissions': {
            'handlers': ['console'],
            'level': 'DEBUG',
        },
    },
}
```

### Permission Testing View

Create a debug view to test permissions:

```python
from binder.router import list_route
from django.http import JsonResponse

class ArticleView(PermissionView):
    model = Article
    
    @list_route(name='debug_permissions')
    def debug_permissions(self, request):
        """Debug endpoint to check user permissions"""
        if not settings.DEBUG:
            return JsonResponse({'error': 'Only available in debug mode'})
        
        permissions = {}
        
        # Check each permission type
        for perm_type in ['view', 'add', 'change', 'delete']:
            try:
                scopes = self._require_model_perm(perm_type, request)
                permissions[perm_type] = scopes
            except Exception as e:
                permissions[perm_type] = f'Error: {str(e)}'
        
        return JsonResponse({
            'user': request.user.username,
            'permissions': permissions,
            'user_groups': [g.name for g in request.user.groups.all()],
        })
```

## Security Best Practices

### 1. Principle of Least Privilege

Grant users the minimum permissions necessary:

```python
permissions = {
    'default': [
        # Only essential permissions for all users
        ('auth.view_user', 'own'),
        ('auth.change_user', 'own'),
    ],
    
    'content_viewer': [
        # Minimal read-only access
        ('blog.view_article', 'published'),
        ('blog.view_category', 'all'),
    ],
    
    # Add more specific roles as needed
}
```

### 2. Validate Permission Scopes

Always validate that custom scopes work correctly:

```python
class ArticleView(PermissionView):
    model = Article
    
    def _scope_view_own(self, request):
        """Ensure user can only see their own articles"""
        if not request.user.is_authenticated:
            return Q(pk__in=[])  # No access for anonymous users
        
        return Q(author=request.user)
    
    def _scope_change_own(self, request, obj, values):
        """Ensure user can only edit their own articles"""
        if not request.user.is_authenticated:
            return False
        
        # Additional validation
        if obj.published and not request.user.is_staff:
            return False  # Can't edit published articles unless staff
        
        return obj.author == request.user
```

### 3. Handle Edge Cases

Account for edge cases in permission logic:

```python
class ArticleView(PermissionView):
    model = Article
    
    def _scope_view_department(self, request):
        """Handle users without department assignment"""
        if not hasattr(request.user, 'profile'):
            # Log security event
            logger.warning(f'User {request.user.id} has no profile')
            return Q(pk__in=[])  # No access
        
        if not request.user.profile.department:
            # Log security event
            logger.warning(f'User {request.user.id} has no department')
            return Q(pk__in=[])  # No access
        
        return Q(category__department=request.user.profile.department)
```

### 4. Audit Permission Changes

Log permission-related events:

```python
import logging

permission_logger = logging.getLogger('security.permissions')

class ArticleView(PermissionView):
    model = Article
    
    def _require_model_perm(self, perm_type, request, pk=None):
        """Override to add audit logging"""
        try:
            scopes = super()._require_model_perm(perm_type, request, pk)
            
            # Log successful permission check
            permission_logger.info(
                f'Permission granted: user={request.user.id}, '
                f'perm={perm_type}, model={self.model.__name__}, '
                f'pk={pk}, scopes={scopes}'
            )
            
            return scopes
            
        except Exception as e:
            # Log permission denial
            permission_logger.warning(
                f'Permission denied: user={request.user.id}, '
                f'perm={perm_type}, model={self.model.__name__}, '
                f'pk={pk}, error={str(e)}'
            )
            raise
```

## Integration with Django Groups

### Group-Based Permissions

Integrate with Django's group system:

```python
# Create groups and assign permissions
from django.contrib.auth.models import Group, Permission

# Create groups
editors_group = Group.objects.create(name='editors')
authors_group = Group.objects.create(name='authors')

# Assign permissions
article_permissions = Permission.objects.filter(content_type__app_label='blog')
editors_group.permissions.set(article_permissions)

# In your permission configuration
permissions = {
    'editors': [
        ('blog.view_article', 'all'),
        ('blog.add_article', 'all'),
        ('blog.change_article', 'all'),
        ('blog.delete_article', 'all'),
    ],
    'authors': [
        ('blog.view_article', 'all'),
        ('blog.add_article', 'all'),
        ('blog.change_article', 'own'),
    ],
}
```

### Dynamic Group Assignment

Automatically assign users to groups:

```python
# signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User, Group

@receiver(post_save, sender=User)
def assign_default_group(sender, instance, created, **kwargs):
    """Assign new users to default group"""
    if created:
        default_group = Group.objects.get(name='readers')
        instance.groups.add(default_group)
```

## Testing Permissions

### Permission Test Cases

```python
from django.test import TestCase
from django.contrib.auth.models import User, Group
from rest_framework.test import APIClient

class PermissionTest(TestCase):
    def setUp(self):
        self.client = APIClient()
        
        # Create test users
        self.admin = User.objects.create_user('admin', 'admin@test.com', 'pass')
        self.author = User.objects.create_user('author', 'author@test.com', 'pass')
        self.reader = User.objects.create_user('reader', 'reader@test.com', 'pass')
        
        # Create groups
        admin_group = Group.objects.create(name='admin')
        author_group = Group.objects.create(name='author')
        reader_group = Group.objects.create(name='reader')
        
        # Assign users to groups
        self.admin.groups.add(admin_group)
        self.author.groups.add(author_group)
        self.reader.groups.add(reader_group)
    
    def test_admin_can_view_all_articles(self):
        """Test admin can view all articles"""
        self.client.force_authenticate(user=self.admin)
        response = self.client.get('/api/article/')
        self.assertEqual(response.status_code, 200)
    
    def test_reader_can_only_view_published(self):
        """Test reader can only view published articles"""
        # Create published and unpublished articles
        published = Article.objects.create(
            title='Published', content='Content', published=True
        )
        unpublished = Article.objects.create(
            title='Unpublished', content='Content', published=False
        )
        
        self.client.force_authenticate(user=self.reader)
        response = self.client.get('/api/article/')
        
        # Should only see published article
        self.assertEqual(len(response.json()['data']), 1)
        self.assertEqual(response.json()['data'][0]['id'], published.id)
    
    def test_author_cannot_edit_others_articles(self):
        """Test author cannot edit articles by other authors"""
        other_author = User.objects.create_user('other', 'other@test.com', 'pass')
        article = Article.objects.create(
            title='Test', content='Content', author=other_author
        )
        
        self.client.force_authenticate(user=self.author)
        response = self.client.put(f'/api/article/{article.id}/', {
            'title': 'Modified Title'
        })
        
        # Should be forbidden
        self.assertEqual(response.status_code, 403)
```

This comprehensive permission system provides fine-grained access control while maintaining security and auditability.
