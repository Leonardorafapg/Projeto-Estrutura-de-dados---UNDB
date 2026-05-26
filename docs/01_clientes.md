# Módulo 01 — Cadastro de Clientes

## Visão Geral

| Item | Detalhe |
|---|---|
| App Django | `clientes` |
| Estrutura de dados | HashMap (dict Python / índice no banco) |
| Complexidade antes | O(n) |
| Complexidade depois | O(1) |

---

## Model

**Arquivo:** `clientes/models.py`

```python
from django.db import models

class Cliente(models.Model):
    cpf = models.CharField(
        max_length=11,
        unique=True,      # constraint de unicidade + índice automático
        db_index=True
    )
    nome      = models.CharField(max_length=200)
    email     = models.EmailField(unique=True)
    telefone  = models.CharField(max_length=20)
    criado_em = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=["cpf"]),
            models.Index(fields=["email"]),
        ]

    def __str__(self):
        return f"{self.nome} ({self.cpf})"
```

---

## Serializer

**Arquivo:** `clientes/serializers.py`

```python
from rest_framework import serializers
from .models import Cliente

class ClienteSerializer(serializers.ModelSerializer):
    class Meta:
        model  = Cliente
        fields = ["id", "cpf", "nome", "email", "telefone", "criado_em"]

    def validate_cpf(self, value):
        cpf = "".join(filter(str.isdigit, value))
        if len(cpf) != 11:
            raise serializers.ValidationError("CPF deve ter 11 dígitos.")
        return cpf
```

---

## Views

**Arquivo:** `clientes/views.py`

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import Cliente
from .serializers import ClienteSerializer

# HashMap em memória — busca O(1) para sessão ativa
_cache_clientes = {}

class ClienteListView(APIView):
    """Lista todos os clientes e cria novo cliente."""

    def get(self, request):
        clientes = Cliente.objects.all()
        serializer = ClienteSerializer(clientes, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = ClienteSerializer(data=request.data)
        if serializer.is_valid():
            cliente = serializer.save()
            # Popula o cache após criação
            _cache_clientes[cliente.cpf] = ClienteSerializer(cliente).data
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class ClienteDetalheView(APIView):
    """Busca, atualiza e remove cliente por CPF."""

    def get(self, request, cpf):
        # 1. Tenta o cache — O(1)
        if cpf in _cache_clientes:
            return Response({"source": "cache", "data": _cache_clientes[cpf]})

        # 2. Vai ao banco com índice — O(log n)
        try:
            cliente = Cliente.objects.get(cpf=cpf)
        except Cliente.DoesNotExist:
            return Response({"erro": "Cliente não encontrado."}, status=status.HTTP_404_NOT_FOUND)

        dados = ClienteSerializer(cliente).data
        _cache_clientes[cpf] = dados  # salva no cache
        return Response({"source": "db", "data": dados})

    def put(self, request, cpf):
        try:
            cliente = Cliente.objects.get(cpf=cpf)
        except Cliente.DoesNotExist:
            return Response({"erro": "Cliente não encontrado."}, status=status.HTTP_404_NOT_FOUND)

        serializer = ClienteSerializer(cliente, data=request.data)
        if serializer.is_valid():
            serializer.save()
            _cache_clientes[cpf] = serializer.data  # atualiza cache
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, cpf):
        try:
            cliente = Cliente.objects.get(cpf=cpf)
        except Cliente.DoesNotExist:
            return Response({"erro": "Cliente não encontrado."}, status=status.HTTP_404_NOT_FOUND)

        cliente.delete()
        _cache_clientes.pop(cpf, None)  # remove do cache
        return Response(status=status.HTTP_204_NO_CONTENT)
```

---

## URLs

**Arquivo:** `clientes/urls.py`

```python
from django.urls import path
from .views import ClienteListView, ClienteDetalheView

urlpatterns = [
    path("", ClienteListView.as_view()),          # GET, POST
    path("<str:cpf>/", ClienteDetalheView.as_view()),  # GET, PUT, DELETE
]
```

**Registro em** `hublog/urls.py`:

```python
from django.urls import path, include

urlpatterns = [
    path("api/clientes/", include("clientes.urls")),
]
```

---

## Endpoints

| Método | URL | Descrição |
|---|---|---|
| GET | `/api/clientes/` | Lista todos os clientes |
| POST | `/api/clientes/` | Cria novo cliente |
| GET | `/api/clientes/<cpf>/` | Busca cliente por CPF — O(1) |
| PUT | `/api/clientes/<cpf>/` | Atualiza cliente |
| DELETE | `/api/clientes/<cpf>/` | Remove cliente |

---

## Exemplo de Payload

**POST /api/clientes/**
```json
{
  "cpf": "12345678901",
  "nome": "Ana Silva",
  "email": "ana@email.com",
  "telefone": "98999990000"
}
```

**Resposta 201:**
```json
{
  "id": 1,
  "cpf": "12345678901",
  "nome": "Ana Silva",
  "email": "ana@email.com",
  "telefone": "98999990000",
  "criado_em": "2026-05-25T10:00:00Z"
}
```

**GET /api/clientes/12345678901/ (cache hit)**
```json
{
  "source": "cache",
  "data": { "id": 1, "cpf": "12345678901", "nome": "Ana Silva" }
}
```

---

## Análise de Complexidade

| Operação | Antes | Depois |
|---|---|---|
| Busca por CPF | O(n) | O(1) cache / O(log n) banco |
| Inserção | O(1) | O(1) |
| Remoção | O(n) | O(1) |
