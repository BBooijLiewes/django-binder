# File Handling

Django Binder provides comprehensive file handling capabilities with enhanced file fields, automatic validation, image processing, and efficient file serving.

## Enhanced File Fields

### BinderFileField

Enhanced file field with additional metadata and features:

```python
from binder.models import BinderModel, BinderFileField

class Document(BinderModel):
    title = models.CharField(max_length=200)
    file = BinderFileField(
        upload_to='documents/%Y/%m/',
        allowed_extensions=['pdf', 'doc', 'docx', 'txt'],
        max_length=200,
        blank=True,
        null=True
    )
    
    class Meta:
        db_table = 'documents'
```

**BinderFileField Features:**
- **Automatic SHA1 Hash**: Generated for change detection
- **Content Type Detection**: Automatic MIME type detection
- **Filename Preservation**: Original filename is preserved
- **Extension Validation**: Restrict allowed file extensions
- **File Size Validation**: Built-in size limits

### BinderImageField

Enhanced image field with processing capabilities:

```python
from binder.models import BinderImageField

class Photo(BinderModel):
    title = models.CharField(max_length=200)
    image = BinderImageField(
        upload_to='photos/%Y/%m/',
        allowed_extensions=['jpg', 'jpeg', 'png', 'gif', 'webp'],
        serve_directly=True,  # Delegate to web server
        max_length=200
    )
    thumbnail = BinderImageField(
        upload_to='thumbnails/',
        blank=True,
        null=True
    )
```

**BinderImageField Additional Features:**
- **Image Validation**: Validates uploaded files are valid images
- **Format Support**: JPEG, PNG, GIF, WebP support
- **Automatic Processing**: Optional thumbnail generation
- **Format Conversion**: Convert between image formats

## File Upload Configuration

### View Configuration

Configure file handling in your views:

```python
class DocumentView(ModelView):
    model = Document
    file_fields = ['file', 'attachment', 'cover_image']
    
    # Image processing configuration
    image_resize_threshold = {
        'cover_image': 1200,  # Resize if larger than 1200px
        'thumbnail': 300,
    }
    
    # Image format conversion
    image_format_override = {
        'cover_image': 'jpeg',  # Convert to JPEG
        'thumbnail': 'png',     # Convert to PNG
    }
```

### File Validation

Implement custom file validation:

```python
from django.core.exceptions import ValidationError
from PIL import Image
import magic

class DocumentView(ModelView):
    model = Document
    file_fields = ['file', 'image']
    
    def validate_file_upload(self, field_name, file_obj):
        """Custom file validation"""
        
        if field_name == 'file':
            # Validate file size (10MB limit)
            if file_obj.size > 10 * 1024 * 1024:
                raise ValidationError('File size cannot exceed 10MB')
            
            # Validate file type using python-magic
            file_type = magic.from_buffer(file_obj.read(1024), mime=True)
            file_obj.seek(0)  # Reset file pointer
            
            allowed_types = [
                'application/pdf',
                'application/msword',
                'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
                'text/plain'
            ]
            
            if file_type not in allowed_types:
                raise ValidationError(f'File type {file_type} not allowed')
        
        elif field_name == 'image':
            # Validate image dimensions
            try:
                img = Image.open(file_obj)
                if img.width < 100 or img.height < 100:
                    raise ValidationError('Image must be at least 100x100 pixels')
                
                if img.width > 4000 or img.height > 4000:
                    raise ValidationError('Image cannot exceed 4000x4000 pixels')
                
            except Exception as e:
                raise ValidationError(f'Invalid image file: {str(e)}')
        
        return super().validate_file_upload(field_name, file_obj)
```

## File Upload Endpoints

### Standard File Upload

Upload files to specific fields:

```bash
# Upload a file to the 'file' field of document with ID 1
POST /api/document/1/file/
Content-Type: multipart/form-data

file: [binary file data]
```

### Multiple File Upload

Handle multiple files:

```python
class GalleryView(ModelView):
    model = Gallery
    file_fields = ['images']  # Assuming images is an ArrayField or similar
    
    def handle_file_upload(self, obj, field_name, file_obj):
        """Handle multiple file uploads"""
        if field_name == 'images':
            # Process each uploaded image
            processed_file = self.process_image(file_obj)
            return processed_file
        
        return super().handle_file_upload(obj, field_name, file_obj)
    
    def process_image(self, image_file):
        """Process uploaded image"""
        from PIL import Image
        import io
        
        # Open and process image
        img = Image.open(image_file)
        
        # Resize if too large
        if img.width > 1200 or img.height > 1200:
            img.thumbnail((1200, 1200), Image.Resampling.LANCZOS)
        
        # Convert to RGB if necessary
        if img.mode in ('RGBA', 'LA', 'P'):
            img = img.convert('RGB')
        
        # Save processed image
        output = io.BytesIO()
        img.save(output, format='JPEG', quality=85, optimize=True)
        output.seek(0)
        
        return ContentFile(output.read(), name=image_file.name)
```

