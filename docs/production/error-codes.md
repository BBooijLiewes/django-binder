# Error Codes

This comprehensive reference covers all error codes, HTTP status codes, and error handling in Django Binder applications.

## HTTP Status Codes

### Success Codes (2xx)

#### 200 OK
Standard success response for most operations.

```json
{
  "id": 1,
  "title": "Article Title",
  "content": "Article content"
}
```

**Used for:**
- GET requests (single object and lists)
- POST requests (object creation)
- PUT requests (object updates)
- Custom endpoints returning data

#### 201 Created
Resource successfully created (rarely used in Django Binder).

#### 204 No Content
Successful request with no response body.

**Used for:**
- DELETE requests
- Some custom endpoints that don't return data

### Client Error Codes (4xx)

#### 400 Bad Request
Invalid request data or validation errors.

```json
{
  "code": "ValidationError",
  "error": {
    "validation_errors": {
      "title": [
        {
          "code": "This field is required."
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
```

**Common causes:**
- Missing required fields
- Invalid field values
- Validation rule violations
- Malformed JSON

#### 401 Unauthorized
Authentication required but not provided.

```json
{
  "code": "NotAuthenticated",
  "error": {
    "message": "Authentication credentials were not provided."
  }
}
```

**Common causes:**
- Missing authentication token
- Invalid authentication token
- Expired authentication token

#### 403 Forbidden
Authentication provided but insufficient permissions.

```json
{
  "code": "PermissionDenied",
  "error": {
    "message": "You do not have permission to perform this action.",
    "required_permission": "myapp.change_mymodel"
  }
}
```

**Common causes:**
- User lacks required permissions
- Object-level permission denied
- Scope restrictions applied

#### 404 Not Found
Requested resource does not exist.

```json
{
  "code": "NotFound",
  "error": {
    "message": "Object with id 999 not found."
  }
}
```

**Common causes:**
- Invalid object ID
- Object deleted or soft-deleted
- User lacks view permissions for object

#### 405 Method Not Allowed
HTTP method not supported for endpoint.

```json
{
  "code": "MethodNotAllowed",
  "error": {
    "message": "Method 'PATCH' not allowed.",
    "allowed_methods": ["GET", "POST", "PUT", "DELETE"]
  }
}
```

**Common causes:**
- Using PATCH instead of PUT
- Calling POST on detail endpoints
- Custom endpoints with method restrictions

#### 413 Payload Too Large
Request payload exceeds size limits.

```json
{
  "code": "PayloadTooLarge",
  "error": {
    "message": "Request payload too large.",
    "max_size": "10MB"
  }
}
```

**Common causes:**
- File uploads exceeding limits
- Large JSON payloads
- Multi-PUT with too many objects

#### 415 Unsupported Media Type
Content-Type not supported.

```json
{
  "code": "UnsupportedMediaType",
  "error": {
    "message": "Unsupported media type 'text/xml'.",
    "supported_types": ["application/json", "multipart/form-data"]
  }
}
```

#### 429 Too Many Requests
Rate limit exceeded.

```json
{
  "code": "RateLimitExceeded",
  "error": {
    "message": "Rate limit exceeded.",
    "retry_after": 3600,
    "limit": 100,
    "window": 3600
  }
}
```

### Server Error Codes (5xx)

#### 500 Internal Server Error
Unexpected server error.

```json
{
  "code": "InternalServerError",
  "error": {
    "message": "An unexpected error occurred.",
    "request_id": "abc123def456"
  }
}
```

**Common causes:**
- Unhandled exceptions
- Database connection errors
- Configuration issues

#### 502 Bad Gateway
Upstream server error (load balancer, proxy).

#### 503 Service Unavailable
Service temporarily unavailable.

```json
{
  "code": "ServiceUnavailable",
  "error": {
    "message": "Service temporarily unavailable.",
    "retry_after": 300
  }
}
```

## Django Binder Error Codes

### Validation Errors

#### ValidationError
Field validation failures.

```json
{
  "code": "ValidationError",
  "error": {
    "validation_errors": {
      "field_name": [
        {
          "code": "Error message"
        }
      ]
    }
  }
}
```

**Field-specific errors:**

```json
{
  "code": "ValidationError",
  "error": {
    "validation_errors": {
      "title": [
        {
          "code": "This field is required."
        }
      ],
      "email": [
        {
          "code": "Enter a valid email address."
        }
      ],
      "age": [
        {
          "code": "Ensure this value is greater than or equal to 0."
        }
      ],
      "password": [
        {
          "code": "This password is too short. It must contain at least 8 characters."
        }
      ]
    }
  }
}
```

