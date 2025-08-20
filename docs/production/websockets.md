# WebSocket Support

Django Binder provides real-time communication capabilities through WebSocket integration, enabling live updates, notifications, and collaborative features in your applications.

## WebSocket Setup

### Install Dependencies

Install required packages for WebSocket support:

```bash
pip install channels
pip install channels-redis  # For Redis channel layer
pip install pika  # For RabbitMQ integration
```

### Configure Channels

Add Channels to your Django settings:

```python
# settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # Channels for WebSocket support
    'channels',
    
    # Django Binder
    'binder',
    
    # Your apps
    'your_app',
]

# ASGI application
ASGI_APPLICATION = 'your_project.asgi.application'

# Channel layer configuration
CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('127.0.0.1', 6379)],
        },
    },
}
```

### ASGI Configuration

Create or update your ASGI configuration:

```python
# asgi.py
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from binder.websocket import websocket_urlpatterns

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'your_project.settings')

application = ProtocolTypeRouter({
    'http': get_asgi_application(),
    'websocket': AuthMiddlewareStack(
        URLRouter(
            websocket_urlpatterns
        )
    ),
})
```

## Room-Based Messaging

### User Room Registration

Django Binder uses a room-based messaging system where users join rooms to receive relevant updates:

```python
from binder.websocket import register_user_room

# Register user for specific rooms
def register_user_rooms(user):
    """Register user for relevant rooms"""
    rooms = []
    
    # Global notifications room
    rooms.append('notifications')
    
    # User-specific room
    rooms.append(f'user_{user.id}')
    
    # Department-specific room
    if hasattr(user, 'profile') and user.profile.department:
        rooms.append(f'department_{user.profile.department}')
    
    # Project-specific rooms
    for project in user.projects.all():
        rooms.append(f'project_{project.id}')
    
    return rooms

# In your view or signal handler
rooms = register_user_rooms(request.user)
for room in rooms:
    register_user_room(request.user, room)
```

### Room Management

Manage user room memberships:

```python
from binder.websocket import register_user_room, unregister_user_room, get_user_rooms

class UserRoomManager:
    @staticmethod
    def join_room(user, room_name):
        """Add user to a room"""
        register_user_room(user, room_name)
    
    @staticmethod
    def leave_room(user, room_name):
        """Remove user from a room"""
        unregister_user_room(user, room_name)
    
    @staticmethod
    def get_user_rooms(user):
        """Get all rooms user is in"""
        return get_user_rooms(user)
    
    @staticmethod
    def update_user_rooms(user):
        """Update user's room memberships based on current permissions"""
        # Clear existing rooms
        current_rooms = get_user_rooms(user)
        for room in current_rooms:
            unregister_user_room(user, room)
        
        # Register for new rooms
        new_rooms = register_user_rooms(user)
        for room in new_rooms:
            register_user_room(user, room)
```

## Real-Time Model Updates

### Automatic Model Notifications

Send WebSocket notifications when models change:

```python
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from binder.websocket import send_to_room

@receiver(post_save, sender=Article)
def article_saved(sender, instance, created, **kwargs):
    """Send WebSocket notification when article is saved"""
    
    # Determine the type of change
    change_type = 'created' if created else 'updated'
    
    # Prepare notification data
    notification = {
        'type': 'model_change',
        'model': 'article',
        'action': change_type,
        'object_id': instance.id,
        'data': {
            'id': instance.id,
            'title': instance.title,
            'published': instance.published,
            'author': instance.author.username,
        }
    }
    
    # Send to relevant rooms
    rooms = []
    
    # Send to all users room for published articles
    if instance.published:
        rooms.append('public_articles')
    
    # Send to author's room
    rooms.append(f'user_{instance.author.id}')
    
    # Send to category subscribers
    if instance.category:
        rooms.append(f'category_{instance.category.id}')
    
    # Send notifications
    for room in rooms:
        send_to_room(room, notification)

@receiver(post_delete, sender=Article)
def article_deleted(sender, instance, **kwargs):
    """Send WebSocket notification when article is deleted"""
    
    notification = {
        'type': 'model_change',
        'model': 'article',
        'action': 'deleted',
        'object_id': instance.id,
        'data': {
            'id': instance.id,
            'title': instance.title,
        }
    }
    
    # Send to relevant rooms
    send_to_room('public_articles', notification)
    send_to_room(f'user_{instance.author.id}', notification)
```

### Custom WebSocket Endpoints

Create custom WebSocket endpoints for specific functionality:

