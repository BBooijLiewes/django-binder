# History & Auditing

Django Binder provides comprehensive change tracking and auditing capabilities that automatically log all modifications to your models, including who made changes, when they were made, and what the changes were.

## Enabling History Tracking

### Basic History Setup

Enable history tracking on your models:

```python
from binder.models import BinderModel

class Article(BinderModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    published = models.BooleanField(default=False)
    
    class Binder:
        history = True  # Enable change tracking
```

### Install Signal Handlers

Configure history signal handlers in your URL configuration:

```python
# urls.py
import binder.models

# Install history signal handlers - required for history tracking
binder.models.install_history_signal_handlers(binder.models.BinderModel)
```

## History Models

Django Binder creates two models to track changes:

### ChangeSet Model

Represents a group of changes made in a single operation:

```python
class ChangeSet(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, null=True)
    timestamp = models.DateTimeField(auto_now_add=True)
    request_id = models.CharField(max_length=36, null=True)
    
    # Automatically created for each save operation
```

### Change Model

Represents individual field changes:

```python
class Change(models.Model):
    changeset = models.ForeignKey(ChangeSet, on_delete=models.CASCADE)
    model_name = models.CharField(max_length=100)
    object_id = models.PositiveIntegerField()
    field = models.CharField(max_length=100)
    old_value = models.TextField(null=True)
    new_value = models.TextField(null=True)
    
    # One record per changed field
```

## Viewing History

### API Endpoints

History is automatically available via API endpoints:

```bash
# View change history for a specific object
GET /api/article/1/history/

# Debug view with more detailed information
GET /api/article/1/history/debug/
```

### History Response Format

```json
{
  "data": [
    {
      "id": 1,
      "timestamp": "2023-12-01T10:30:00Z",
      "user": {
        "id": 5,
        "username": "editor",
        "email": "editor@example.com"
      },
      "changes": [
        {
          "field": "title",
          "old_value": "Draft Title",
          "new_value": "Published Title"
        },
        {
          "field": "published",
          "old_value": false,
          "new_value": true
        },
        {
          "field": "published_at",
          "old_value": null,
          "new_value": "2023-12-01T10:30:00Z"
        }
      ]
    }
  ]
}
```

## Custom History Display

### Format Instance for History

Customize how related objects are displayed in history:

```python
class Author(BinderModel):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    bio = models.TextField(blank=True)
    
    class Binder:
        history = True
    
    @classmethod
    def format_instance_for_history(cls, id: int):
        """Custom display format for history entries"""
        try:
            author = cls.objects.select_related('user').get(id=id)
            return f"{author.user.get_full_name()} ({author.user.username})"
        except cls.DoesNotExist:
            return f'deleted author {id}'

class Article(BinderModel):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    
    class Binder:
        history = True
    
    @classmethod
    def format_instance_for_history(cls, id: int):
        """Custom display format for articles"""
        try:
            article = cls.objects.get(id=id)
            return f'"{article.title}" (ID: {id})'
        except cls.DoesNotExist:
            return f'deleted article {id}'
```

### History Display Example

With custom formatting, foreign key changes show readable names:

```json
{
  "field": "author",
  "old_value": "John Doe (johndoe)",
  "new_value": "Jane Smith (janesmith)"
}
```

## Advanced History Features

### Exclude Fields from History

Exclude sensitive or irrelevant fields from history tracking:

```python
class User(BinderModel):
    username = models.CharField(max_length=150)
    email = models.EmailField()
    password = models.CharField(max_length=128)
    last_login = models.DateTimeField(null=True)
    
    class Binder:
        history = True
        history_exclude_fields = ['password', 'last_login']
```

### Custom History Serialization

Customize how field values are serialized in history:

```python
class Product(BinderModel):
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    metadata = models.JSONField(default=dict)
    
    class Binder:
        history = True
    
    def serialize_for_history(self, field_name, value):
        """Custom serialization for history"""
        if field_name == 'price':
            # Format price with currency
            return f"${value:.2f}" if value else None
        elif field_name == 'metadata':
            # Pretty print JSON
            return json.dumps(value, indent=2) if value else None
        
        return super().serialize_for_history(field_name, value)
```

### Bulk Operation History

Track bulk operations with context:

```python
from binder.history import ChangeSet

class ArticleView(ModelView):
    model = Article
    
    def bulk_publish(self, request):
        """Bulk publish articles with history context"""
        article_ids = request.data.get('article_ids', [])
        
        # Create a changeset for the bulk operation
        changeset = ChangeSet.objects.create(
            user=request.user,
            context={'operation': 'bulk_publish', 'count': len(article_ids)}
        )
        
        # Update articles
        articles = Article.objects.filter(id__in=article_ids)
        for article in articles:
            article.published = True
            article.published_at = timezone.now()
            article.save()  # History automatically tracked
        
        return JsonResponse({
            'success': True,
            'published_count': len(article_ids),
            'changeset_id': changeset.id
        })
```

## History Queries

### Query History Data

Query history data directly:

```python
from binder.models import ChangeSet, Change

# Get all changes for a specific object
article_changes = Change.objects.filter(
    model_name='Article',
    object_id=1
).select_related('changeset', 'changeset__user')

# Get changes by a specific user
user_changes = Change.objects.filter(
    changeset__user=user
).order_by('-changeset__timestamp')

# Get recent changes across all models
recent_changes = Change.objects.filter(
    changeset__timestamp__gte=timezone.now() - timedelta(days=7)
).select_related('changeset', 'changeset__user')

# Get changes for a specific field
title_changes = Change.objects.filter(
    model_name='Article',
    field='title'
).order_by('-changeset__timestamp')
```

### History Analytics

Create history analytics:

```python
from django.db.models import Count, Q
from binder.models import Change, ChangeSet

class HistoryAnalytics:
    @staticmethod
    def get_user_activity(user, days=30):
        """Get user activity statistics"""
        cutoff_date = timezone.now() - timedelta(days=days)
        
        changesets = ChangeSet.objects.filter(
            user=user,
            timestamp__gte=cutoff_date
        )
        
        return {
            'total_changes': changesets.count(),
            'models_modified': Change.objects.filter(
                changeset__in=changesets
            ).values('model_name').distinct().count(),
            'daily_activity': changesets.extra(
                select={'day': 'date(timestamp)'}
            ).values('day').annotate(count=Count('id')),
        }
    
    @staticmethod
    def get_model_activity(model_name, days=30):
        """Get activity statistics for a model"""
        cutoff_date = timezone.now() - timedelta(days=days)
        
        changes = Change.objects.filter(
            model_name=model_name,
            changeset__timestamp__gte=cutoff_date
        )
        
        return {
            'total_changes': changes.count(),
            'unique_objects': changes.values('object_id').distinct().count(),
            'field_changes': changes.values('field').annotate(
                count=Count('id')
            ).order_by('-count'),
            'top_contributors': changes.values(
                'changeset__user__username'
            ).annotate(count=Count('id')).order_by('-count')[:10],
        }
```

## History Management

### History Cleanup

Clean up old history data:

```python
# Management command: cleanup_history.py
from django.core.management.base import BaseCommand
from django.utils import timezone
from datetime import timedelta
from binder.models import ChangeSet

class Command(BaseCommand):
    help = 'Clean up old history data'
    
    def add_arguments(self, parser):
        parser.add_argument(
            '--days',
            type=int,
            default=365,
            help='Keep history for this many days'
        )
        parser.add_argument(
            '--dry-run',
            action='store_true',
            help='Show what would be deleted without actually deleting'
        )
    
    def handle(self, *args, **options):
        cutoff_date = timezone.now() - timedelta(days=options['days'])
        
        old_changesets = ChangeSet.objects.filter(
            timestamp__lt=cutoff_date
        )
        
        count = old_changesets.count()
        
        if options['dry_run']:
            self.stdout.write(f'Would delete {count} changesets older than {cutoff_date}')
        else:
            old_changesets.delete()
            self.stdout.write(f'Deleted {count} old changesets')
```

### History Archival

Archive old history data:

```python
class HistoryArchiver:
    def __init__(self, archive_days=365):
        self.archive_days = archive_days
    
    def archive_old_history(self):
        """Archive history older than specified days"""
        cutoff_date = timezone.now() - timedelta(days=self.archive_days)
        
        # Export to JSON before deletion
        old_changesets = ChangeSet.objects.filter(
            timestamp__lt=cutoff_date
        ).prefetch_related('change_set')
        
        archive_data = []
        for changeset in old_changesets:
            changeset_data = {
                'id': changeset.id,
                'user_id': changeset.user_id,
                'timestamp': changeset.timestamp.isoformat(),
                'changes': [
                    {
                        'model_name': change.model_name,
                        'object_id': change.object_id,
                        'field': change.field,
                        'old_value': change.old_value,
                        'new_value': change.new_value,
                    }
                    for change in changeset.change_set.all()
                ]
            }
            archive_data.append(changeset_data)
        
        # Save to archive file
        archive_file = f'history_archive_{cutoff_date.strftime("%Y%m%d")}.json'
        with open(archive_file, 'w') as f:
            json.dump(archive_data, f, indent=2)
        
        # Delete archived data
        old_changesets.delete()
        
        return len(archive_data)
```