**Multi-PUT validation errors:**

```json
{
  "code": "ValidationError",
  "error": {
    "validation_errors": {
      "data": {
        "0": {
          "title": [
            {
              "code": "This field is required."
            }
          ]
        },
        "1": {
          "email": [
            {
              "code": "Enter a valid email address."
            }
          ]
        }
      },
      "with": {
        "category": {
          "0": {
            "name": [
              {
                "code": "This field is required."
              }
            ]
          }
        }
      }
    }
  }
}
```

### Permission Errors

#### PermissionDenied
Insufficient permissions for operation.

```json
{
  "code": "PermissionDenied",
  "error": {
    "message": "You do not have permission to perform this action.",
    "required_permission": "myapp.change_mymodel",
    "user_permissions": ["myapp.view_mymodel"]
  }
}
```

#### NotAuthenticated
Authentication required.

```json
{
  "code": "NotAuthenticated",
  "error": {
    "message": "Authentication credentials were not provided.",
    "authentication_methods": ["Token", "Session"]
  }
}
```

### File Upload Errors

#### FileSizeExceeded
File size exceeds limits.

```json
{
  "code": "FileSizeExceeded",
  "error": {
    "message": "File size exceeds maximum allowed size.",
    "file_size": 15728640,
    "max_size": 10485760,
    "field": "attachment"
  }
}
```

#### FileTypeIncorrect
Invalid file type.

```json
{
  "code": "FileTypeIncorrect",
  "error": {
    "message": "File type not allowed.",
    "file_type": "application/x-executable",
    "allowed_types": ["image/jpeg", "image/png", "application/pdf"],
    "field": "document"
  }
}
```

#### ImageError
Image processing error.

```json
{
  "code": "ImageError",
  "error": {
    "message": "Invalid image file.",
    "details": "Cannot identify image file",
    "field": "photo"
  }
}
```

#### ImageSizeExceeded
Image dimensions exceed limits.

```json
{
  "code": "ImageSizeExceeded",
  "error": {
    "message": "Image dimensions exceed maximum allowed size.",
    "image_size": [4000, 3000],
    "max_size": [2000, 2000],
    "field": "photo"
  }
}
```

### Database Errors

#### ObjectNotFound
Requested object does not exist.

```json
{
  "code": "ObjectNotFound",
  "error": {
    "message": "Article with id 999 not found.",
    "model": "Article",
    "lookup": {"id": 999}
  }
}
```

#### IntegrityError
Database integrity constraint violation.

```json
{
  "code": "IntegrityError",
  "error": {
    "message": "Duplicate entry for unique field.",
    "field": "email",
    "value": "user@example.com"
  }
}
```

### Business Logic Errors

#### BusinessRuleViolation
Custom business rule violation.

```json
{
  "code": "BusinessRuleViolation",
  "error": {
    "message": "Cannot publish article without content.",
    "rule": "published_articles_require_content",
    "field": "published"
  }
}
```

#### StateTransitionError
Invalid state transition.

```json
{
  "code": "StateTransitionError",
  "error": {
    "message": "Cannot transition from 'published' to 'draft'.",
    "current_state": "published",
    "requested_state": "draft",
    "allowed_transitions": ["archived"]
  }
}
```

## Error Response Format

### Standard Error Structure

All Django Binder errors follow this structure:

```json
{
  "code": "ErrorCode",
  "error": {
    "message": "Human-readable error message",
    // Additional error-specific fields
  }
}
```

### Error Context

Errors may include additional context:

```json
{
  "code": "ValidationError",
  "error": {
    "message": "Validation failed",
    "validation_errors": { /* field errors */ },
    "timestamp": "2023-12-01T10:30:00Z",
    "request_id": "abc123def456",
    "user_id": 5
  }
}
```

## Custom Error Handling

### Creating Custom Errors

```python
from binder.exceptions import BinderException

class CustomBusinessError(BinderException):
    http_code = 400
    
    def __init__(self, message, details=None):
        self.message = message
        self.details = details or {}
        super().__init__(message)
    
    def response_data(self):
        return {
            'code': 'CustomBusinessError',
            'error': {
                'message': self.message,
                **self.details
            }
        }

# Usage
raise CustomBusinessError(
    "Invalid operation",
    details={'operation': 'publish', 'reason': 'missing_content'}
)
```

### Error Handling in Views

