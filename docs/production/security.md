# Security Best Practices

This guide covers comprehensive security practices for Django Binder applications, from authentication and authorization to data protection and vulnerability prevention.

## Authentication Security

### Token-Based Authentication

Implement secure token authentication:

```python
# settings.py
INSTALLED_APPS += ['binder.plugins.token_auth']

# Token security settings
TOKEN_EXPIRY_HOURS = 24  # Tokens expire after 24 hours
TOKEN_CLEANUP_DAYS = 7   # Clean up expired tokens after 7 days
TOKEN_MAX_PER_USER = 5   # Maximum tokens per user

# Secure token generation
import secrets
TOKEN_KEY_LENGTH = 40  # Use longer tokens for better security
```

### Token Management

Secure token lifecycle management:

```python
from binder.plugins.token_auth.models import Token
from django.utils import timezone
from datetime import timedelta

class SecureTokenManager:
    @staticmethod
    def create_token(user, device_info=None):
        """Create secure token with metadata"""
        # Revoke old tokens if limit exceeded
        user_tokens = Token.objects.filter(user=user).order_by('-created')
        if user_tokens.count() >= settings.TOKEN_MAX_PER_USER:
            # Remove oldest tokens
            old_tokens = user_tokens[settings.TOKEN_MAX_PER_USER-1:]
            Token.objects.filter(id__in=[t.id for t in old_tokens]).delete()
        
        # Create new token
        token = Token.objects.create(
            user=user,
            device_info=device_info or {},
            expires_at=timezone.now() + timedelta(hours=settings.TOKEN_EXPIRY_HOURS)
        )
        
        return token
    
    @staticmethod
    def revoke_token(token_key):
        """Securely revoke token"""
        try:
            token = Token.objects.get(key=token_key)
            token.delete()
            return True
        except Token.DoesNotExist:
            return False
    
    @staticmethod
    def cleanup_expired_tokens():
        """Clean up expired tokens"""
        expired_tokens = Token.objects.filter(
            expires_at__lt=timezone.now()
        )
        count = expired_tokens.count()
        expired_tokens.delete()
        return count
```

### Session Security

Secure session configuration:

```python
# settings.py
# Session security
SESSION_COOKIE_SECURE = True  # HTTPS only
SESSION_COOKIE_HTTPONLY = True  # No JavaScript access
SESSION_COOKIE_SAMESITE = 'Strict'  # CSRF protection
SESSION_COOKIE_AGE = 3600  # 1 hour session timeout

# CSRF protection
CSRF_COOKIE_SECURE = True
CSRF_COOKIE_HTTPONLY = True
CSRF_COOKIE_SAMESITE = 'Strict'
CSRF_TRUSTED_ORIGINS = ['https://yourdomain.com']

# Additional security headers
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
X_FRAME_OPTIONS = 'DENY'
```

## Authorization Security

### Permission Validation

Implement robust permission checking:

```python
from binder.permissions.views import PermissionView
from binder.exceptions import BinderForbidden
import logging

security_logger = logging.getLogger('security')

class SecurePermissionView(PermissionView):
    def _require_model_perm(self, perm_type, request, pk=None):
        """Enhanced permission checking with logging"""
        try:
            # Log permission check attempt
            security_logger.info(
                f'Permission check: user={request.user.id}, '
                f'perm={perm_type}, model={self.model.__name__}, pk={pk}'
            )
            
            scopes = super()._require_model_perm(perm_type, request, pk)
            
            # Log successful permission grant
            security_logger.info(
                f'Permission granted: user={request.user.id}, '
                f'perm={perm_type}, scopes={scopes}'
            )
            
            return scopes
            
        except BinderForbidden as e:
            # Log permission denial
            security_logger.warning(
                f'Permission denied: user={request.user.id}, '
                f'perm={perm_type}, model={self.model.__name__}, '
                f'reason={str(e)}'
            )
            raise
    
    def validate_object_access(self, obj, request, action):
        """Additional object-level security validation"""
        # Check if object is soft-deleted
        if hasattr(obj, 'deleted_at') and obj.deleted_at:
            if not request.user.is_staff:
                raise BinderForbidden("Cannot access deleted objects")
        
        # Check object ownership for sensitive actions
        if action in ['change', 'delete']:
            if hasattr(obj, 'owner') and obj.owner != request.user:
                if not request.user.is_staff:
                    raise BinderForbidden("Can only modify own objects")
        
        # Check business rules
        if hasattr(obj, 'is_locked') and obj.is_locked:
            if not request.user.has_perm(f'{obj._meta.app_label}.unlock_{obj._meta.model_name}'):
                raise BinderForbidden("Object is locked")

class ArticleView(SecurePermissionView):
    model = Article
```

