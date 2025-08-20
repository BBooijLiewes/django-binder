# Multi-PUT Operations

Django Binder's Multi-PUT feature allows you to create and update multiple related objects in a single atomic transaction, making it ideal for complex forms and batch operations.

## Basic Multi-PUT

### Simple Multi-PUT Request

Create multiple objects in one request:

```bash
PUT /api/article/
Content-Type: application/json

{
  "data": [
    {
      "id": -1,
      "title": "First Article",
      "content": "Content for first article",
      "author": 1,
      "category": 1
    },
    {
      "id": -2,
      "title": "Second Article", 
      "content": "Content for second article",
      "author": 1,
      "category": 2
    }
  ]
}
```

**Key Points:**
- Use negative IDs for new objects (temporary IDs)
- All operations are atomic - if one fails, all fail
- Returns ID mapping for created objects

### Multi-PUT Response

```json
{
  "idmap": {
    "article": [
      [-1, 42],  // Temporary ID -1 became real ID 42
      [-2, 43]   // Temporary ID -2 became real ID 43
    ]
  },
  "data": [
    {
      "id": 42,
      "title": "First Article",
      "content": "Content for first article",
      "author": 1,
      "category": 1
    },
    {
      "id": 43,
      "title": "Second Article",
      "content": "Content for second article", 
      "author": 1,
      "category": 2
    }
  ]
}
```

## Related Object Creation

### Creating Objects with Relationships

Create objects and their related objects in one request:

```bash
PUT /api/article/
Content-Type: application/json

{
  "data": [
    {
      "id": -1,
      "title": "Article with New Author",
      "content": "Article content",
      "author": -1,     // Reference to new author
      "category": -1    // Reference to new category
    }
  ],
  "with": {
    "author": [
      {
        "id": -1,
        "username": "newauthor",
        "email": "newauthor@example.com",
        "first_name": "New",
        "last_name": "Author"
      }
    ],
    "category": [
      {
        "id": -1,
        "name": "New Category",
        "description": "A brand new category"
      }
    ]
  }
}
```

### Complex Relationship Example

Create a blog post with tags, comments, and author:

```bash
PUT /api/post/
Content-Type: application/json

{
  "data": [
    {
      "id": -1,
      "title": "Complete Blog Post",
      "content": "Post content here",
      "author": -1,
      "tags": [-1, -2],  // Multiple new tags
      "published": true
    }
  ],
  "with": {
    "author": [
      {
        "id": -1,
        "username": "blogger",
        "email": "blogger@example.com"
      }
    ],
    "tag": [
      {
        "id": -1,
        "name": "Django",
        "color": "#092e20"
      },
      {
        "id": -2,
        "name": "Python", 
        "color": "#3776ab"
      }
    ],
    "comment": [
      {
        "id": -1,
        "post": -1,  // Reference to the new post
        "author_name": "First Commenter",
        "content": "Great post!",
        "approved": true
      }
    ]
  }
}
```

## Update Operations

### Mixing Creates and Updates

Combine creating new objects with updating existing ones:

```bash
PUT /api/article/
Content-Type: application/json

{
  "data": [
    {
      "id": 5,  // Existing article - will be updated
      "title": "Updated Title",
      "content": "Updated content"
    },
    {
      "id": -1,  // New article - will be created
      "title": "New Article",
      "content": "New content",
      "author": 1,
      "category": 1
    }
  ]
}
```

### Partial Updates

Update only specific fields:

```bash
PUT /api/article/
Content-Type: application/json

{
  "data": [
    {
      "id": 5,
      "published": true,  // Only update published status
      "published_at": "2023-12-01T10:00:00Z"
    },
    {
      "id": 6,
      "title": "New Title"  // Only update title
    }
  ]
}
```

## Advanced Multi-PUT Patterns

### Hierarchical Object Creation

Create parent and child objects:

```bash
PUT /api/category/
Content-Type: application/json

{
  "data": [
    {
      "id": -1,
      "name": "Parent Category",
      "parent": null
    },
    {
      "id": -2,
      "name": "Child Category",
      "parent": -1  // Reference to parent being created
    },
    {
      "id": -3,
      "name": "Grandchild Category", 
      "parent": -2  // Reference to child being created
    }
  ]
}
```

### Bulk Data Import

Import large datasets efficiently:

