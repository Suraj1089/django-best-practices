## logging-structured

### Why it matters
Calling `print()` dumps unsearchable unstructured text into stdout. When debugging scale apps across 10 servers, you need centralized searchable logs.

### ❌ Wrong
```python
def process_webhook(request):
 print("Received webhook request from Stripe") # Completely unsearchable in production
 print(f"User UUID: {request.user.uuid}")
 
 try:
 data = request.json()
 except Exception as e:
 # Fails silently in standard output buffers 
 print(e)
```

### ✅ Right
```python
import logging

# Always instantiate a named logger directly
logger = logging.getLogger(__name__)

def process_webhook(request):
 logger.info("Received Stripe webhook", extra={"user_uuid": str(request.user.uuid)})
 
 try:
 data = request.json()
 except Exception as e:
 # Pushes exact stacktraces smoothly to Datadog/Sentry 
 logger.exception("Failed to parse Stripe JSON payload", exc_info=e)
```

### Notes
- Configure `LOGGING` structurally in `settings.py` and map it to a JSON formatter.
- Rely on `logger.exception()` directly inside except blocks to capture the entire traceback perfectly directly safely.
