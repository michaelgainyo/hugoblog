---
title: 'Dynamic Serialization With Django Rest Framework'
date: 2023-01-09T23:18:45+01:00
tags: ['drf', 'django', 'python']
description: 'How to create a drf serializer class dynamically using the type function'
# draft: true
---

Suppose we have two django models, `Product` and `Size`, defined as follows:

```py
class Product(models.Model):
    name = models.CharField(max_length=99)
    description = models.TextField(null=True, blank=True)

class Size(models.Model):
    product = models.ForeignKey('Product', related_name='sizes', on_delete=models.CASCADE)
    code = models.CharField(max_length=10)
    text = models.CharField(max_length=20)
    quantity = models.PositiveIntegerField(default=1)
```

On a product detail view page, we might want to return a JSON response similar to this:

```json
{
  "id": 1,
  "name": "Fur Flight Jacket",
  "description": "Jacket made of textured tear-resistant ripstop fabric",
  "sizes": [
    { "code": "xs", "text": "Extra-Small", "quantity": 15 },
    { "code": "s", "text": "Small", "quantity": 20 },
    { "code": "m", "text": "Medium", "quantity": 0 },
    { "code": "l", "text": "Large", "quantity": 10 },
    { "code": "xl", "text": "Extra-Large", "quantity": 5 }
  ]
}
```

This response includes not only the `id`, `name`, and `description` fields of a product, but also a list of its available sizes.

However, on a product list page, we might prefer a simpler response that only includes the id, name, and description fields, like this:

```json
[
  {
    "id": 1,
    "name": "Flight Jacket",
    "description": "Jacket made of textured tear-resistant ripstop fabric"
  },
  {
    "id": 2,
    "name": "Stretch Linen Dress",
    "description": "A dress made of stretch linen blend fabric. Halter neck with bead appliqué"
  },
  {
    "id": 3,
    "name": "Denim Jacket",
    "description": "Lapel collar jacket with long sleeves with buttoned cuffs"
  }
]
```

The easiest, and probably recommended way to handle these two different responses, is to create two separate serializers, `ProductListSerializer` and `ProductDetailSerializer`.
For example, ProductListSerializer would be defined like this:

```py
class ProductListSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields=('id','name', 'description')
```

ProductDetailSerializer would be defined like this:

```py
class ProductDetailSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields=('id','name', 'description','sizes')

    sizes = SizeSerializer(many=True, read_only=True)
```

Note that ProductDetailSerializer includes the sizes field, which is a list of Size objects related to the Product. Finally, SizeSerializer would look like this:

```py
class SizeSerializer(serializers.ModelSerializer):
    class Meta:
        model = Size
        fields = ('code','text', 'quantity')

```

Alternatively, we can create these serializers dynamically by taking advantage of the `type` function which allows you to create a new class dynamically by specifying its name, base classes, and attributes. For example,

```py

def get_product_serializer_class(*, fields:tuple=()):
    fields = fields + ('id', 'name')
    attrs = {'Meta': type('Meta', (), {'model': Product,'fields': fields,})}
    if "sizes" in fields:
        attrs['sizes'] = SizeSerializer(many=True, read_only=True)
    return type('ProductSerializer', (serializers.ModelSerializer,), attrs)

```

This function takes a tuple of fields as an argument and returns a new serializer class with those fields included. For example, we could redefine our previous serializers as such:

```py
ProductListSerializer = get_product_serializer_class(fields=('description',))
ProductDetailSerializer = get_product_serializer_class(fields=('description', 'sizes'))

```

To use either of these serializers, we simply instantiate the class and pass it a Product instance. For example:

```py
product = Product.objects.get(id=1)
print(ProductDetailSerializer(product).data)

```

We can also use it in a view or viewset as such:

```py
class ProductViewset(viewsets.ModelViewSet):
    queryset = Product.objects.active()
    serializer_class = ProductSerializer

    def get_serializer_class(self):
        if self.action == 'list':
            return ProductListSerializer

        return ProductSerializer

```

Or maybe, you want to select the fields based on a request's query parameters

```py
class ProductViewset(viewsets.ModelViewSet):
    queryset = Product.objects.active()
    serializer_class = ProductSerializer

    def get_serializer_class(self):
        # getattr(request, 'query_params', {})
        fields = set()
        for item in self.request.query_params.getlist('fields'):
            fields.update(item.split(','))

        return get_product_serializer_class(fields=fields)

```

Keep in mind that using this approach has some disadvantages, such as performance overhead and increased complexity, so it should be used with caution. However, it allows us to create a product serializer for any set of fields we need without having to define a separate class for each combination of fields.