```bash
PUT /api/product/
Content-Type: application/json

{
  "data": [
    {
      "id": -1,
      "sku": "PROD-001",
      "name": "Product 1",
      "price": "29.99",
      "category": -1
    },
    {
      "id": -2,
      "sku": "PROD-002", 
      "name": "Product 2",
      "price": "39.99",
      "category": -1
    }
    // ... more products
  ],
  "with": {
    "category": [
      {
        "id": -1,
        "name": "Electronics",
        "description": "Electronic products"
      }
    ]
  }
}
```

## Multi-PUT with Files

### File Upload in Multi-PUT

Handle file uploads in multi-PUT operations:

```bash
PUT /api/article/
Content-Type: multipart/form-data

data: {
  "data": [
    {
      "id": -1,
      "title": "Article with Image",
      "content": "Article content",
      "featured_image": null  // Will be replaced with uploaded file
    }
  ]
}
file:data.0.featured_image: [binary image data]
```

### Multiple Files

Upload multiple files for different objects:

```bash
PUT /api/product/
Content-Type: multipart/form-data

data: {
  "data": [
    {
      "id": -1,
      "name": "Product 1",
      "image": null
    },
    {
      "id": -2,
      "name": "Product 2", 
      "image": null
    }
  ]
}
file:data.0.image: [image1.jpg]
file:data.1.image: [image2.jpg]
```

## Error Handling

### Validation Errors

Handle validation errors in multi-PUT:

```json
// Error response
{
  "code": "ValidationError",
  "error": {
    "validation_errors": {
      "data": {
        "0": {  // First object in data array
          "title": [
            {
              "code": "This field is required."
            }
          ]
        },
        "1": {  // Second object in data array
          "email": [
            {
              "code": "Enter a valid email address."
            }
          ]
        }
      },
      "with": {
        "author": {
          "0": {  // First author in with.author array
            "username": [
              {
                "code": "This field must be unique."
              }
            ]
          }
        }
      }
    }
  }
}
```

### Transaction Rollback

All operations are atomic - if any object fails validation or saving, the entire transaction is rolled back:

```python
class ArticleView(ModelView):
    model = Article
    
    def store_multi(self, objects_data, request):
        """Custom multi-PUT logic with transaction handling"""
        try:
            with transaction.atomic():
                results = []
                for obj_data in objects_data:
                    # Custom validation
                    self.validate_multi_put_object(obj_data, request)
                    
                    # Process object
                    result = self.process_multi_put_object(obj_data, request)
                    results.append(result)
                
                return results
                
        except Exception as e:
            # Transaction automatically rolled back
            logger.error(f'Multi-PUT failed: {e}')
            raise
    
    def validate_multi_put_object(self, obj_data, request):
        """Custom validation for multi-PUT objects"""
        # Business logic validation
        if obj_data.get('published') and not obj_data.get('content'):
            raise ValidationError('Published articles must have content')
```

## Performance Optimization

### Batch Processing

Optimize multi-PUT for large datasets:

```python
class ArticleView(ModelView):
    model = Article
    
    def store_multi(self, objects_data, request):
        """Optimized multi-PUT processing"""
        
        # Separate creates and updates
        creates = [obj for obj in objects_data if obj.get('id', 0) < 0]
        updates = [obj for obj in objects_data if obj.get('id', 0) > 0]
        
        results = []
        
        with transaction.atomic():
            # Bulk create new objects
            if creates:
                new_objects = self.bulk_create_objects(creates, request)
                results.extend(new_objects)
            
            # Bulk update existing objects
            if updates:
                updated_objects = self.bulk_update_objects(updates, request)
                results.extend(updated_objects)
        
        return results
    
    def bulk_create_objects(self, creates_data, request):
        """Bulk create objects for better performance"""
        objects_to_create = []
        
        for obj_data in creates_data:
            obj = self.model(**obj_data)
            obj.full_clean()  # Validate
            objects_to_create.append(obj)
        
        # Bulk create
        created_objects = self.model.objects.bulk_create(objects_to_create)
        
        return created_objects
```

### Memory Management

Handle large multi-PUT requests efficiently:

```python
class ArticleView(ModelView):
    model = Article
    max_multi_put_objects = 1000  # Limit batch size
    
    def store_multi(self, objects_data, request):
        """Process large multi-PUT in chunks"""
        
        if len(objects_data) > self.max_multi_put_objects:
            raise ValidationError(
                f'Too many objects. Maximum {self.max_multi_put_objects} allowed.'
            )
        
        # Process in chunks for memory efficiency
        chunk_size = 100
        all_results = []
        
        for i in range(0, len(objects_data), chunk_size):
            chunk = objects_data[i:i + chunk_size]
            chunk_results = self.process_chunk(chunk, request)
            all_results.extend(chunk_results)
        
        return all_results
```

## Custom Multi-PUT Logic

### Override Multi-PUT Behavior

Customize multi-PUT processing:

```python
class OrderView(ModelView):
    model = Order
    
    def store_multi(self, objects_data, request):
        """Custom multi-PUT logic for orders"""
        
        with transaction.atomic():
            results = []
            total_amount = 0
            
            for obj_data in objects_data:
                # Custom business logic
                if obj_data.get('status') == 'completed':
                    obj_data['completed_at'] = timezone.now()
                
                # Calculate running total
                total_amount += obj_data.get('amount', 0)
                
                # Process object
                result = super().store_object(obj_data, request)
                results.append(result)
            
            # Apply bulk discount if total is high
            if total_amount > 1000:
                self.apply_bulk_discount(results)
            
            return results
    
    def apply_bulk_discount(self, orders):
        """Apply discount to bulk orders"""
        for order in orders:
            order.discount_applied = True
            order.save()
```

### Multi-PUT Hooks

Add hooks for multi-PUT operations:

```python
class ArticleView(ModelView):
    model = Article
    
    def pre_multi_put(self, objects_data, request):
        """Called before multi-PUT processing"""
        # Log the operation
        logger.info(f'Multi-PUT started: {len(objects_data)} objects by user {request.user.id}')
        
        # Custom validation
        self.validate_multi_put_permissions(objects_data, request)
    
    def post_multi_put(self, results, request):
        """Called after successful multi-PUT processing"""
        # Send notifications
        self.send_multi_put_notifications(results, request)
        
        # Update search index
        self.update_search_index(results)
        
        # Log completion
        logger.info(f'Multi-PUT completed: {len(results)} objects processed')
    
    def validate_multi_put_permissions(self, objects_data, request):
        """Validate permissions for all objects in multi-PUT"""
        for obj_data in objects_data:
            if obj_data.get('published') and not request.user.has_perm('article.publish'):
                raise PermissionDenied('No permission to publish articles')
```

## Testing Multi-PUT

### Multi-PUT Tests

Test multi-PUT functionality:

```python
from django.test import TestCase
from django.contrib.auth.models import User

class MultiPutTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        self.client.force_login(self.user)
    
    def test_multi_put_create(self):
        """Test creating multiple objects"""
        data = {
            "data": [
                {
                    "id": -1,
                    "title": "Article 1",
                    "content": "Content 1",
                    "author": self.user.id
                },
                {
                    "id": -2,
                    "title": "Article 2",
                    "content": "Content 2", 
                    "author": self.user.id
                }
            ]
        }
        
        response = self.client.put('/api/article/', data, content_type='application/json')
        self.assertEqual(response.status_code, 200)
        
        # Check objects were created
        self.assertEqual(Article.objects.count(), 2)
        
        # Check ID mapping
        response_data = response.json()
        self.assertIn('idmap', response_data)
        self.assertEqual(len(response_data['idmap']['article']), 2)
    
    def test_multi_put_with_relations(self):
        """Test multi-PUT with related objects"""
        data = {
            "data": [
                {
                    "id": -1,
                    "title": "Article with New Category",
                    "content": "Content",
                    "author": self.user.id,
                    "category": -1
                }
            ],
            "with": {
                "category": [
                    {
                        "id": -1,
                        "name": "New Category",
                        "description": "A new category"
                    }
                ]
            }
        }
        
        response = self.client.put('/api/article/', data, content_type='application/json')
        self.assertEqual(response.status_code, 200)
        
        # Check both objects were created
        self.assertEqual(Article.objects.count(), 1)
        self.assertEqual(Category.objects.count(), 1)
        
        # Check relationship
        article = Article.objects.first()
        self.assertEqual(article.category.name, "New Category")
```

Multi-PUT operations provide powerful batch processing capabilities while maintaining data integrity through atomic transactions.