```python
from binder.router import detail_route
from binder.websocket import send_to_room

class ArticleView(ModelView):
    model = Article
    
    @detail_route(name='publish', methods=['POST'])
    def publish_article(self, request, pk):
        """Publish article and send real-time notification"""
        article = self.get_object(pk)
        
        # Update article
        article.published = True
        article.published_at = timezone.now()
        article.save()
        
        # Send WebSocket notification
        notification = {
            'type': 'article_published',
            'article_id': article.id,
            'title': article.title,
            'author': article.author.username,
            'published_at': article.published_at.isoformat(),
        }
        
        # Notify subscribers
        send_to_room('article_updates', notification)
        send_to_room(f'author_{article.author.id}', notification)
        
        return JsonResponse({
            'success': True,
            'message': 'Article published and notifications sent'
        })
    
    @detail_route(name='subscribe', methods=['POST'])
    def subscribe_to_article(self, request, pk):
        """Subscribe user to article updates"""
        article = self.get_object(pk)
        room_name = f'article_{article.id}'
        
        # Add user to article-specific room
        register_user_room(request.user, room_name)
        
        # Send confirmation
        notification = {
            'type': 'subscription_confirmed',
            'message': f'Subscribed to updates for "{article.title}"'
        }
        send_to_room(f'user_{request.user.id}', notification)
        
        return JsonResponse({
            'success': True,
            'subscribed_to': room_name
        })
```

## RabbitMQ Integration

### RabbitMQ Configuration

Configure RabbitMQ for WebSocket messaging:

```python
# settings.py
RABBITMQ_CONFIG = {
    'host': 'localhost',
    'port': 5672,
    'username': 'guest',
    'password': 'guest',
    'virtual_host': '/',
    'exchange': 'binder_websocket',
    'routing_key': 'websocket.message',
}

# Enable RabbitMQ WebSocket integration
BINDER_WEBSOCKET_RABBITMQ = True
```

### RabbitMQ Message Publisher

Send messages through RabbitMQ:

```python
import pika
import json
from django.conf import settings

class RabbitMQPublisher:
    def __init__(self):
        self.config = settings.RABBITMQ_CONFIG
        self.connection = None
        self.channel = None
    
    def connect(self):
        """Establish RabbitMQ connection"""
        credentials = pika.PlainCredentials(
            self.config['username'],
            self.config['password']
        )
        
        parameters = pika.ConnectionParameters(
            host=self.config['host'],
            port=self.config['port'],
            virtual_host=self.config['virtual_host'],
            credentials=credentials
        )
        
        self.connection = pika.BlockingConnection(parameters)
        self.channel = self.connection.channel()
        
        # Declare exchange
        self.channel.exchange_declare(
            exchange=self.config['exchange'],
            exchange_type='topic',
            durable=True
        )
    
    def publish_message(self, room, message):
        """Publish message to RabbitMQ"""
        if not self.connection or self.connection.is_closed:
            self.connect()
        
        # Prepare message
        message_body = json.dumps({
            'room': room,
            'message': message,
            'timestamp': timezone.now().isoformat(),
        })
        
        # Publish message
        self.channel.basic_publish(
            exchange=self.config['exchange'],
            routing_key=f"room.{room}",
            body=message_body,
            properties=pika.BasicProperties(
                delivery_mode=2,  # Make message persistent
            )
        )
    
    def close(self):
        """Close RabbitMQ connection"""
        if self.connection and not self.connection.is_closed:
            self.connection.close()

# Usage
publisher = RabbitMQPublisher()
publisher.publish_message('notifications', {
    'type': 'system_alert',
    'message': 'System maintenance in 10 minutes'
})
publisher.close()
```

## Frontend WebSocket Integration

### JavaScript WebSocket Client

Connect to WebSocket from frontend:

```javascript
class BinderWebSocket {
    constructor(wsUrl, authToken) {
        this.wsUrl = wsUrl;
        this.authToken = authToken;
        this.socket = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
        this.reconnectDelay = 1000;
        this.messageHandlers = new Map();
    }
    
    connect() {
        // Create WebSocket connection
        this.socket = new WebSocket(`${this.wsUrl}?token=${this.authToken}`);
        
        this.socket.onopen = (event) => {
            console.log('WebSocket connected');
            this.reconnectAttempts = 0;
            this.onConnected(event);
        };
        
        this.socket.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.handleMessage(data);
        };
        
        this.socket.onclose = (event) => {
            console.log('WebSocket disconnected');
            this.onDisconnected(event);
            this.attemptReconnect();
        };
        
        this.socket.onerror = (error) => {
            console.error('WebSocket error:', error);
            this.onError(error);
        };
    }
    
    handleMessage(data) {
        const { type, ...payload } = data;
        
        // Call registered handlers
        if (this.messageHandlers.has(type)) {
            const handlers = this.messageHandlers.get(type);
            handlers.forEach(handler => handler(payload));
        }
        
        // Default handlers
        switch (type) {
            case 'model_change':
                this.handleModelChange(payload);
                break;
            case 'notification':
                this.handleNotification(payload);
                break;
            case 'user_message':
                this.handleUserMessage(payload);
                break;
        }
    }
    
    handleModelChange(payload) {
        const { model, action, object_id, data } = payload;
        
        // Emit custom event for model changes
        const event = new CustomEvent('binderModelChange', {
            detail: { model, action, object_id, data }
        });
        document.dispatchEvent(event);
    }
    
    handleNotification(payload) {
        // Show notification to user
        if ('Notification' in window && Notification.permission === 'granted') {
            new Notification(payload.title, {
                body: payload.message,
                icon: payload.icon || '/static/icon.png'
            });
        }
    }
    
    // Register message handler
    on(messageType, handler) {
        if (!this.messageHandlers.has(messageType)) {
            this.messageHandlers.set(messageType, []);
        }
        this.messageHandlers.get(messageType).push(handler);
    }
    
    // Send message
    send(message) {
        if (this.socket && this.socket.readyState === WebSocket.OPEN) {
            this.socket.send(JSON.stringify(message));
        }
    }
    
    // Join room
    joinRoom(roomName) {
        this.send({
            type: 'join_room',
            room: roomName
        });
    }
    
    // Leave room
    leaveRoom(roomName) {
        this.send({
            type: 'leave_room',
            room: roomName
        });
    }
    
    attemptReconnect() {
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
            this.reconnectAttempts++;
            setTimeout(() => {
                console.log(`Reconnecting... (${this.reconnectAttempts}/${this.maxReconnectAttempts})`);
                this.connect();
            }, this.reconnectDelay * this.reconnectAttempts);
        }
    }
    
    disconnect() {
        if (this.socket) {
            this.socket.close();
        }
    }
}

// Usage
const ws = new BinderWebSocket('ws://localhost:8000/ws/', 'your-auth-token');

// Register handlers
ws.on('article_published', (data) => {
    console.log('New article published:', data.title);
    // Update UI
});

ws.on('comment_added', (data) => {
    console.log('New comment on article:', data.article_id);
    // Add comment to UI
});

// Connect
ws.connect();

// Join specific rooms
ws.joinRoom('article_updates');
ws.joinRoom('user_notifications');
```

### React Integration

Integrate WebSocket with React:

```jsx
import React, { useEffect, useState, useContext } from 'react';

// WebSocket Context
const WebSocketContext = React.createContext();

export const WebSocketProvider = ({ children, wsUrl, authToken }) => {
    const [socket, setSocket] = useState(null);
    const [connected, setConnected] = useState(false);
    const [messages, setMessages] = useState([]);
    
    useEffect(() => {
        const ws = new BinderWebSocket(wsUrl, authToken);
        
        ws.onConnected = () => setConnected(true);
        ws.onDisconnected = () => setConnected(false);
        
        // Store all messages
        ws.on('*', (data) => {
            setMessages(prev => [...prev, data]);
        });
        
        ws.connect();
        setSocket(ws);
        
        return () => {
            ws.disconnect();
        };
    }, [wsUrl, authToken]);
    
    return (
        <WebSocketContext.Provider value={{ socket, connected, messages }}>
            {children}
        </WebSocketContext.Provider>
    );
};

// Hook to use WebSocket
export const useWebSocket = () => {
    const context = useContext(WebSocketContext);
    if (!context) {
        throw new Error('useWebSocket must be used within WebSocketProvider');
    }
    return context;
};

// Component using WebSocket
const ArticleList = () => {
    const { socket } = useWebSocket();
    const [articles, setArticles] = useState([]);
    
    useEffect(() => {
        if (socket) {
            // Listen for article changes
            socket.on('model_change', (data) => {
                if (data.model === 'article') {
                    if (data.action === 'created') {
                        setArticles(prev => [data.data, ...prev]);
                    } else if (data.action === 'updated') {
                        setArticles(prev => prev.map(article => 
                            article.id === data.object_id ? { ...article, ...data.data } : article
                        ));
                    } else if (data.action === 'deleted') {
                        setArticles(prev => prev.filter(article => article.id !== data.object_id));
                    }
                }
            });
            
            // Join article updates room
            socket.joinRoom('article_updates');
        }
    }, [socket]);
    
    return (
        <div>
            <h2>Articles (Real-time)</h2>
            {articles.map(article => (
                <div key={article.id}>
                    <h3>{article.title}</h3>
                    <p>Status: {article.published ? 'Published' : 'Draft'}</p>
                </div>
            ))}
        </div>
    );
};
```

## WebSocket Security

### Authentication

Secure WebSocket connections with authentication:

```python
# websocket_auth.py
from channels.auth import AuthMiddlewareStack
from channels.db import database_sync_to_async
from django.contrib.auth.models import AnonymousUser
from django.contrib.auth import get_user_model
from rest_framework.authtoken.models import Token

User = get_user_model()

@database_sync_to_async
def get_user_from_token(token_key):
    try:
        token = Token.objects.select_related('user').get(key=token_key)
        return token.user
    except Token.DoesNotExist:
        return AnonymousUser()

class TokenAuthMiddleware:
    def __init__(self, inner):
        self.inner = inner
    
    async def __call__(self, scope, receive, send):
        # Get token from query string
        query_string = scope.get('query_string', b'').decode()
        token = None
        
        for param in query_string.split('&'):
            if param.startswith('token='):
                token = param.split('=')[1]
                break
        
        # Authenticate user
        if token:
            scope['user'] = await get_user_from_token(token)
        else:
            scope['user'] = AnonymousUser()
        
        return await self.inner(scope, receive, send)

# Use in ASGI configuration
application = ProtocolTypeRouter({
    'websocket': TokenAuthMiddleware(
        URLRouter(websocket_urlpatterns)
    ),
})
```

### Permission Checking

Check permissions for WebSocket operations:

```python
from binder.websocket import WebSocketConsumer

class SecureWebSocketConsumer(WebSocketConsumer):
    async def connect(self):
        # Check if user is authenticated
        if self.scope['user'].is_anonymous:
            await self.close()
            return
        
        await self.accept()
    
    async def join_room(self, event):
        room_name = event['room']
        
        # Check if user can join this room
        if not await self.can_join_room(room_name):
            await self.send_json({
                'type': 'error',
                'message': 'Permission denied for room'
            })
            return
        
        await self.channel_layer.group_add(room_name, self.channel_name)
    
    async def can_join_room(self, room_name):
        """Check if user can join the specified room"""
        user = self.scope['user']
        
        # Public rooms
        if room_name in ['notifications', 'public_articles']:
            return True
        
        # User-specific rooms
        if room_name == f'user_{user.id}':
            return True
        
        # Department rooms
        if room_name.startswith('department_'):
            dept_id = room_name.split('_')[1]
            return hasattr(user, 'profile') and str(user.profile.department_id) == dept_id
        
        # Project rooms
        if room_name.startswith('project_'):
            project_id = room_name.split('_')[1]
            return user.projects.filter(id=project_id).exists()
        
        return False
```

## Testing WebSocket Functionality

### WebSocket Tests

Test WebSocket functionality:

```python
from channels.testing import WebsocketCommunicator
from django.test import TestCase
from django.contrib.auth.models import User

class WebSocketTest(TestCase):
    def setUp(self):
        self.user = User.objects.create_user('testuser', 'test@example.com', 'password')
    
    async def test_websocket_connection(self):
        """Test WebSocket connection"""
        communicator = WebsocketCommunicator(
            application,
            f'/ws/?token={self.user.auth_token.key}'
        )
        
        connected, subprotocol = await communicator.connect()
        self.assertTrue(connected)
        
        await communicator.disconnect()
    
    async def test_room_joining(self):
        """Test joining rooms"""
        communicator = WebsocketCommunicator(
            application,
            f'/ws/?token={self.user.auth_token.key}'
        )
        
        await communicator.connect()
        
        # Join room
        await communicator.send_json_to({
            'type': 'join_room',
            'room': 'test_room'
        })
        
        # Should receive confirmation
        response = await communicator.receive_json_from()
        self.assertEqual(response['type'], 'room_joined')
        
        await communicator.disconnect()
    
    async def test_message_broadcasting(self):
        """Test message broadcasting to rooms"""
        # Create two communicators
        comm1 = WebsocketCommunicator(application, f'/ws/?token={self.user.auth_token.key}')
        comm2 = WebsocketCommunicator(application, f'/ws/?token={self.user.auth_token.key}')
        
        await comm1.connect()
        await comm2.connect()
        
        # Both join same room
        await comm1.send_json_to({'type': 'join_room', 'room': 'test_room'})
        await comm2.send_json_to({'type': 'join_room', 'room': 'test_room'})
        
        # Send message to room
        from binder.websocket import send_to_room
        send_to_room('test_room', {'type': 'test_message', 'content': 'Hello'})
        
        # Both should receive message
        msg1 = await comm1.receive_json_from()
        msg2 = await comm2.receive_json_from()
        
        self.assertEqual(msg1['type'], 'test_message')
        self.assertEqual(msg2['type'], 'test_message')
        
        await comm1.disconnect()
        await comm2.disconnect()
```

This comprehensive WebSocket system enables real-time features while maintaining security and scalability.
