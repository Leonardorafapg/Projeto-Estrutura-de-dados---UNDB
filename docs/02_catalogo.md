# Módulo 02 — Catálogo de Produtos

## Visão Geral

| Item | Detalhe |
|---|---|
| App Django | `catalogo` |
| Estrutura de dados | HashSet + HashMap (set + dict Python / unique constraint) |
| Complexidade antes | O(n) para detectar duplicata |
| Complexidade depois | O(1) automático |

---

## Model

**Arquivo:** `catalogo/models.py`

```python
from django.db import models

class Produto(models.Model):
    sku       = models.CharField(max_length=50, unique=True, db_index=True)
    nome      = models.CharField(max_length=200)
    preco     = models.DecimalField(max_digits=10, decimal_places=2)
    estoque   = models.PositiveIntegerField(default=0)
    categoria = models.CharField(max_length=100)
    ativo     = models.BooleanField(default=True)
    criado_em = models.DateTimeField(auto_now_add=True)
    atualizado = models.DateTimeField(auto_now=True)

    class Meta:
        indexes = [
            models.Index(fields=["sku"]),
            models.Index(fields=["categoria", "ativo"]),
        ]
        constraints = [
            # Garante unicidade de nome dentro de uma mesma categoria
            models.UniqueConstraint(
                fields=["nome", "categoria"],
                name="produto_unico_por_categoria"
            )
        ]

    def __str__(self):
        return f"[{self.sku}] {self.nome}"
```

---

## Serializer

**Arquivo:** `catalogo/serializers.py`

```python
from rest_framework import serializers
from .models import Produto

class ProdutoSerializer(serializers.ModelSerializer):
    class Meta:
        model  = Produto
        fields = ["id", "sku", "nome", "preco", "estoque",
                  "categoria", "ativo", "criado_em"]

    def validate_preco(self, value):
        if value <= 0:
            raise serializers.ValidationError("Preço deve ser maior que zero.")
        return value

    def validate_sku(self, value):
        if not value.strip():
            raise serializers.ValidationError("SKU não pode ser vazio.")
        return value.upper()


class AjusteEstoqueSerializer(serializers.Serializer):
    delta = serializers.IntegerField(
        help_text="Positivo para entrada, negativo para saída."
    )

    def validate_delta(self, value):
        if value == 0:
            raise serializers.ValidationError("Delta não pode ser zero.")
        return value
```

---

## Views

**Arquivo:** `catalogo/views.py`

