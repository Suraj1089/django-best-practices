## views-fbv-vs-cbv

### Why it matters
Standardizing on when to use Class-Based Views (CBVs) or Function-Based Views (FBVs) prevents unreadable views. CBVs excel at handling generic patterns (list, detail, create). FBVs excel at explicit complex procedural logic.

### ❌ Wrong
```python
# Re-inventing the wheel with an FBV for standard generic operations
def post_list(request):
 posts = Post.objects.all()
 paginator = Paginator(posts, 10)
 page_obj = paginator.get_page(request.GET.get('page'))
 return render(request, 'post_list.html', {'page_obj': page_obj})

# Using a CBV for a highly complex custom procedural task
class GenerateReportView(View):
 def get(self, request, *args, **kwargs):
 # 100 lines of complex procedural math and file generation
 # Better suited for an FBV
 pass
```

### ✅ Right
```python
# Use generic CBVs directly for CRUD 
class PostListView(ListView):
 model = Post
 paginate_by = 10
 template_name = 'post_list.html'

# Use FBVs directly for distinct, procedural customized actions
@require_POST
def generate_report(request):
 data = build_report_data()
 return JsonResponse(data)
```

### Notes
- Stick to `django.views.generic` exclusively for simple CRUD.
- Fall back strictly to FBVs for anything procedural.

---

## api-drf-serializers

### Why it matters
Putting custom data validation inside API views severely violates DRY protocols. DRF provides Serializers to securely validate data. Keep validation in serializers, not views.

### ❌ Wrong
```python
@api_view(['POST'])
def create_customer(request):
 # View contains validation logic
 if 'email' not in request.data:
 return Response({'error': 'Email required'}, status=400)
 if not '@' in request.data['email']:
 return Response({'error': 'Invalid email'}, status=400)
 
 Customer.objects.create(**request.data)
 return Response({'status': 'ok'})
```

### ✅ Right
```python
class CustomerSerializer(serializers.ModelSerializer):
 class Meta:
 model = Customer
 fields = ['email', 'name']

 def validate_email(self, value):
 if '@' not in value:
 raise serializers.ValidationError("Invalid email")
 return value

@api_view(['POST'])
def create_customer(request):
 serializer = CustomerSerializer(data=request.data)
 serializer.is_valid(raise_exception=True)
 serializer.save()
 return Response({'status': 'ok'})
```

### Notes
- Use `raise_exception=True` on `.is_valid()` to completely avoid `if` statements in views.

---

## api-viewset-optimization

### Why it matters
DRF ModelViewSets directly evaluate `.queryset` automatically. Returning a naive queryset triggers N+1 problems dramatically whenever a serializer tries to map foreign keys.

### ❌ Wrong
```python
class AuthorViewSet(viewsets.ModelViewSet):
 # Serializer automatically accesses author.books, triggering N+1 queries
 queryset = Author.objects.all()
 serializer_class = AuthorSerializer
```

### ✅ Right
```python
class AuthorViewSet(viewsets.ModelViewSet):
 # Optimize the queryset easily 
 queryset = Author.objects.prefetch_related('books').all()
 serializer_class = AuthorSerializer
```

### Notes
- Pre-fetch relations securely directly inside the `queryset` attribute.

---

## api-pagination

### Why it matters
Returning all items from a single model effectively overloads servers and network transmission during growth.

### ❌ Wrong
```python
class ProductViewSet(viewsets.ModelViewSet):
 queryset = Product.objects.all()
 serializer_class = ProductSerializer
 pagination_class = None # DANGEROUS!
```

### ✅ Right
```python
# In settings.py:
REST_FRAMEWORK = {
 'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
 'PAGE_SIZE': 50
}
```

### Notes
- NEVER return unpaginated lists.