## History Security

### Sensitive Data Handling

Handle sensitive data in history:

```python
class User(BinderModel):
    username = models.CharField(max_length=150)
    email = models.EmailField()
    phone = models.CharField(max_length=20, blank=True)
    
    class Binder:
        history = True
    
    def serialize_for_history(self, field_name, value):
        """Mask sensitive data in history"""
        if field_name == 'phone' and value:
            # Mask phone number
            return f"***-***-{value[-4:]}"
        elif field_name == 'email' and value:
            # Mask email
            username, domain = value.split('@')
            return f"{username[:2]}***@{domain}"
        
        return super().serialize_for_history(field_name, value)
```

### History Access Control

Control access to history data:

```python
class ArticleView(ModelView):
    model = Article
    
    def get_history(self, request, pk):
        """Custom history endpoint with access control"""
        article = self.get_object(pk)
        
        # Only allow history access to owners and staff
        if article.author != request.user and not request.user.is_staff:
            raise PermissionDenied("Cannot view history for this article")
        
        # Get history data
        changesets = ChangeSet.objects.filter(
            change__model_name='Article',
            change__object_id=pk
        ).distinct().order_by('-timestamp')
        
        # Serialize history data
        history_data = []
        for changeset in changesets:
            changes = changeset.change_set.filter(
                model_name='Article',
                object_id=pk
            )
            
            history_data.append({
                'timestamp': changeset.timestamp,
                'user': changeset.user.username if changeset.user else 'System',
                'changes': [
                    {
                        'field': change.field,
                        'old_value': change.old_value,
                        'new_value': change.new_value,
                    }
                    for change in changes
                ]
            })
        
        return JsonResponse({'history': history_data})
```

## Testing History

### History Tests

Test history tracking functionality:

```python
from django.test import TestCase
from django.contrib.auth.models import User
from binder.models import ChangeSet, Change

class HistoryTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        
    def test_history_creation(self):
        """Test that history is created on model save"""
        # Create article
        article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.user
        )
        
        # Check changeset was created
        changeset = ChangeSet.objects.filter(user=self.user).first()
        self.assertIsNotNone(changeset)
        
        # Check changes were recorded
        changes = Change.objects.filter(
            changeset=changeset,
            model_name='Article',
            object_id=article.id
        )
        
        # Should have changes for all fields (creation)
        self.assertTrue(changes.filter(field='title').exists())
        self.assertTrue(changes.filter(field='content').exists())
    
    def test_history_update(self):
        """Test history tracking on updates"""
        # Create article
        article = Article.objects.create(
            title='Original Title',
            content='Original content',
            author=self.user
        )
        
        # Clear existing history
        ChangeSet.objects.all().delete()
        
        # Update article
        article.title = 'Updated Title'
        article.published = True
        article.save()
        
        # Check update history
        changeset = ChangeSet.objects.filter(user=self.user).first()
        self.assertIsNotNone(changeset)
        
        changes = Change.objects.filter(changeset=changeset)
        
        # Should only have changes for modified fields
        title_change = changes.filter(field='title').first()
        self.assertEqual(title_change.old_value, 'Original Title')
        self.assertEqual(title_change.new_value, 'Updated Title')
        
        published_change = changes.filter(field='published').first()
        self.assertEqual(published_change.old_value, 'False')
        self.assertEqual(published_change.new_value, 'True')
    
    def test_history_api_endpoint(self):
        """Test history API endpoint"""
        # Create and update article
        article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.user
        )
        
        article.title = 'Updated Title'
        article.save()
        
        # Test history endpoint
        self.client.force_login(self.user)
        response = self.client.get(f'/api/article/{article.id}/history/')
        
        self.assertEqual(response.status_code, 200)
        history_data = response.json()
        
        # Should have history entries
        self.assertGreater(len(history_data['data']), 0)
        
        # Check history structure
        entry = history_data['data'][0]
        self.assertIn('timestamp', entry)
        self.assertIn('user', entry)
        self.assertIn('changes', entry)
```

This comprehensive history system provides complete audit trails for all model changes while maintaining performance and security.