```python
from django.db import IntegrityError
from django.db.models import F
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import Produto
from .serializers import ProdutoSerializer, AjusteEstoqueSerializer

# HashSet em memória — controle de SKUs cadastrados O(1)
_skus_cadastrados = set()


class ProdutoListView(APIView):
    """Lista todos os produtos e cadastra novo produto."""

    def get(self, request):
        categoria = request.query_params.get("categoria")
        qs = Produto.objects.filter(ativo=True)
        if categoria:
            qs = qs.filter(categoria=categoria)
        return Response(ProdutoSerializer(qs, many=True).data)

    def post(self, request):
        serializer = ProdutoSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        sku = serializer.validated_data["sku"]

        # Verifica duplicata no HashSet — O(1)
        if sku in _skus_cadastrados:
            return Response(
                {"erro": "SKU já cadastrado.", "sku": sku},
                status=status.HTTP_409_CONFLICT
            )

        try:
            produto = serializer.save()
            _skus_cadastrados.add(sku)  # adiciona ao set — O(1)
            return Response(ProdutoSerializer(produto).data, status=status.HTTP_201_CREATED)
        except IntegrityError:
            return Response(
                {"erro": "Produto duplicado. Verifique nome e categoria."},
                status=status.HTTP_409_CONFLICT
            )


class ProdutoDetalheView(APIView):
    """Busca, atualiza e desativa produto por SKU."""

    def get(self, request, sku):
        try:
            produto = Produto.objects.get(sku=sku)
        except Produto.DoesNotExist:
            return Response({"erro": "Produto não encontrado."}, status=status.HTTP_404_NOT_FOUND)
        return Response(ProdutoSerializer(produto).data)

    def put(self, request, sku):
        try:
            produto = Produto.objects.get(sku=sku)
        except Produto.DoesNotExist:
            return Response({"erro": "Produto não encontrado."}, status=status.HTTP_404_NOT_FOUND)

        serializer = ProdutoSerializer(produto, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, sku):
        # Soft delete — mantém histórico
        updated = Produto.objects.filter(sku=sku).update(ativo=False)
        if not updated:
            return Response({"erro": "Produto não encontrado."}, status=status.HTTP_404_NOT_FOUND)
        _skus_cadastrados.discard(sku)
        return Response(status=status.HTTP_204_NO_CONTENT)


class AjusteEstoqueView(APIView):
    """Ajuste atômico de estoque — entrada ou saída."""

    def patch(self, request, sku):
        serializer = AjusteEstoqueSerializer(data=request.data)
        if not serializer.is_valid():
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        delta = serializer.validated_data["delta"]

        # F() garante operação atômica no banco — sem race condition
        updated = Produto.objects.filter(sku=sku, ativo=True).update(
            estoque=F("estoque") + delta
        )
        if not updated:
            return Response({"erro": "Produto não encontrado."}, status=status.HTTP_404_NOT_FOUND)

        estoque_atual = Produto.objects.get(sku=sku).estoque
        if estoque_atual < 0:
            # Reverte se ficou negativo
            Produto.objects.filter(sku=sku).update(estoque=F("estoque") - delta)
            return Response({"erro": "Estoque insuficiente."}, status=status.HTTP_400_BAD_REQUEST)

        return Response({"sku": sku, "estoque": estoque_atual})
```

---

## URLs

**Arquivo:** `catalogo/urls.py`

```python
from django.urls import path
from .views import ProdutoListView, ProdutoDetalheView, AjusteEstoqueView

urlpatterns = [
    path("", ProdutoListView.as_view()),                        # GET, POST
    path("<str:sku>/", ProdutoDetalheView.as_view()),           # GET, PUT, DELETE
    path("<str:sku>/estoque/", AjusteEstoqueView.as_view()),    # PATCH
]
```

**Registro em** `hublog/urls.py`:

```python
path("api/produtos/", include("catalogo.urls")),
```

---

## Endpoints

| Método | URL | Descrição |
|---|---|---|
| GET | `/api/produtos/` | Lista produtos ativos |
| GET | `/api/produtos/?categoria=frutas` | Filtra por categoria |
| POST | `/api/produtos/` | Cadastra produto — verifica dup. O(1) |
| GET | `/api/produtos/<sku>/` | Busca produto por SKU |
| PUT | `/api/produtos/<sku>/` | Atualiza produto |
| DELETE | `/api/produtos/<sku>/` | Desativa produto (soft delete) |
| PATCH | `/api/produtos/<sku>/estoque/` | Ajusta estoque (entrada/saída) |

---

## Exemplo de Payload

**POST /api/produtos/**
```json
{
  "sku": "FRU001",
  "nome": "Banana Orgânica",
  "preco": 8.90,
  "estoque": 100,
  "categoria": "frutas"
}
```

**Resposta 409 — duplicata:**
```json
{
  "erro": "SKU já cadastrado.",
  "sku": "FRU001"
}
```

**PATCH /api/produtos/FRU001/estoque/ — entrada de 50 unidades:**
```json
{ "delta": 50 }
```

**PATCH /api/produtos/FRU001/estoque/ — saída de 10 unidades:**
```json
{ "delta": -10 }
```

---

## Análise de Complexidade

| Operação | Antes | Depois |
|---|---|---|
| Verificar duplicata | O(n) | O(1) via HashSet |
| Inserção segura | O(n) | O(1) |
| Busca por SKU | O(n) | O(1) via índice |
| Ajuste de estoque | O(n) | O(1) atômico |