```python
class ArticleView(ModelView):
    model = Article
    
    def validate_request_data(self, request, data):
        """Custom validation with detailed errors"""
        errors = {}
        
        # Business rule validation
        if data.get('published') and not data.get('content'):
            errors['content'] = ['Published articles must have content']
        
        # Custom field validation
        if 'title' in data and len(data['title']) < 5:
            errors['title'] = ['Title must be at least 5 characters']
        
        if errors:
            raise ValidationError(errors)
        
        return super().validate_request_data(request, data)
    
    def store(self, obj, fields, request):
        """Custom save with error handling"""
        try:
            return super().store(obj, fields, request)
        except IntegrityError as e:
            if 'unique constraint' in str(e).lower():
                raise CustomBusinessError(
                    "Article with this title already exists",
                    details={'field': 'title', 'constraint': 'unique'}
                )
            raise
```

## Error Logging

### Error Logging Configuration

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'error_formatter': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'error_file': {
            'level': 'ERROR',
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': 'logs/errors.log',
            'maxBytes': 1024*1024*10,  # 10MB
            'backupCount': 5,
            'formatter': 'error_formatter',
        },
    },
    'loggers': {
        'binder.errors': {
            'handlers': ['error_file'],
            'level': 'ERROR',
            'propagate': True,
        },
    },
}
```

### Error Logging in Views

```python
import logging

error_logger = logging.getLogger('binder.errors')

class ArticleView(ModelView):
    model = Article
    
    def dispatch(self, request, *args, **kwargs):
        try:
            return super().dispatch(request, *args, **kwargs)
        except Exception as e:
            # Log error with context
            error_logger.error(
                f"Error in {self.__class__.__name__}: {str(e)}",
                extra={
                    'user_id': request.user.id if request.user.is_authenticated else None,
                    'request_path': request.path,
                    'request_method': request.method,
                    'exception_type': type(e).__name__,
                }
            )
            raise
```

## Error Monitoring

### Sentry Integration

```python
# settings.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn="your-sentry-dsn",
    integrations=[DjangoIntegration()],
    traces_sample_rate=0.1,
    send_default_pii=True
)
```

### Custom Error Tracking

```python
class ErrorTrackingMixin:
    def dispatch(self, request, *args, **kwargs):
        try:
            return super().dispatch(request, *args, **kwargs)
        except Exception as e:
            # Track error metrics
            self.track_error(e, request)
            raise
    
    def track_error(self, exception, request):
        """Track error for monitoring"""
        from django.core.cache import cache
        
        error_key = f"error_count:{type(exception).__name__}"
        current_count = cache.get(error_key, 0)
        cache.set(error_key, current_count + 1, 3600)  # 1 hour
        
        # Alert if error rate is high
        if current_count > 10:
            self.send_error_alert(exception, current_count)
    
    def send_error_alert(self, exception, count):
        """Send alert for high error rates"""
        # Implementation depends on your alerting system
        pass
```

## Client Error Handling

### JavaScript Error Handling

```javascript
class APIClient {
    async request(url, options = {}) {
        try {
            const response = await fetch(url, options);
            
            if (!response.ok) {
                const errorData = await response.json();
                throw new APIError(errorData, response.status);
            }
            
            return await response.json();
        } catch (error) {
            if (error instanceof APIError) {
                throw error;
            }
            
            // Network or other errors
            throw new APIError({
                code: 'NetworkError',
                error: { message: 'Network request failed' }
            }, 0);
        }
    }
}

class APIError extends Error {
    constructor(errorData, statusCode) {
        super(errorData.error?.message || 'API Error');
        this.code = errorData.code;
        this.statusCode = statusCode;
        this.errorData = errorData;
    }
    
    isValidationError() {
        return this.code === 'ValidationError';
    }
    
    isPermissionError() {
        return this.code === 'PermissionDenied';
    }
    
    getFieldErrors() {
        if (this.isValidationError()) {
            return this.errorData.error.validation_errors || {};
        }
        return {};
    }
}

// Usage
try {
    const article = await api.createArticle(articleData);
} catch (error) {
    if (error.isValidationError()) {
        const fieldErrors = error.getFieldErrors();
        // Display field-specific errors
        Object.entries(fieldErrors).forEach(([field, errors]) => {
            console.log(`${field}: ${errors.map(e => e.code).join(', ')}`);
        });
    } else if (error.isPermissionError()) {
        // Handle permission error
        console.log('Permission denied:', error.message);
    } else {
        // Handle other errors
        console.log('Error:', error.message);
    }
}
```

This comprehensive error code reference helps developers understand and handle all types of errors in Django Binder applications.