### File Upload with Data

Upload files together with model data:

```bash
# Upload file with model data using multipart form
PUT /api/document/1/
Content-Type: multipart/form-data

data: {"title": "Updated Document", "description": "New description", "file": null}
file:file: [binary file data]
```

```python
class DocumentView(ModelView):
    model = Document
    file_fields = ['file']
    
    def _get_request_values(self, request):
        """Handle multipart form data with files"""
        # This method automatically handles the file:field_name format
        # and integrates files into the data structure
        return super()._get_request_values(request)
```

## File Serving

### Direct File Serving

Configure direct file serving by web server:

```python
# settings.py
# For nginx X-Accel-Redirect
INTERNAL_MEDIA_HEADER = 'X-Accel-Redirect'
INTERNAL_MEDIA_LOCATION = '/internal/media/'

# For Apache X-Sendfile
# INTERNAL_MEDIA_HEADER = 'X-Sendfile'
# INTERNAL_MEDIA_LOCATION = '/path/to/media/'

# Model configuration
class Document(BinderModel):
    file = BinderFileField(
        upload_to='documents/',
        serve_directly=True  # Enable direct serving
    )
```

### Custom File Serving

Implement custom file serving logic:

```python
from django.http import FileResponse, Http404
from django.core.exceptions import PermissionDenied

class DocumentView(ModelView):
    model = Document
    file_fields = ['file']
    
    def serve_file(self, request, obj, field_name):
        """Custom file serving with access control"""
        
        # Check permissions
        if not self.has_file_access(request.user, obj, field_name):
            raise PermissionDenied("Access denied")
        
        # Get file field
        file_field = getattr(obj, field_name)
        if not file_field:
            raise Http404("File not found")
        
        # Log file access
        self.log_file_access(request.user, obj, field_name)
        
        # Serve file with custom headers
        response = FileResponse(
            file_field.open('rb'),
            content_type=file_field.content_type or 'application/octet-stream'
        )
        
        # Set download filename
        response['Content-Disposition'] = f'attachment; filename="{file_field.name}"'
        
        # Add custom headers
        response['X-File-Hash'] = file_field.hash
        response['X-File-Size'] = str(file_field.size)
        
        return response
    
    def has_file_access(self, user, obj, field_name):
        """Check if user can access this file"""
        # Implement your access control logic
        if field_name == 'private_file':
            return obj.owner == user or user.is_staff
        return True
    
    def log_file_access(self, user, obj, field_name):
        """Log file access for auditing"""
        import logging
        logger = logging.getLogger('file_access')
        logger.info(f'User {user.id} accessed {field_name} on {obj.__class__.__name__} {obj.id}')
```

## Image Processing

### Automatic Image Processing

Configure automatic image processing:

```python
class PhotoView(ModelView):
    model = Photo
    file_fields = ['image', 'thumbnail']
    
    # Resize images larger than threshold
    image_resize_threshold = {
        'image': 1920,      # Resize to max 1920px
        'thumbnail': 300,   # Resize to max 300px
    }
    
    # Convert image formats
    image_format_override = {
        'image': 'jpeg',     # Convert to JPEG
        'thumbnail': 'webp', # Convert to WebP
    }
    
    def handle_file_upload(self, obj, field_name, file_obj):
        """Custom image processing"""
        if field_name == 'image':
            # Generate thumbnail automatically
            self.generate_thumbnail(obj, file_obj)
            
            # Extract EXIF data
            self.extract_exif_data(obj, file_obj)
        
        return super().handle_file_upload(obj, field_name, file_obj)
    
    def generate_thumbnail(self, obj, image_file):
        """Generate thumbnail from uploaded image"""
        from PIL import Image
        import io
        
        # Open original image
        img = Image.open(image_file)
        
        # Create thumbnail
        img.thumbnail((300, 300), Image.Resampling.LANCZOS)
        
        # Save thumbnail
        thumb_io = io.BytesIO()
        img.save(thumb_io, format='JPEG', quality=85)
        thumb_io.seek(0)
        
        # Save to thumbnail field
        obj.thumbnail.save(
            f'thumb_{obj.id}.jpg',
            ContentFile(thumb_io.read()),
            save=False
        )
    
    def extract_exif_data(self, obj, image_file):
        """Extract EXIF data from image"""
        from PIL import Image
        from PIL.ExifTags import TAGS
        
        try:
            img = Image.open(image_file)
            exif_data = {}
            
            if hasattr(img, '_getexif'):
                exif = img._getexif()
                if exif:
                    for tag_id, value in exif.items():
                        tag = TAGS.get(tag_id, tag_id)
                        exif_data[tag] = value
            
            # Store EXIF data (assuming you have a JSONField)
            if hasattr(obj, 'exif_data'):
                obj.exif_data = exif_data
                
        except Exception as e:
            # Log error but don't fail upload
            logger.warning(f'Failed to extract EXIF data: {e}')
```

