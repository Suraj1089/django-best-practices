## channels-sync-to-async

### Why it matters
Django's ORM is synchronous. If you perform any database queries directly inside an asynchronous WebSocket Consumer (e.g., `AsyncWebsocketConsumer`), you will block the single asynchronous event loop. This severe mistake severely throttles all active WebSocket connections simultaneously.

### ❌ Wrong
```python
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
 async def receive(self, text_data):
 # Disastrous: Synchronous ORM call directly blocking the ASGI loop
 user = User.objects.get(username="test")
 
 await self.send(text_data=f"Hello, {user.username}")
```

### ✅ Right
```python
from channels.generic.websocket import AsyncWebsocketConsumer
from channels.db import database_sync_to_async

class ChatConsumer(AsyncWebsocketConsumer):
 async def receive(self, text_data):
 # Await the ORM query cleanly inside a separate threadpool
 user = await self.get_user("test")
 
 await self.send(text_data=f"Hello, {user.username}")
 
 # Decorate synchronous logic directly 
 @database_sync_to_async
 def get_user(self, username):
 return User.objects.get(username=username)
```

### Notes
- Any interaction with Django models from an `async def` function requires `database_sync_to_async`.

---

## channels-channel-layers

### Why it matters
Channel layers facilitate cross-process communication (via Redis). Sending massive python ORM objects directly crashes the serializer. Only raw valid JSON primitives efficiently traverse over the Redis channel.

### ❌ Wrong
```python
class PostNotifier(AsyncWebsocketConsumer):
 async def notify_post_created(self, post_instance):
 # Fails : You cannot send native Django ORM models across a channel.
 await self.channel_layer.group_send(
 'post_notifications',
 {
 'type': 'new_post',
 'post': post_instance # Unserializable 
 }
 )
```

### ✅ Right
```python
class PostNotifier(AsyncWebsocketConsumer):
 async def notify_post_created(self, post_id, title):
 await self.channel_layer.group_send(
 'post_notifications',
 {
 'type': 'new_post',
 # JSON-serializable primitives
 'post_id': post_id,
 'title': title
 }
 )
```

### Notes
- Send only `str`, `int`, `float`, `dict`, and `list`. 

---

## channels-security-auth

### Why it matters
WebSockets bypass normal view-level permissions. You must directly authenticate the ASGI connection.

### ❌ Wrong
```python
# Directly accepting all connections without authentication 
class PrivateDataConsumer(AsyncWebsocketConsumer):
 async def connect(self):
 await self.accept() # Accepts anonymous users
```

### ✅ Right
```python
class PrivateDataConsumer(AsyncWebsocketConsumer):
 async def connect(self):
 if self.scope["user"].is_anonymous:
 # Drop the connection for unauthenticated users
 await self.close()
 else:
 await self.accept()
```

### Notes
- Ensure `AuthMiddlewareStack` is correctly configured in your `asgi.py`.
