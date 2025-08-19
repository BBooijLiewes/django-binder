# Models & Fields

Django Binder extends Django's model system with powerful features designed for API development. This guide covers all aspects of working with models in Django Binder.

## BinderModel Base Class

All Django Binder models should inherit from `BinderModel` instead of Django's `Model`:

```python
from binder.models import BinderModel
from django.db import models

class Article(BinderModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

### Benefits of BinderModel

1. **Automatic API Integration**: Seamless integration with Django Binder views
2. **Enhanced Serialization**: Optimized JSON serialization
3. **History Tracking**: Optional change tracking and audit trails
4. **Soft Delete Support**: Built-in soft delete functionality
5. **Validation Enhancements**: Improved validation with detailed error reporting

## Enhanced Field Types

Django Binder provides several enhanced field types beyond Django's standard fields.

### Case-Sensitive Text Fields

```python
from binder.models import UpperCaseCharField, LowerCaseCharField

class Product(BinderModel):
    sku = UpperCaseCharField(max_length=20)  # Always stored in uppercase
    slug = LowerCaseCharField(max_length=100)  # Always stored in lowercase
```

### File and Image Fields

Enhanced file fields with additional metadata and features:

```python
from binder.models import BinderFileField, BinderImageField

class Document(BinderModel):
    title = models.CharField(max_length=200)
    file = BinderFileField(
        upload_to='documents/%Y/%m/',
        allowed_extensions=['pdf', 'doc', 'docx'],
        max_length=200
    )
    thumbnail = BinderImageField(
        upload_to='thumbnails/',
        blank=True,
        null=True,
        serve_directly=True  # Delegate to web server
    )
```

**BinderFileField Features:**
- **File Hash**: Automatic SHA1 hash generation for change detection
- **Content Type Detection**: Automatic MIME type detection
- **Filename Preservation**: Original filename is preserved
- **Extension Validation**: Restrict allowed file extensions
- **Direct Serving**: Option to delegate file serving to web server

**BinderImageField Additional Features:**
- **Image Validation**: Validates that uploaded files are valid images
- **Automatic Thumbnails**: Can generate thumbnails automatically
- **Format Conversion**: Convert between image formats

### Enum Fields

Type-safe enums with automatic choices generation:

```python
from binder.models import ChoiceEnum

class Order(BinderModel):
    STATUS = ChoiceEnum('pending', 'processing', 'shipped', 'delivered', 'cancelled')
    PRIORITY = ChoiceEnum(
        ('low', 'Low Priority'),
        ('normal', 'Normal Priority'), 
        ('high', 'High Priority'),
        ('urgent', 'Urgent')
    )
    
    status = models.CharField(max_length=20, choices=STATUS.choices())
    priority = models.CharField(max_length=10, choices=PRIORITY.choices(), default='normal')
    
    def __str__(self):
        return f"Order {self.id} - {self.STATUS.get_display(self.status)}"
```

### JSON Fields

Enhanced JSON field support with validation:

```python
from django.contrib.postgres.fields import JSONField
from django.core.exceptions import ValidationError

class Configuration(BinderModel):
    name = models.CharField(max_length=100)
    settings = JSONField(default=dict)
    metadata = JSONField(blank=True, null=True)
    
    def clean(self):
        # Validate JSON structure
        if 'required_key' not in self.settings:
            raise ValidationError({'settings': 'required_key is mandatory'})
```

### Array Fields (PostgreSQL)

Support for PostgreSQL array fields:

```python
from django.contrib.postgres.fields import ArrayField

class Tag(BinderModel):
    name = models.CharField(max_length=50)
    aliases = ArrayField(
        models.CharField(max_length=50),
        blank=True,
        default=list
    )
