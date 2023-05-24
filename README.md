# multiple-upload-files

Como fazer upload de varíos arquivos no django.

<details><summary><b>Criar Modelo (multipload)</b></summary>

- **Criar Modelo (multipload)**
    
    No nosso *myapp/models.py* vamos criar um modelo.
    
    **ProductImage** é modelo onde vamos salvar todas as imagens. E está relacionado com Product.
    
    ```python
    from django.db import models
    
    # Create your models here.
    class Product(models.Model):
        name = models.CharField(max_length=100)
        price = models.DecimalField(max_digits=8, decimal_places=2)
        description = models.TextField()
    
        def __str__(self):
            return self.name
    
    class ProductImage(models.Model):
        image = models.FileField('Arquivos',upload_to='image')
        product = models.ForeignKey(Product, related_name='products', on_delete=models.CASCADE)
     
        def __str__(self):
            return self.product.name
    ```
    
    *myapp/admin.py*
    
    ```python
    from django.contrib import admin
    
    from myapp.models import Product, ProductImage
    
    # Register your models here.
    admin.site.register(Product)
    admin.site.register(ProductImage)
    ```
    
    ```python
    python manage.py makemigrations
    python manage.py migrate
    
    python manage.py createsuperuser # para acessar django admin
    ```

</details>

<details><summary><b>Configurar forms.py (Campos Formulário)</b></summary>

- **Configurar forms.py (Campos Formulário)**
    
    Como vamos receber multiplos arquivos adicionei um widet no campo products da tabela **ProductImage**.
    
    Documentação:  `ClearableFileInput` serve para limpar o campo. 
    [https://docs.djangoproject.com/en/4.1/ref/forms/widgets/#django.forms.ClearableFileInput](https://docs.djangoproject.com/en/4.1/ref/forms/widgets/#django.forms.ClearableFileInput)
    
     
    
    Lembrando que “**products**” tem que ser mesmo nome que está no campo ***related_name.***
    
    `products = forms.FileField(widget=forms.ClearableFileInput(attrs={'multiple': True}))`
    
    *myapp/forms.py*
    
    ```python
    from django import forms
    from .models import Product, ProductImage
    
    class ProductForm(forms.ModelForm):
        products = forms.FileField(widget=forms.ClearableFileInput(attrs={'multiple': True}))
        class Meta:
            model = Product
            fields = ['name', 'price', 'description'] 
            
        def __init__(self, *args, **kwargs): # Adiciona 
            super().__init__(*args, **kwargs)  
            for field_name, field in self.fields.items():   
                  field.widget.attrs['class'] = 'form-control'
                  
    
    class ProductImageForm(forms.ModelForm):
        class Meta:
            model = ProductImage
            fields = ['image', 'product']
    ```
    
    *myapp/form-create.html*
    
    ```html
    {% extends 'base.html' %}
    
    {% block title %}Cadastrar Produto{% endblock %}
    
    {% block content %}
    <form action="{% url 'product-create' %}" method="post" enctype="multipart/form-data">
        {% csrf_token %}
        {{form}}
        <input type="submit" class="btn btn-primary" value="Salvar">
    </form>
    {% endblock %}
    ```

</details>

<details><summary><b>Configurar Views.py (Create)</b></summary>

- **Configurar Views.py (Create)**
    
    *myapp/views.py*
    
    ```python
    from django.shortcuts import redirect
    
    def form_product(request):
        form = ProductForm
        if request.method == 'POST':
            form = ProductForm(request.POST, request.FILES)
            if form.is_valid():
                product = form.save()
                files = request.FILES.getlist('products')
                if files:
                    for f in files:
                        ProductImage.objects.create(
                            product=product, 
                            image=f)
                return redirect('product-list')  
                     
        return render(request, 'form-create.html', {'form': form})
    ```
    
    *myapp/urls.py*
    
    ```python
    from django.urls import path 
    from myapp import views
    
    urlpatterns = [ 
        path('create-product/', views.form_product, name='product-create'), 
    ]
    ```
    
    Testar o formulário.
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bb5f2961-5342-4f21-943e-98d210c6a14c/Untitled.png)

</details>

<details><summary><b>Listar os Produtos</b></summary>
 
