# Testing Guide

This comprehensive guide covers testing strategies for Django Binder applications, from unit tests to integration tests and performance testing.

## Test Setup

### Test Configuration

Configure your test environment:

```python
# settings/test.py
from .base import *

# Test database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': ':memory:',
    }
}

# Disable migrations for faster tests
class DisableMigrations:
    def __contains__(self, item):
        return True
    
    def __getitem__(self, item):
        return None

MIGRATION_MODULES = DisableMigrations()

# Test-specific settings
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.MD5PasswordHasher',  # Faster for tests
]

# Disable caching in tests
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    }
}

# Test logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'loggers': {
        'binder': {
            'handlers': ['console'],
            'level': 'DEBUG',
        },
    },
}
```

### Test Base Classes

Create base test classes for common functionality:

```python
# tests/base.py
from django.test import TestCase, TransactionTestCase
from django.contrib.auth.models import User
from rest_framework.test import APIClient
import json

class BinderTestCase(TestCase):
    """Base test case for Django Binder tests"""
    
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.admin = User.objects.create_superuser(
            username='admin',
            email='admin@example.com',
            password='adminpass123'
        )
    
    def login_user(self, user=None):
        """Login a user for authenticated requests"""
        user = user or self.user
        self.client.force_authenticate(user=user)
    
    def login_admin(self):
        """Login admin user"""
        self.client.force_authenticate(user=self.admin)
    
    def logout(self):
        """Logout current user"""
        self.client.force_authenticate(user=None)
    
    def assertValidationError(self, response, field=None):
        """Assert response contains validation errors"""
        self.assertEqual(response.status_code, 400)
        data = response.json()
        self.assertEqual(data['code'], 'ValidationError')
        
        if field:
            self.assertIn('validation_errors', data['error'])
            self.assertIn(field, data['error']['validation_errors'])
    
    def assertPermissionDenied(self, response):
        """Assert response is permission denied"""
        self.assertEqual(response.status_code, 403)
    
    def assertNotFound(self, response):
        """Assert response is not found"""
        self.assertEqual(response.status_code, 404)

class BinderTransactionTestCase(TransactionTestCase):
    """Base transaction test case for testing database transactions"""
    
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
```

## Model Testing

### Model Unit Tests

Test your Django Binder models:

```python
# tests/test_models.py
from django.test import TestCase
from django.core.exceptions import ValidationError
from django.contrib.auth.models import User
from myapp.models import Article, Category

class ArticleModelTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.category = Category.objects.create(
            name='Technology',
            description='Tech articles'
        )
    
    def test_article_creation(self):
        """Test basic article creation"""
        article = Article.objects.create(
            title='Test Article',
            content='This is test content',
            author=self.user,
            category=self.category
        )
        
        self.assertEqual(article.title, 'Test Article')
        self.assertEqual(article.author, self.user)
        self.assertEqual(article.category, self.category)
        self.assertFalse(article.published)  # Default value
        self.assertIsNotNone(article.created_at)
    
    def test_article_str_representation(self):
        """Test string representation"""
        article = Article.objects.create(
            title='Test Article',
            content='Content',
            author=self.user,
            category=self.category
        )
        
        self.assertEqual(str(article), 'Test Article')
    
    def test_article_validation(self):
        """Test model validation"""
        # Test required fields
        with self.assertRaises(ValidationError):
            article = Article(
                content='Content without title',
                author=self.user,
                category=self.category
            )
            article.full_clean()
        
        # Test field length validation
        with self.assertRaises(ValidationError):
            article = Article(
                title='x' * 201,  # Exceeds max_length
                content='Content',
                author=self.user,
                category=self.category
            )
            article.full_clean()
    
    def test_article_properties(self):
        """Test computed properties"""
        article = Article.objects.create(
            title='Test Article',
            content='This is a test article with multiple words for testing',
            author=self.user,
            category=self.category
        )
        
        # Test word count property (if implemented)
        if hasattr(article, 'word_count'):
            self.assertGreater(article.word_count, 0)
    
    def test_article_methods(self):
        """Test custom model methods"""
        article = Article.objects.create(
            title='Test Article',
            content='Content',
            author=self.user,
            category=self.category
        )
        
        # Test publish method (if implemented)
        if hasattr(article, 'publish'):
            article.publish()
            self.assertTrue(article.published)
            self.assertIsNotNone(article.published_at)
    
    def test_article_relationships(self):
        """Test model relationships"""
        article = Article.objects.create(
            title='Test Article',
            content='Content',
            author=self.user,
            category=self.category
        )
        
        # Test reverse relationships
        self.assertIn(article, self.user.articles.all())
        self.assertIn(article, self.category.articles.all())
    
    def test_article_history_tracking(self):
        """Test history tracking (if enabled)"""
        article = Article.objects.create(
            title='Original Title',
            content='Original content',
            author=self.user,
            category=self.category
        )
        
        # Update article
        article.title = 'Updated Title'
        article.save()
        
        # Check if history is tracked
        if hasattr(article, 'history'):
            history = article.history.all()
            self.assertGreater(len(history), 0)
```