### Data Access Control

Implement fine-grained data access control:

```python
class DataAccessControlMixin:
    def get_queryset(self, request):
        """Apply data access controls"""
        queryset = super().get_queryset(request)
        
        # Apply user-based filtering
        if not request.user.is_staff:
            queryset = self.apply_user_data_filter(queryset, request.user)
        
        # Apply department-based filtering
        if hasattr(request.user, 'profile') and request.user.profile.department:
            queryset = self.apply_department_filter(queryset, request.user.profile.department)
        
        # Apply security level filtering
        user_security_level = self.get_user_security_level(request.user)
        queryset = queryset.filter(security_level__lte=user_security_level)
        
        return queryset
    
    def apply_user_data_filter(self, queryset, user):
        """Filter data based on user access rights"""
        # Users can see their own data and public data
        return queryset.filter(
            Q(owner=user) | Q(visibility='public')
        )
    
    def apply_department_filter(self, queryset, department):
        """Filter data based on department access"""
        return queryset.filter(
            Q(department=department) | Q(shared_with_departments=department)
        )
    
    def get_user_security_level(self, user):
        """Get user's security clearance level"""
        if user.is_superuser:
            return 10  # Highest level
        elif user.is_staff:
            return 5   # Medium level
        else:
            return 1   # Basic level

class SecureArticleView(DataAccessControlMixin, SecurePermissionView):
    model = Article
```

## Input Validation Security

### SQL Injection Prevention

Prevent SQL injection attacks:

```python
from django.db.models import Q
from django.core.exceptions import ValidationError
import re

class SQLInjectionProtectionMixin:
    # Dangerous SQL keywords to detect
    SQL_INJECTION_PATTERNS = [
        r'\b(union|select|insert|update|delete|drop|create|alter)\b',
        r'[\'";]',  # SQL injection characters
        r'--',      # SQL comments
        r'/\*.*\*/', # SQL block comments
    ]
    
    def validate_query_parameters(self, request):
        """Validate query parameters for SQL injection"""
        for key, value in request.GET.items():
            if isinstance(value, str):
                self.check_sql_injection(value, f"Query parameter '{key}'")
    
    def check_sql_injection(self, value, context="Input"):
        """Check for SQL injection patterns"""
        value_lower = value.lower()
        
        for pattern in self.SQL_INJECTION_PATTERNS:
            if re.search(pattern, value_lower, re.IGNORECASE):
                security_logger.warning(
                    f'Potential SQL injection detected: {context} = {value}'
                )
                raise ValidationError(f"Invalid characters in {context}")
    
    def filter_queryset(self, request, queryset):
        """Safe query filtering"""
        # Validate parameters before filtering
        self.validate_query_parameters(request)
        
        return super().filter_queryset(request, queryset)

class SecureArticleView(SQLInjectionProtectionMixin, ModelView):
    model = Article
```

### XSS Prevention

Prevent Cross-Site Scripting attacks:

```python
import bleach
from django.utils.html import escape
from django.core.exceptions import ValidationError

class XSSProtectionMixin:
    # Allowed HTML tags and attributes
    ALLOWED_TAGS = ['p', 'br', 'strong', 'em', 'ul', 'ol', 'li', 'a']
    ALLOWED_ATTRIBUTES = {
        'a': ['href', 'title'],
    }
    
    def sanitize_html_input(self, value, field_name):
        """Sanitize HTML input to prevent XSS"""
        if not isinstance(value, str):
            return value
        
        # Check for dangerous scripts
        if self.contains_dangerous_content(value):
            security_logger.warning(
                f'Potential XSS attempt in field {field_name}: {value[:100]}'
            )
            raise ValidationError(f"Invalid content in {field_name}")
        
        # Clean HTML content
        cleaned_value = bleach.clean(
            value,
            tags=self.ALLOWED_TAGS,
            attributes=self.ALLOWED_ATTRIBUTES,
            strip=True
        )
        
        return cleaned_value
    
    def contains_dangerous_content(self, value):
        """Check for dangerous XSS patterns"""
        dangerous_patterns = [
            r'<script[^>]*>',
            r'javascript:',
            r'on\w+\s*=',  # Event handlers like onclick=
            r'<iframe[^>]*>',
            r'<object[^>]*>',
            r'<embed[^>]*>',
        ]
        
        value_lower = value.lower()
        for pattern in dangerous_patterns:
            if re.search(pattern, value_lower, re.IGNORECASE):
                return True
        
        return False
    
    def validate_request_data(self, request, data):
        """Validate and sanitize request data"""
        # Sanitize HTML fields
        html_fields = getattr(self, 'html_fields', [])
        for field_name in html_fields:
            if field_name in data:
                data[field_name] = self.sanitize_html_input(data[field_name], field_name)
        
        return super().validate_request_data(request, data)

class ArticleView(XSSProtectionMixin, ModelView):
    model = Article
    html_fields = ['content', 'excerpt']
```

## File Upload Security

### Secure File Validation

Implement comprehensive file upload security:

```python
import magic
import hashlib
from PIL import Image
from django.core.exceptions import ValidationError
from django.conf import settings

class SecureFileUploadMixin:
    # Allowed file types
    ALLOWED_FILE_TYPES = {
        'image': ['image/jpeg', 'image/png', 'image/gif', 'image/webp'],
        'document': ['application/pdf', 'application/msword', 
                    'application/vnd.openxmlformats-officedocument.wordprocessingml.document'],
        'text': ['text/plain', 'text/csv'],
    }
    
    # Maximum file sizes (in bytes)
    MAX_FILE_SIZES = {
        'image': 10 * 1024 * 1024,    # 10MB
        'document': 50 * 1024 * 1024, # 50MB
        'text': 1 * 1024 * 1024,      # 1MB
    }
    
    def validate_file_upload(self, field_name, file_obj):
        """Comprehensive file validation"""
        # Basic validation
        super().validate_file_upload(field_name, file_obj)
        
        # Get file type category
        file_category = self.get_file_category(field_name)
        
        # Validate file type using magic numbers
        self.validate_file_type(file_obj, file_category)
        
        # Validate file size
        self.validate_file_size(file_obj, file_category)
        
        # Validate file content
        self.validate_file_content(file_obj, file_category)
        
        # Scan for malware (if configured)
        if hasattr(settings, 'ENABLE_MALWARE_SCANNING') and settings.ENABLE_MALWARE_SCANNING:
            self.scan_file_for_malware(file_obj)
    
    def get_file_category(self, field_name):
        """Determine file category from field name"""
        field_categories = getattr(self, 'file_field_categories', {})
        return field_categories.get(field_name, 'document')
    
    def validate_file_type(self, file_obj, category):
        """Validate file type using magic numbers"""
        # Read file header to determine actual type
        file_obj.seek(0)
        file_header = file_obj.read(1024)
        file_obj.seek(0)
        
        # Get MIME type from file content
        actual_mime_type = magic.from_buffer(file_header, mime=True)
        
        # Check against allowed types
        allowed_types = self.ALLOWED_FILE_TYPES.get(category, [])
        if actual_mime_type not in allowed_types:
            raise ValidationError(
                f'File type {actual_mime_type} not allowed. '
                f'Allowed types: {", ".join(allowed_types)}'
            )
    
    def validate_file_size(self, file_obj, category):
        """Validate file size"""
        max_size = self.MAX_FILE_SIZES.get(category, 1024 * 1024)  # Default 1MB
        
        if file_obj.size > max_size:
            raise ValidationError(
                f'File size ({file_obj.size} bytes) exceeds maximum '
                f'allowed size ({max_size} bytes)'
            )
    
    def validate_file_content(self, file_obj, category):
        """Validate file content for security issues"""
        if category == 'image':
            self.validate_image_content(file_obj)
        elif category == 'document':
            self.validate_document_content(file_obj)
    
    def validate_image_content(self, file_obj):
        """Validate image file content"""
        try:
            # Open image to validate it's not corrupted
            img = Image.open(file_obj)
            img.verify()
            
            # Check image dimensions
            file_obj.seek(0)
            img = Image.open(file_obj)
            
            max_dimensions = getattr(settings, 'MAX_IMAGE_DIMENSIONS', (4000, 4000))
            if img.width > max_dimensions[0] or img.height > max_dimensions[1]:
                raise ValidationError(
                    f'Image dimensions ({img.width}x{img.height}) exceed '
                    f'maximum allowed ({max_dimensions[0]}x{max_dimensions[1]})'
                )
            
        except Exception as e:
            raise ValidationError(f'Invalid image file: {str(e)}')
    
    def validate_document_content(self, file_obj):
        """Validate document file content"""
        # Check for embedded scripts or macros
        file_obj.seek(0)
        content = file_obj.read(8192)  # Read first 8KB
        file_obj.seek(0)
        
        # Look for dangerous patterns
        dangerous_patterns = [
            b'<script',
            b'javascript:',
            b'vbscript:',
            b'ActiveXObject',
        ]
        
        content_lower = content.lower()
        for pattern in dangerous_patterns:
            if pattern in content_lower:
                raise ValidationError('Document contains potentially dangerous content')
    
    def scan_file_for_malware(self, file_obj):
        """Scan file for malware using ClamAV"""
        import subprocess
        import tempfile
        import os
        
        # Save file temporarily
        with tempfile.NamedTemporaryFile(delete=False) as temp_file:
            file_obj.seek(0)
            temp_file.write(file_obj.read())
            temp_file_path = temp_file.name
        
        try:
            # Run ClamAV scan
            result = subprocess.run(
                ['clamscan', '--no-summary', temp_file_path],
                capture_output=True,
                text=True,
                timeout=30
            )
            
            if result.returncode != 0:
                security_logger.warning(f'Malware detected in uploaded file: {result.stdout}')
                raise ValidationError('File failed security scan')
                
        except subprocess.TimeoutExpired:
            raise ValidationError('File scan timeout')
        except FileNotFoundError:
            # ClamAV not installed, log warning but don't fail
            security_logger.warning('ClamAV not available for file scanning')
        finally:
            # Clean up temp file
            os.unlink(temp_file_path)
            file_obj.seek(0)

class DocumentView(SecureFileUploadMixin, ModelView):
    model = Document
    file_fields = ['file', 'attachment']
    file_field_categories = {
        'file': 'document',
        'attachment': 'document'
    }
```

## Rate Limiting

### API Rate Limiting

Implement rate limiting to prevent abuse:

```python
from django.core.cache import cache
from django.http import JsonResponse
from functools import wraps
import time

class RateLimitMixin:
    # Rate limit settings
    rate_limit_requests = 100  # Requests per window
    rate_limit_window = 3600   # Window in seconds (1 hour)
    rate_limit_key_func = None # Custom key function
    
    def dispatch(self, request, *args, **kwargs):
        """Apply rate limiting to requests"""
        if self.should_rate_limit(request):
            if not self.check_rate_limit(request):
                return self.rate_limit_exceeded_response(request)
        
        return super().dispatch(request, *args, **kwargs)
    
    def should_rate_limit(self, request):
        """Determine if request should be rate limited"""
        # Don't rate limit staff users
        if request.user.is_authenticated and request.user.is_staff:
            return False
        
        # Don't rate limit GET requests as strictly
        if request.method == 'GET':
            return False
        
        return True
    
    def get_rate_limit_key(self, request):
        """Generate rate limit cache key"""
        if self.rate_limit_key_func:
            return self.rate_limit_key_func(request)
        
        # Use IP address for anonymous users, user ID for authenticated users
        if request.user.is_authenticated:
            identifier = f"user_{request.user.id}"
        else:
            identifier = f"ip_{self.get_client_ip(request)}"
        
        return f"rate_limit:{self.__class__.__name__}:{identifier}"
    
    def get_client_ip(self, request):
        """Get client IP address"""
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            ip = x_forwarded_for.split(',')[0]
        else:
            ip = request.META.get('REMOTE_ADDR')
        return ip
    
    def check_rate_limit(self, request):
        """Check if request is within rate limit"""
        cache_key = self.get_rate_limit_key(request)
        
        # Get current request count
        current_requests = cache.get(cache_key, 0)
        
        if current_requests >= self.rate_limit_requests:
            # Log rate limit violation
            security_logger.warning(
                f'Rate limit exceeded: {cache_key}, '
                f'requests: {current_requests}/{self.rate_limit_requests}'
            )
            return False
        
        # Increment request count
        cache.set(cache_key, current_requests + 1, self.rate_limit_window)
        
        return True
    
    def rate_limit_exceeded_response(self, request):
        """Response when rate limit is exceeded"""
        return JsonResponse({
            'error': 'Rate limit exceeded',
            'message': f'Maximum {self.rate_limit_requests} requests per {self.rate_limit_window} seconds',
            'retry_after': self.rate_limit_window
        }, status=429)

class ArticleView(RateLimitMixin, ModelView):
    model = Article
    rate_limit_requests = 50  # 50 requests per hour for articles
```

## Security Headers

### HTTP Security Headers

Implement comprehensive security headers:

```python
from django.utils.deprecation import MiddlewareMixin

class SecurityHeadersMiddleware(MiddlewareMixin):
    def process_response(self, request, response):
        """Add security headers to all responses"""
        
        # Content Security Policy
        csp_policy = (
            "default-src 'self'; "
            "script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; "
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; "
            "font-src 'self' https://fonts.gstatic.com; "
            "img-src 'self' data: https:; "
            "connect-src 'self'; "
            "frame-ancestors 'none';"
        )
        response['Content-Security-Policy'] = csp_policy
        
        # Additional security headers
        response['X-Content-Type-Options'] = 'nosniff'
        response['X-Frame-Options'] = 'DENY'
        response['X-XSS-Protection'] = '1; mode=block'
        response['Referrer-Policy'] = 'strict-origin-when-cross-origin'
        response['Permissions-Policy'] = 'geolocation=(), microphone=(), camera=()'
        
        # Remove server information
        if 'Server' in response:
            del response['Server']
        
        return response

# Add to MIDDLEWARE
MIDDLEWARE += ['your_app.middleware.SecurityHeadersMiddleware']
```

## Audit Logging

### Security Event Logging

Implement comprehensive security logging:

```python
import logging
from django.contrib.auth.signals import user_logged_in, user_logged_out, user_login_failed
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver

# Configure security logger
security_logger = logging.getLogger('security')

class SecurityAuditLogger:
    @staticmethod
    def log_security_event(event_type, user, details, request=None):
        """Log security events"""
        log_data = {
            'event_type': event_type,
            'user_id': user.id if user and user.is_authenticated else None,
            'username': user.username if user and user.is_authenticated else 'anonymous',
            'details': details,
            'timestamp': timezone.now().isoformat(),
        }
        
        if request:
            log_data.update({
                'ip_address': get_client_ip(request),
                'user_agent': request.META.get('HTTP_USER_AGENT', ''),
                'path': request.path,
                'method': request.method,
            })
        
        security_logger.info(f"Security Event: {log_data}")

# Signal handlers for authentication events
@receiver(user_logged_in)
def log_user_login(sender, request, user, **kwargs):
    SecurityAuditLogger.log_security_event(
        'user_login',
        user,
        {'success': True},
        request
    )

@receiver(user_logged_out)
def log_user_logout(sender, request, user, **kwargs):
    SecurityAuditLogger.log_security_event(
        'user_logout',
        user,
        {'success': True},
        request
    )

@receiver(user_login_failed)
def log_login_failure(sender, credentials, request, **kwargs):
    SecurityAuditLogger.log_security_event(
        'login_failed',
        None,
        {'username': credentials.get('username', 'unknown')},
        request
    )

# Model change logging
class SecurityAuditMixin:
    def store(self, obj, fields, request):
        """Log model changes for security audit"""
        action = 'created' if not obj.pk else 'updated'
        
        result = super().store(obj, fields, request)
        
        # Log the change
        SecurityAuditLogger.log_security_event(
            f'model_{action}',
            request.user,
            {
                'model': self.model.__name__,
                'object_id': obj.pk,
                'fields_changed': list(fields.keys()) if fields else []
            },
            request
        )
        
        return result

class SecureArticleView(SecurityAuditMixin, ModelView):
    model = Article
```