### Image Manipulation Endpoints

Add custom image manipulation endpoints:

```python
from binder.router import detail_route
from PIL import Image
import io

class PhotoView(ModelView):
    model = Photo
    file_fields = ['image']
    
    @detail_route(name='rotate', methods=['POST'])
    def rotate_image(self, request, pk):
        """Rotate image by specified degrees"""
        photo = self.get_object(pk)
        
        if not photo.image:
            return JsonResponse({'error': 'No image found'}, status=404)
        
        # Get rotation angle from request
        data = self._get_request_data(request)
        angle = data.get('angle', 0)
        
        try:
            # Open and rotate image
            img = Image.open(photo.image.path)
            rotated_img = img.rotate(angle, expand=True)
            
            # Save rotated image
            output = io.BytesIO()
            rotated_img.save(output, format='JPEG', quality=90)
            output.seek(0)
            
            # Update file
            photo.image.save(
                photo.image.name,
                ContentFile(output.read()),
                save=True
            )
            
            return JsonResponse({
                'success': True,
                'message': f'Image rotated by {angle} degrees'
            })
            
        except Exception as e:
            return JsonResponse({
                'error': f'Failed to rotate image: {str(e)}'
            }, status=500)
    
    @detail_route(name='crop', methods=['POST'])
    def crop_image(self, request, pk):
        """Crop image to specified dimensions"""
        photo = self.get_object(pk)
        
        if not photo.image:
            return JsonResponse({'error': 'No image found'}, status=404)
        
        # Get crop parameters
        data = self._get_request_data(request)
        x = int(data.get('x', 0))
        y = int(data.get('y', 0))
        width = int(data.get('width', 100))
        height = int(data.get('height', 100))
        
        try:
            # Open and crop image
            img = Image.open(photo.image.path)
            cropped_img = img.crop((x, y, x + width, y + height))
            
            # Save cropped image
            output = io.BytesIO()
            cropped_img.save(output, format='JPEG', quality=90)
            output.seek(0)
            
            # Update file
            photo.image.save(
                photo.image.name,
                ContentFile(output.read()),
                save=True
            )
            
            return JsonResponse({
                'success': True,
                'message': 'Image cropped successfully'
            })
            
        except Exception as e:
            return JsonResponse({
                'error': f'Failed to crop image: {str(e)}'
            }, status=500)
```

## File Security

### Access Control

Implement file access control:

```python
class SecureDocumentView(ModelView):
    model = Document
    file_fields = ['file']
    
    def serve_file(self, request, obj, field_name):
        """Secure file serving with access control"""
        
        # Check if user owns the document or is staff
        if obj.owner != request.user and not request.user.is_staff:
            raise PermissionDenied("You don't have permission to access this file")
        
        # Check if document is published or user is owner
        if not obj.published and obj.owner != request.user:
            raise PermissionDenied("Document is not published")
        
        # Log access attempt
        logger.info(f'File access: user={request.user.id}, document={obj.id}, field={field_name}')
        
        return super().serve_file(request, obj, field_name)
```

### File Scanning

Implement virus scanning:

```python
import subprocess

class SecureDocumentView(ModelView):
    model = Document
    file_fields = ['file']
    
    def validate_file_upload(self, field_name, file_obj):
        """Validate file including virus scan"""
        
        # Run basic validation first
        super().validate_file_upload(field_name, file_obj)
        
        # Virus scan (requires ClamAV)
        if self.scan_for_viruses(file_obj):
            raise ValidationError('File failed security scan')
        
        return True
    
    def scan_for_viruses(self, file_obj):
        """Scan file for viruses using ClamAV"""
        try:
            # Save file temporarily
            temp_path = f'/tmp/scan_{file_obj.name}'
            with open(temp_path, 'wb') as temp_file:
                for chunk in file_obj.chunks():
                    temp_file.write(chunk)
            
            # Run ClamAV scan
            result = subprocess.run(
                ['clamscan', '--no-summary', temp_path],
                capture_output=True,
                text=True
            )
            
            # Clean up temp file
            os.unlink(temp_path)
            
            # Return True if virus found
            return result.returncode != 0
            
        except Exception as e:
            logger.error(f'Virus scan failed: {e}')
            # Fail safe - reject file if scan fails
            return True
```

## File Storage Backends

### Cloud Storage Integration

Configure cloud storage:

```python
# settings.py
# AWS S3 Configuration
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_ACCESS_KEY_ID = 'your-access-key'
AWS_SECRET_ACCESS_KEY = 'your-secret-key'
AWS_STORAGE_BUCKET_NAME = 'your-bucket'
AWS_S3_REGION_NAME = 'us-east-1'
AWS_S3_FILE_OVERWRITE = False
AWS_DEFAULT_ACL = None

# Google Cloud Storage
# DEFAULT_FILE_STORAGE = 'storages.backends.gcloud.GoogleCloudStorage'
# GS_BUCKET_NAME = 'your-bucket'

# Azure Storage
# DEFAULT_FILE_STORAGE = 'storages.backends.azure_storage.AzureStorage'
# AZURE_ACCOUNT_NAME = 'your-account'
# AZURE_CONTAINER = 'your-container'
```

### Custom Storage Backend

Create custom storage backend:

```python
from django.core.files.storage import FileSystemStorage
from django.conf import settings
import os

class SecureFileStorage(FileSystemStorage):
    """Custom storage with additional security"""
    
    def __init__(self, location=None, base_url=None):
        if location is None:
            location = settings.SECURE_MEDIA_ROOT
        if base_url is None:
            base_url = settings.SECURE_MEDIA_URL
        super().__init__(location, base_url)
    
    def _save(self, name, content):
        """Save file with additional security checks"""
        
        # Sanitize filename
        name = self.sanitize_filename(name)
        
        # Check file size
        if content.size > settings.MAX_FILE_SIZE:
            raise ValueError('File too large')
        
        return super()._save(name, content)
    
    def sanitize_filename(self, filename):
        """Sanitize filename for security"""
        # Remove dangerous characters
        filename = re.sub(r'[^\w\-_\.]', '_', filename)
        
        # Limit length
        name, ext = os.path.splitext(filename)
        if len(name) > 100:
            name = name[:100]
        
        return name + ext

# Use custom storage
class Document(BinderModel):
    file = BinderFileField(
        upload_to='secure/',
        storage=SecureFileStorage()
    )
```

## Testing File Handling

### File Upload Tests

Test file upload functionality:

```python
from django.test import TestCase
from django.core.files.uploadedfile import SimpleUploadedFile
from django.contrib.auth.models import User

class FileUploadTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
        self.client.force_login(self.user)
    
    def test_file_upload(self):
        """Test basic file upload"""
        # Create test file
        test_file = SimpleUploadedFile(
            "test.txt",
            b"file content",
            content_type="text/plain"
        )
        
        # Create document
        response = self.client.post('/api/document/', {
            'title': 'Test Document',
            'file': test_file
        })
        
        self.assertEqual(response.status_code, 200)
        
        # Verify file was saved
        document = Document.objects.get(title='Test Document')
        self.assertTrue(document.file)
        self.assertEqual(document.file.name.split('/')[-1], 'test.txt')
    
    def test_image_upload(self):
        """Test image upload with processing"""
        from PIL import Image
        import io
        
        # Create test image
        img = Image.new('RGB', (100, 100), color='red')
        img_io = io.BytesIO()
        img.save(img_io, format='JPEG')
        img_io.seek(0)
        
        test_image = SimpleUploadedFile(
            "test.jpg",
            img_io.read(),
            content_type="image/jpeg"
        )
        
        # Upload image
        response = self.client.post('/api/photo/', {
            'title': 'Test Photo',
            'image': test_image
        })
        
        self.assertEqual(response.status_code, 200)
        
        # Verify image was processed
        photo = Photo.objects.get(title='Test Photo')
        self.assertTrue(photo.image)
        self.assertTrue(photo.thumbnail)  # Should be auto-generated
```

This comprehensive file handling system provides secure, efficient file management with extensive customization options.