```

## Model Configuration

### Binder Meta Class

Configure Django Binder-specific options using the `Binder` inner class:

```python
class Article(BinderModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    
    class Binder:
        # Enable change tracking
        history = True
        
        # Custom serialization options
        serialize_fields = ['id', 'title', 'content', 'created_at']
        
        # Soft delete configuration
        soft_delete = True
        soft_delete_field = 'deleted_at'
```

### History Tracking

Enable comprehensive change tracking for your models:

```python
class User(BinderModel):
    username = models.CharField(max_length=150, unique=True)
    email = models.EmailField()
    first_name = models.CharField(max_length=30, blank=True)
    last_name = models.CharField(max_length=30, blank=True)
    
    class Binder:
        history = True
    
    @classmethod
    def format_instance_for_history(cls, id: int):
        """Custom display format for history entries"""
        try:
            user = cls.objects.get(id=id)
            return f"{user.username} ({user.email})"
        except cls.DoesNotExist:
            return f'deleted user {id}'
```

**History Features:**
- **Change Tracking**: All field changes are logged
- **User Attribution**: Changes are attributed to the requesting user
- **Timestamp Tracking**: Precise timestamps for all changes
- **Relationship Tracking**: Changes to foreign keys show readable names
- **API Access**: History is accessible via API endpoints

### Soft Delete

Implement soft delete functionality:

```python
from django.utils import timezone

class SoftDeleteModel(BinderModel):
    deleted_at = models.DateTimeField(null=True, blank=True)
    
    class Binder:
        soft_delete = True
        soft_delete_field = 'deleted_at'
    
    def delete(self, using=None, keep_parents=False):
        """Soft delete - mark as deleted instead of removing"""
        self.deleted_at = timezone.now()
        self.save(using=using)
    
    def hard_delete(self, using=None, keep_parents=False):
        """Actually delete the record"""
        super().delete(using=using, keep_parents=keep_parents)
    
    def restore(self):
        """Restore a soft-deleted record"""
        self.deleted_at = None
        self.save()
    
    class Meta:
        abstract = True

class Article(SoftDeleteModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
```

## Advanced Model Patterns

### Versioned Models

Implement model versioning:

```python
class VersionedModel(BinderModel):
    version = models.PositiveIntegerField(default=1)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def save(self, *args, **kwargs):
        if self.pk:  # Existing object
            self.version += 1
        super().save(*args, **kwargs)
    
    class Meta:
        abstract = True

class Document(VersionedModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
```

### Hierarchical Models

Implement tree-like structures:

```python
class Category(BinderModel):
    name = models.CharField(max_length=100)
    parent = models.ForeignKey(
        'self', 
        on_delete=models.CASCADE, 
        null=True, 
        blank=True,
        related_name='children'
    )
    level = models.PositiveIntegerField(default=0)
    
    def save(self, *args, **kwargs):
        if self.parent:
            self.level = self.parent.level + 1
        else:
            self.level = 0
        super().save(*args, **kwargs)
    
    def get_ancestors(self):
        """Get all parent categories"""
        ancestors = []
        current = self.parent
        while current:
            ancestors.append(current)
            current = current.parent
        return ancestors
    
    def get_descendants(self):
        """Get all child categories"""
        return Category.objects.filter(
            level__gt=self.level,
            id__in=self.get_descendant_ids()
        )
    
    class Meta:
        verbose_name_plural = "categories"
        ordering = ['level', 'name']
```

### Timestamped Models

Base class for models with automatic timestamps:

```python
class TimestampedModel(BinderModel):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True

class Article(TimestampedModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    # created_at and updated_at are automatically included
```

### Owned Models

Models with ownership tracking:

```python
from django.contrib.auth.models import User

class OwnedModel(BinderModel):
    owner = models.ForeignKey(User, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        abstract = True

class Article(OwnedModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    
    def save(self, *args, **kwargs):
        # Set owner on creation if not set
        if not self.pk and not self.owner_id:
            # Owner should be set by the view
            pass
        super().save(*args, **kwargs)
```

## Model Relationships

### Foreign Key Relationships

```python
class Author(BinderModel):
    name = models.CharField(max_length=100)
    email = models.EmailField()

class Category(BinderModel):
    name = models.CharField(max_length=50)

class Article(BinderModel):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(
        Author, 
        on_delete=models.CASCADE,
        related_name='articles'
    )
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='articles'
    )
```

### Many-to-Many Relationships

```python
class Tag(BinderModel):
    name = models.CharField(max_length=50, unique=True)

class Article(BinderModel):
    title = models.CharField(max_length=200)
    tags = models.ManyToManyField(
        Tag,
        blank=True,
        related_name='articles'
    )
```

### Through Models for Many-to-Many

```python
class Author(BinderModel):
    name = models.CharField(max_length=100)

class Book(BinderModel):
    title = models.CharField(max_length=200)
    authors = models.ManyToManyField(
        Author,
        through='BookAuthor',
        related_name='books'
    )

class BookAuthor(BinderModel):
    book = models.ForeignKey(Book, on_delete=models.CASCADE)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    role = models.CharField(max_length=50)  # e.g., "primary", "contributor"
    order = models.PositiveIntegerField(default=0)
    
    class Meta:
        unique_together = ['book', 'author']
        ordering = ['order']
```

## Model Validation

### Field-Level Validation

```python
from django.core.exceptions import ValidationError
from django.core.validators import RegexValidator

class Product(BinderModel):
    sku = models.CharField(
        max_length=20,
        validators=[
            RegexValidator(
                regex=r'^[A-Z]{2}-\d{4}-[A-Z]{2}$',
                message='SKU must be in format: XX-0000-XX'
            )
        ]
    )
    price = models.DecimalField(max_digits=10, decimal_places=2)
    
    def clean_price(self):
        """Field-specific validation"""
        if self.price <= 0:
            raise ValidationError('Price must be positive')
        return self.price
```

### Model-Level Validation

```python
class Event(BinderModel):
    name = models.CharField(max_length=200)
    start_date = models.DateTimeField()
    end_date = models.DateTimeField()
    max_attendees = models.PositiveIntegerField()
    
    def clean(self):
        """Model-level validation"""
        super().clean()
        
        if self.start_date and self.end_date:
            if self.start_date >= self.end_date:
                raise ValidationError({
                    'end_date': 'End date must be after start date'
                })
        
        if self.max_attendees and self.max_attendees > 1000:
            raise ValidationError({
                'max_attendees': 'Maximum attendees cannot exceed 1000'
            })
```

### Custom Validators

```python
from django.core.exceptions import ValidationError

def validate_file_size(value):
    """Validate file size is under 10MB"""
    if value.size > 10 * 1024 * 1024:  # 10MB
        raise ValidationError('File size cannot exceed 10MB')

def validate_image_dimensions(value):
    """Validate image dimensions"""
    from PIL import Image
    img = Image.open(value)
    if img.width > 2000 or img.height > 2000:
        raise ValidationError('Image dimensions cannot exceed 2000x2000 pixels')

class Photo(BinderModel):
    title = models.CharField(max_length=200)
    image = BinderImageField(
        upload_to='photos/',
        validators=[validate_file_size, validate_image_dimensions]
    )
```

## Model Properties and Methods

### Computed Properties

```python
class Order(BinderModel):
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)
    tax_rate = models.DecimalField(max_digits=5, decimal_places=4, default=0.0825)
    shipping = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    @property
    def tax_amount(self):
        """Calculate tax amount"""
        return self.subtotal * self.tax_rate
    
    @property
    def total(self):
        """Calculate total amount"""
        return self.subtotal + self.tax_amount + self.shipping
    
    @property
    def is_large_order(self):
        """Check if this is a large order"""
        return self.total > 1000
```

### Custom Managers

```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(published=True)

class Article(BinderModel):
    title = models.CharField(max_length=200)
    content = models.TextField()
    published = models.BooleanField(default=False)
    published_at = models.DateTimeField(null=True, blank=True)
    
    objects = models.Manager()  # Default manager
    published_objects = PublishedManager()  # Custom manager
    
    def publish(self):
        """Publish the article"""
        self.published = True
        self.published_at = timezone.now()
        self.save()
    
    def unpublish(self):
        """Unpublish the article"""
        self.published = False
        self.published_at = None
        self.save()
```

## Database Optimization

### Indexes

```python
class Article(BinderModel):
    title = models.CharField(max_length=200, db_index=True)
    slug = models.SlugField(unique=True)  # Automatically indexed
    content = models.TextField()
    published = models.BooleanField(default=False, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    
    class Meta:
        # Composite indexes
        indexes = [
            models.Index(fields=['published', '-created_at']),
            models.Index(fields=['author', 'published']),
        ]
        
        # Unique constraints
        unique_together = [['title', 'author']]
```

### Query Optimization

```python
class ArticleQuerySet(models.QuerySet):
    def published(self):
        return self.filter(published=True)
    
    def by_author(self, author):
        return self.filter(author=author)
    
    def with_related(self):
        return self.select_related('author', 'category').prefetch_related('tags')

class Article(BinderModel):
    title = models.CharField(max_length=200)
    author = models.ForeignKey('Author', on_delete=models.CASCADE)
    category = models.ForeignKey('Category', on_delete=models.CASCADE)
    tags = models.ManyToManyField('Tag')
    
    objects = ArticleQuerySet.as_manager()
    
    @classmethod
    def get_optimized_queryset(cls):
        """Get queryset optimized for API responses"""
        return cls.objects.with_related()
```

## Testing Models

### Model Tests

```python
from django.test import TestCase
from django.core.exceptions import ValidationError
from .models import Article, Author

class ArticleModelTest(TestCase):
    def setUp(self):
        self.author = Author.objects.create(
            name='John Doe',
            email='john@example.com'
        )
    
    def test_article_creation(self):
        """Test basic article creation"""
        article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.author
        )
        self.assertEqual(article.title, 'Test Article')
        self.assertEqual(article.author, self.author)
    
    def test_article_validation(self):
        """Test article validation"""
        with self.assertRaises(ValidationError):
            article = Article(
                title='',  # Empty title should fail
                content='Test content',
                author=self.author
            )
            article.full_clean()
    
    def test_article_properties(self):
        """Test computed properties"""
        article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.author,
            published=True
        )
        # Test any computed properties
        self.assertTrue(hasattr(article, 'is_published'))
```

## Migration Considerations

### Custom Migrations

```python
# migrations/0002_add_custom_field.py
from django.db import migrations, models

def populate_slug_field(apps, schema_editor):
    """Populate slug field for existing records"""
    Article = apps.get_model('myapp', 'Article')
    for article in Article.objects.all():
        article.slug = article.title.lower().replace(' ', '-')
        article.save()

class Migration(migrations.Migration):
    dependencies = [
        ('myapp', '0001_initial'),
    ]
    
    operations = [
        migrations.AddField(
            model_name='article',
            name='slug',
            field=models.SlugField(null=True),
        ),
        migrations.RunPython(populate_slug_field),
        migrations.AlterField(
            model_name='article',
            name='slug',
            field=models.SlugField(unique=True),
        ),
    ]
```

This comprehensive guide covers all aspects of working with models in Django Binder. The enhanced model system provides powerful features while maintaining compatibility with Django's ORM.