### Model Manager Tests

Test custom model managers:

```python
class ArticleManagerTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        self.category = Category.objects.create(name='Tech')
        
        # Create test articles
        self.published_article = Article.objects.create(
            title='Published Article',
            content='Content',
            author=self.user,
            category=self.category,
            published=True
        )
        
        self.draft_article = Article.objects.create(
            title='Draft Article',
            content='Content',
            author=self.user,
            category=self.category,
            published=False
        )
    
    def test_published_manager(self):
        """Test published articles manager"""
        if hasattr(Article, 'published_objects'):
            published_articles = Article.published_objects.all()
            self.assertIn(self.published_article, published_articles)
            self.assertNotIn(self.draft_article, published_articles)
    
    def test_custom_queryset_methods(self):
        """Test custom queryset methods"""
        if hasattr(Article.objects, 'by_author'):
            user_articles = Article.objects.by_author(self.user)
            self.assertEqual(user_articles.count(), 2)
        
        if hasattr(Article.objects, 'published'):
            published_articles = Article.objects.published()
            self.assertEqual(published_articles.count(), 1)
```

## View Testing

### API Endpoint Tests

Test your Django Binder API endpoints:

```python
# tests/test_views.py
from tests.base import BinderTestCase
from myapp.models import Article, Category
import json

class ArticleViewTest(BinderTestCase):
    def setUp(self):
        super().setUp()
        self.category = Category.objects.create(
            name='Technology',
            description='Tech articles'
        )
        self.article = Article.objects.create(
            title='Test Article',
            content='Test content',
            author=self.user,
            category=self.category,
            published=True
        )
    
    def test_list_articles(self):
        """Test article list endpoint"""
        response = self.client.get('/api/article/')
        
        self.assertEqual(response.status_code, 200)
        data = response.json()
        
        self.assertIn('data', data)
        self.assertEqual(len(data['data']), 1)
        self.assertEqual(data['data'][0]['title'], 'Test Article')
    
    def test_get_article(self):
        """Test get single article"""
        response = self.client.get(f'/api/article/{self.article.id}/')
        
        self.assertEqual(response.status_code, 200)
        data = response.json()
        
        self.assertEqual(data['id'], self.article.id)
        self.assertEqual(data['title'], 'Test Article')
        self.assertEqual(data['author'], self.user.id)
    
    def test_create_article(self):
        """Test article creation"""
        self.login_user()
        
        article_data = {
            'title': 'New Article',
            'content': 'New article content',
            'category': self.category.id,
            'published': False
        }
        
        response = self.client.post(
            '/api/article/',
            json.dumps(article_data),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, 200)
        data = response.json()
        
        self.assertEqual(data['title'], 'New Article')
        self.assertEqual(data['author'], self.user.id)
        
        # Verify article was created in database
        article = Article.objects.get(id=data['id'])
        self.assertEqual(article.title, 'New Article')
    
    def test_update_article(self):
        """Test article update"""
        self.login_user()
        
        update_data = {
            'title': 'Updated Title',
            'published': True
        }
        
        response = self.client.put(
            f'/api/article/{self.article.id}/',
            json.dumps(update_data),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, 200)
        data = response.json()
        
        self.assertEqual(data['title'], 'Updated Title')
        self.assertTrue(data['published'])
        
        # Verify update in database
        self.article.refresh_from_db()
        self.assertEqual(self.article.title, 'Updated Title')
    
    def test_delete_article(self):
        """Test article deletion"""
        self.login_user()
        
        response = self.client.delete(f'/api/article/{self.article.id}/')
        
        self.assertEqual(response.status_code, 200)
        
        # Verify article is deleted (or soft deleted)
        with self.assertRaises(Article.DoesNotExist):
            Article.objects.get(id=self.article.id)
    
    def test_article_filtering(self):
        """Test article filtering"""
        # Create additional test data
        Article.objects.create(
            title='Another Article',
            content='Different content',
            author=self.user,
            category=self.category,
            published=False
        )
        
        # Test published filter
        response = self.client.get('/api/article/?published=true')
        data = response.json()
        self.assertEqual(len(data['data']), 1)
        
        # Test title search
        response = self.client.get('/api/article/?search=Test')
        data = response.json()
        self.assertEqual(len(data['data']), 1)
        self.assertEqual(data['data'][0]['title'], 'Test Article')
    
    def test_article_ordering(self):
        """Test article ordering"""
        # Create additional article
        Article.objects.create(
            title='Older Article',
            content='Content',
            author=self.user,
            category=self.category,
            published=True
        )
        
        # Test ordering by title
        response = self.client.get('/api/article/?order_by=title')
        data = response.json()
        
        titles = [article['title'] for article in data['data']]
        self.assertEqual(titles, sorted(titles))
        
        # Test reverse ordering
        response = self.client.get('/api/article/?order_by=-title')
        data = response.json()
        
        titles = [article['title'] for article in data['data']]
        self.assertEqual(titles, sorted(titles, reverse=True))
    
    def test_article_pagination(self):
        """Test article pagination"""
        # Create multiple articles
        for i in range(25):
            Article.objects.create(
                title=f'Article {i}',
                content='Content',
                author=self.user,
                category=self.category
            )
        
        # Test default pagination
        response = self.client.get('/api/article/')
        data = response.json()
        
        self.assertIn('meta', data)
        self.assertIn('total_records', data['meta'])
        self.assertEqual(data['meta']['total_records'], 26)  # 25 + original
        
        # Test custom limit
        response = self.client.get('/api/article/?limit=5')
        data = response.json()
        self.assertEqual(len(data['data']), 5)
        
        # Test offset
        response = self.client.get('/api/article/?limit=5&offset=5')
        data = response.json()
        self.assertEqual(len(data['data']), 5)
```