## Security Testing

### Security Test Suite

Implement comprehensive security tests:

```python
from django.test import TestCase, Client
from django.contrib.auth.models import User
from django.urls import reverse
import json

class SecurityTestCase(TestCase):
    def setUp(self):
        self.client = Client()
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        self.admin = User.objects.create_superuser('admin', 'admin@example.com', 'password')
    
    def test_sql_injection_protection(self):
        """Test SQL injection protection"""
        # Attempt SQL injection in query parameters
        malicious_params = [
            "'; DROP TABLE articles; --",
            "1' OR '1'='1",
            "UNION SELECT * FROM users",
        ]
        
        for param in malicious_params:
            response = self.client.get(f'/api/article/?title={param}')
            # Should not return 500 error or expose data
            self.assertIn(response.status_code, [200, 400])
    
    def test_xss_protection(self):
        """Test XSS protection"""
        self.client.force_login(self.user)
        
        malicious_content = [
            "<script>alert('XSS')</script>",
            "javascript:alert('XSS')",
            "<img src=x onerror=alert('XSS')>",
        ]
        
        for content in malicious_content:
            response = self.client.post('/api/article/', {
                'title': 'Test Article',
                'content': content
            }, content_type='application/json')
            
            # Should either reject or sanitize
            if response.status_code == 200:
                article_data = response.json()
                self.assertNotIn('<script>', article_data.get('content', ''))
    
    def test_unauthorized_access(self):
        """Test unauthorized access protection"""
        # Create article as user
        self.client.force_login(self.user)
        response = self.client.post('/api/article/', {
            'title': 'Private Article',
            'content': 'Private content'
        }, content_type='application/json')
        
        article_id = response.json()['id']
        
        # Try to access as different user
        other_user = User.objects.create_user('other', 'other@example.com', 'password')
        self.client.force_login(other_user)
        
        response = self.client.get(f'/api/article/{article_id}/')
        # Should be forbidden or not found
        self.assertIn(response.status_code, [403, 404])
    
    def test_rate_limiting(self):
        """Test rate limiting"""
        # Make many requests quickly
        for i in range(10):
            response = self.client.get('/api/article/')
        
        # Should eventually get rate limited (if implemented)
        # This test depends on your rate limiting configuration
        pass
    
    def test_file_upload_security(self):
        """Test file upload security"""
        self.client.force_login(self.user)
        
        # Test malicious file upload
        malicious_content = b"<?php system($_GET['cmd']); ?>"
        
        response = self.client.post('/api/document/', {
            'title': 'Test Document',
            'file': SimpleUploadedFile('malicious.php', malicious_content, content_type='text/plain')
        })
        
        # Should reject malicious files
        self.assertEqual(response.status_code, 400)
    
    def test_csrf_protection(self):
        """Test CSRF protection"""
        self.client.force_login(self.user)
        
        # Make request without CSRF token
        response = self.client.post('/api/article/', {
            'title': 'Test Article',
            'content': 'Test content'
        }, content_type='application/json', HTTP_X_CSRFTOKEN='invalid')
        
        # Should be protected by CSRF
        self.assertIn(response.status_code, [403, 400])
```

This comprehensive security guide provides multiple layers of protection for Django Binder applications, from input validation to audit logging and security testing.