- **Listar os Produtos**
    
    *myapp/views.py*
    
    ```python
    from django.core.paginator import Paginator
    from django.shortcuts import redirect, render
    from myapp.forms import ProductForm
    
    from myapp.models import Product, ProductImage
    
    def product_list(request):
        obj = request.GET.get('obj')
        print(obj)
        if obj:  
            product_list = Product.objects.filter(name__icontains=obj)  
        else:
            product_list = Product.objects.all()   
            
        paginator = Paginator(product_list, 3) # mostra 3 produtos por pagina
        page_number = request.GET.get('page')
        page_obj = paginator.get_page(page_number)
        
        return render(request, 'index.html', {'page_obj': page_obj})
    ```
    
    *myapp/urls.py*
    
    ```python
    from django.urls import path 
    from myapp import views
    
    urlpatterns = [
        path('', views.product_list, name='product-list'), 
        path('create-product/', views.form_product, name='product-create'), 
    ]
    ```
    
    *myapp/index.html*
    
    ```python
    {% extends 'base.html' %}
    
    {% block title %}index 1{% endblock %}
    
    {% block content %}
    <h1>Produtos</h1>
    
    <div class="container">
    
        <form class="d-flex gap-3 mt-2" action="{% url 'product-list' %}" method="GET">          
     
            <span class="fw-bold">Pesquisar: </span>   
    
            <input name="obj" type="text" value="{{request.GET.obj}}" class="form-control" placeholder="pesquisar pelo nome do produto..."> 
        
            {% if request.GET.obj %}   
            <a class="btn btn-primary" href="{% url 'product-list' %}">Reset</a>                 
            {% endif %}  
        
            <button type="submit" class="btn btn-primary">Buscar</button> 
        
        </form> 
    
        <a class="btn btn-warning" href="{% url 'product-create' %}">+</a>
       
        <table class="table"> 
            <thead>
                <tr>
                    <th scope="col">#</th>
                    <th scope="col">Nome</th>
                    <th scope="col">Preço</th>
                    <th scope="col">Descrição</th>
                    <th scope="col">Imagens</th>
                </tr>
            </thead> 
            <tbody>
                {% for product in page_obj %}
                <tr>
                    <th scope="row">{{ product.id }}</th>
                    <th scope="row">{{ product.name|upper }}</th>
                    <th scope="row">{{ product.price }}</th>
                    <th scope="row">{{ product.description }}</th>
                    <th scope="row">
                        {% for el in product.products.all %} 
                        <a href="{{el.image.url}}" target="_blank">Link {{forloop.counter}}</a>
                        {% endfor %}
                    </th>
                </tr>
                {% endfor %}
            </tbody>
        </table> 
        {% include 'pagination.html' %} 
    </div> 
    {% endblock %}
    ```
    
    *myapp/pagination.html*
    
    ```html
    {% if page_obj.has_other_pages %}
    
    <div class="btn-group" role="group" aria-label="Item pagination">
    
        {% if page_obj.has_previous %}
    
            {% if request.GET.obj %}
            <a href="?page={{ page_obj.previous_page_number }}&obj={{request.GET.obj}}" class="btn btn-outline-primary">&laquo;</a> 
            {% else %}
            <a href="?page={{ page_obj.previous_page_number }}" class="btn btn-outline-primary">&laquo;</a> 
            {% endif %} 
    
        {% endif %}
    
        {% for page_number in page_obj.paginator.page_range %}
    
            {% if request.GET.obj %}   
          
                {% if page_obj.number == page_number %}
                    <button class="btn btn-outline-primary active">
                        <span>{{ page_number }} <span class="sr-only">(Atual)</span></span>
                    </button>
                {% else %}
                    <a href="?page={{ page_number }}&obj={{request.GET.obj}}" class="btn btn-outline-primary">
                        {{ page_number }}
                    </a>
                {% endif %}
    
            {% else %}  
    
                {% if page_obj.number == page_number %}
                <button class="btn btn-outline-primary active">
                    <span>{{ page_number }} <span class="sr-only">(Atual)</span></span>
                </button>
                {% else %}
                    <a href="?page={{ page_number }}" class="btn btn-outline-primary">
                        {{ page_number }}
                    </a>
                {% endif %}
            
    
            {% endif %}  
    
        {% endfor %}
    
        {% if page_obj.has_next %}
            {% if request.GET.obj %}
            <a href="?page={{ page_obj.next_page_number }}&obj={{request.GET.obj}}" class="btn btn-outline-primary">&raquo;</a>
            {% else %}
            <a href="?page={{ page_obj.next_page_number }}" class="btn btn-outline-primary">&raquo;</a>
            {% endif %}    
        {% endif %}
    
    </div>
    {% endif %}
    ``` 

</details>

DJANGO 4.2 ATUALIZAÇÃO
https://github.com/djangomy/multiple-upload-files/tree/multiple-upload-files-Django42