### Permission Tests

Test permission and security:

```python
class ArticlePermissionTest(BinderTestCase):
    def setUp(self):
        super().setUp()
        self.category = Category.objects.create(name='Tech')
        self.other_user = User.objects.create_user(
            'otheruser', 'other@example.com', 'password'
        )
        
        self.user_article = Article.objects.create(
            title='User Article',
            content='Content',
            author=self.user,
            category=self.category
        )
        
        self.other_article = Article.objects.create(
            title='Other Article',
            content='Content',
            author=self.other_user,
            category=self.category
        )
    
    def test_anonymous_access(self):
        """Test anonymous user access"""
        # Anonymous users should be able to read published articles
        response = self.client.get('/api/article/')
        self.assertEqual(response.status_code, 200)
        
        # But not create articles
        response = self.client.post('/api/article/', {
            'title': 'New Article',
            'content': 'Content'
        })
        self.assertIn(response.status_code, [401, 403])
    
    def test_user_can_edit_own_articles(self):
        """Test users can edit their own articles"""
        self.login_user()
        
        response = self.client.put(
            f'/api/article/{self.user_article.id}/',
            json.dumps({'title': 'Updated Title'}),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, 200)
    
    def test_user_cannot_edit_others_articles(self):
        """Test users cannot edit others' articles"""
        self.login_user()
        
        response = self.client.put(
            f'/api/article/{self.other_article.id}/',
            json.dumps({'title': 'Hacked Title'}),
            content_type='application/json'
        )
        
        self.assertPermissionDenied(response)
    
    def test_admin_can_edit_all_articles(self):
        """Test admin can edit all articles"""
        self.login_admin()
        
        response = self.client.put(
            f'/api/article/{self.other_article.id}/',
            json.dumps({'title': 'Admin Updated'}),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, 200)
```

### Custom Endpoint Tests

Test custom endpoints:

```python
class ArticleCustomEndpointTest(BinderTestCase):
    def setUp(self):
        super().setUp()
        self.category = Category.objects.create(name='Tech')
        self.article = Article.objects.create(
            title='Test Article',
            content='Content',
            author=self.user,
            category=self.category,
            published=False
        )
    
    def test_publish_endpoint(self):
        """Test custom publish endpoint"""
        self.login_user()
        
        response = self.client.post(f'/api/article/{self.article.id}/publish/')
        
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertTrue(data['success'])
        
        # Verify article is published
        self.article.refresh_from_db()
        self.assertTrue(self.article.published)
    
    def test_stats_endpoint(self):
        """Test custom stats endpoint"""
        response = self.client.get('/api/article/stats/')
        
        self.assertEqual(response.status_code, 200)
        data = response.json()
        
        self.assertIn('total', data)
        self.assertIn('published', data)
        self.assertIn('draft', data)
    
    def test_search_endpoint(self):
        """Test custom search endpoint"""
        response = self.client.get('/api/article/search/test/')
        
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertIn('data', data)
```

## Integration Tests

### Multi-PUT Tests

Test multi-PUT operations:

```python
class MultiPutTest(BinderTestCase):
    def setUp(self):
        super().setUp()
        self.login_user()
    
    def test_multi_put_create(self):
        """Test creating multiple objects with multi-PUT"""
        data = {
            "data": [
                {
                    "id": -1,
                    "title": "Article 1",
                    "content": "Content 1",
                    "category": -1
                },
                {
                    "id": -2,
                    "title": "Article 2",
                    "content": "Content 2",
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
        
        response = self.client.put(
            '/api/article/',
            json.dumps(data),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, 200)
        response_data = response.json()
        
        # Check ID mapping
        self.assertIn('idmap', response_data)
        self.assertIn('article', response_data['idmap'])
        self.assertIn('category', response_data['idmap'])
        
        # Verify objects were created
        self.assertEqual(Article.objects.count(), 2)
        self.assertEqual(Category.objects.count(), 1)
    
    def test_multi_put_validation_error(self):
        """Test multi-PUT with validation errors"""
        data = {
            "data": [
                {
                    "id": -1,
                    "title": "",  # Invalid empty title
                    "content": "Content",
                    "category": 1
                }
            ]
        }
        
        response = self.client.put(
            '/api/article/',
            json.dumps(data),
            content_type='application/json'
        )
        
        self.assertValidationError(response)
        
        # Verify no objects were created (atomic transaction)
        self.assertEqual(Article.objects.count(), 0)
```

### File Upload Tests

Test file upload functionality:

```python
from django.core.files.uploadedfile import SimpleUploadedFile

class FileUploadTest(BinderTestCase):
    def setUp(self):
        super().setUp()
        self.login_user()
    
    def test_image_upload(self):
        """Test image file upload"""
        # Create a simple test image
        from PIL import Image
        import io
        
        img = Image.new('RGB', (100, 100), color='red')
        img_io = io.BytesIO()
        img.save(img_io, format='JPEG')
        img_io.seek(0)
        
        test_image = SimpleUploadedFile(
            "test.jpg",
            img_io.read(),
            content_type="image/jpeg"
        )
        
        # Create article with image
        response = self.client.post('/api/article/', {
            'title': 'Article with Image',
            'content': 'Content',
            'featured_image': test_image
        })
        
        self.assertEqual(response.status_code, 200)
        data = response.json()
        
        # Verify image was uploaded
        article = Article.objects.get(id=data['id'])
        self.assertTrue(article.featured_image)
    
    def test_file_upload_validation(self):
        """Test file upload validation"""
        # Test invalid file type
        invalid_file = SimpleUploadedFile(
            "test.exe",
            b"invalid content",
            content_type="application/x-executable"
        )
        
        response = self.client.post('/api/article/', {
            'title': 'Article with Invalid File',
            'content': 'Content',
            'featured_image': invalid_file
        })
        
        self.assertValidationError(response)
```

## Performance Tests

### Load Testing

Test API performance under load:

```python
import time
import threading
from django.test import TestCase
from django.test.utils import override_settings

class PerformanceTest(TestCase):
    def setUp(self):
        # Create test data
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        self.category = Category.objects.create(name='Tech')
        
        # Create multiple articles for testing
        for i in range(100):
            Article.objects.create(
                title=f'Article {i}',
                content=f'Content for article {i}',
                author=self.user,
                category=self.category
            )
    
    def test_list_performance(self):
        """Test list endpoint performance"""
        start_time = time.time()
        
        response = self.client.get('/api/article/')
        
        end_time = time.time()
        response_time = end_time - start_time
        
        self.assertEqual(response.status_code, 200)
        self.assertLess(response_time, 1.0)  # Should respond within 1 second
    
    def test_concurrent_requests(self):
        """Test concurrent request handling"""
        results = []
        
        def make_request():
            start_time = time.time()
            response = self.client.get('/api/article/')
            end_time = time.time()
            
            results.append({
                'status_code': response.status_code,
                'response_time': end_time - start_time
            })
        
        # Create multiple threads
        threads = []
        for i in range(10):
            thread = threading.Thread(target=make_request)
            threads.append(thread)
        
        # Start all threads
        for thread in threads:
            thread.start()
        
        # Wait for all threads to complete
        for thread in threads:
            thread.join()
        
        # Verify all requests succeeded
        self.assertEqual(len(results), 10)
        for result in results:
            self.assertEqual(result['status_code'], 200)
            self.assertLess(result['response_time'], 2.0)
    
    @override_settings(DEBUG=True)
    def test_query_count(self):
        """Test database query count"""
        from django.db import connection
        
        # Reset query log
        connection.queries_log.clear()
        
        response = self.client.get('/api/article/')
        
        query_count = len(connection.queries)
        
        self.assertEqual(response.status_code, 200)
        self.assertLess(query_count, 10)  # Should use reasonable number of queries
```

## Test Utilities

### Test Data Factories

Create test data factories for consistent test data:

```python
# tests/factories.py
import factory
from django.contrib.auth.models import User
from myapp.models import Article, Category

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User
    
    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@example.com')
    first_name = factory.Faker('first_name')
    last_name = factory.Faker('last_name')

class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category
    
    name = factory.Faker('word')
    description = factory.Faker('text')

class ArticleFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Article
    
    title = factory.Faker('sentence', nb_words=4)
    content = factory.Faker('text')
    author = factory.SubFactory(UserFactory)
    category = factory.SubFactory(CategoryFactory)
    published = factory.Faker('boolean')

# Usage in tests
class ArticleTestWithFactory(TestCase):
    def test_article_creation(self):
        article = ArticleFactory()
        self.assertIsNotNone(article.id)
        self.assertIsNotNone(article.title)
    
    def test_multiple_articles(self):
        articles = ArticleFactory.create_batch(5)
        self.assertEqual(len(articles), 5)
```

### Mock External Services

Mock external services for isolated testing:

```python
from unittest.mock import patch, Mock

class ExternalServiceTest(TestCase):
    @patch('myapp.services.external_api_client')
    def test_external_api_integration(self, mock_client):
        """Test integration with external API"""
        # Mock the external API response
        mock_response = Mock()
        mock_response.json.return_value = {'status': 'success', 'data': 'test'}
        mock_client.get.return_value = mock_response
        
        # Test your code that uses the external API
        from myapp.services import get_external_data
        result = get_external_data('test_id')
        
        # Verify the mock was called correctly
        mock_client.get.assert_called_once_with('/api/data/test_id')
        self.assertEqual(result['status'], 'success')
```

## Test Coverage

### Coverage Configuration

Configure test coverage:

```bash
# Install coverage
pip install coverage

# Run tests with coverage
coverage run --source='.' manage.py test
coverage report
coverage html  # Generate HTML report
```

### Coverage Settings

```python
# .coveragerc
[run]
source = .
omit = 
    */venv/*
    */migrations/*
    */tests/*
    manage.py
    */settings/*
    */wsgi.py
    */asgi.py

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
```

## Continuous Integration

### GitHub Actions

Configure CI/CD with GitHub Actions:

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: 3.9
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install coverage
    
    - name: Run tests
      run: |
        coverage run --source='.' manage.py test
        coverage report --fail-under=80
      env:
        DATABASE_URL: postgres://postgres:postgres@localhost:5432/test_db
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v1
```

This comprehensive testing guide ensures your Django Binder application is thoroughly tested and reliable in production.
