# Módulo 03 — Fila de Pedidos dos Entregadores

## Visão Geral

| Item | Detalhe |
|---|---|
| App Django | `pedidos` |
| Estrutura de dados | Min-Heap (heapq nativo do Python) |
| Complexidade antes | O(n) — varredura completa por priorização |
| Complexidade depois | O(log n) inserção/extração, O(1) consulta do topo |

---

## Model

**Arquivo:** `pedidos/models.py`

```python
from django.db import models

class Pedido(models.Model):
    STATUS_CHOICES = [
        ("aguardando", "Aguardando"),
        ("em_rota", "Em Rota"),
        ("entregue", "Entregue"),
        ("cancelado", "Cancelado"),
    ]

    cliente    = models.ForeignKey("clientes.Cliente", on_delete=models.PROTECT)
    prioridade = models.PositiveSmallIntegerField(
        default=5,
        help_text="1 = urgentíssimo, 10 = normal"
    )
    status     = models.CharField(max_length=20, choices=STATUS_CHOICES, default="aguardando")
    descricao  = models.TextField(blank=True)
    criado_em  = models.DateTimeField(auto_now_add=True)
    atualizado = models.DateTimeField(auto_now=True)

    class Meta:
        indexes = [
            models.Index(fields=["prioridade", "status"]),
            models.Index(fields=["status"]),
        ]

    def __str__(self):
        return f"Pedido #{self.id} — Prioridade {self.prioridade}"
```

---

## Serializer

**Arquivo:** `pedidos/serializers.py`

```python
from rest_framework import serializers
from .models import Pedido

class PedidoSerializer(serializers.ModelSerializer):
    class Meta:
        model  = Pedido
        fields = ["id", "cliente", "prioridade", "status",
                  "descricao", "criado_em"]

    def validate_prioridade(self, value):
        if not 1 <= value <= 10:
            raise serializers.ValidationError("Prioridade deve ser entre 1 e 10.")
        return value
```

---

## Fila de Prioridade (Heap)

**Arquivo:** `pedidos/priority_queue.py`

```python
import heapq
import time

# Min-Heap global — menor prioridade = mais urgente
_fila = []

def adicionar_pedido(prioridade: int, pedido_id: int, dados: dict):
    """
    Insere pedido no heap.
    Complexidade: O(log n)
    Desempate por timestamp — garante FIFO dentro da mesma prioridade.
    """
    timestamp = time.time()
    heapq.heappush(_fila, (prioridade, timestamp, pedido_id, dados))

def proximo_pedido():
    """
    Retorna e remove o pedido mais urgente.
    Complexidade: O(log n)
    """
    if _fila:
        return heapq.heappop(_fila)
    return None

def consultar_topo():
    """
    Consulta o pedido mais urgente sem remover.
    Complexidade: O(1)
    """
    if _fila:
        return _fila[0]
    return None

def tamanho_fila():
    return len(_fila)

def reconstruir_fila(pedidos: list):
    """
    Reconstrói o heap a partir de uma lista.
    Usado na inicialização do servidor.
    Complexidade: O(n)
    """
    global _fila
    _fila = [
        (p["prioridade"], p["criado_em"], p["id"], p)
        for p in pedidos
    ]
    heapq.heapify(_fila)  # O(n) — mais eficiente que n inserções O(log n)
```

---

## Views

**Arquivo:** `pedidos/views.py`

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import Pedido
from .serializers import PedidoSerializer
from .priority_queue import (
    adicionar_pedido, proximo_pedido,
    consultar_topo, tamanho_fila
)

class PedidoListView(APIView):
    """Lista pedidos aguardando e cria novo pedido."""

    def get(self, request):
        pedidos = Pedido.objects.filter(status="aguardando").order_by("prioridade", "criado_em")
        return Response(PedidoSerializer(pedidos, many=True).data)

    def post(self, request):
        serializer = PedidoSerializer(data=request.data)
        if serializer.is_valid():
            pedido = serializer.save()
            # Insere no heap — O(log n)
            adicionar_pedido(
                prioridade=pedido.prioridade,
                pedido_id=pedido.id,
                dados=PedidoSerializer(pedido).data
            )
            return Response(PedidoSerializer(pedido).data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class ProximoPedidoView(APIView):
    """Retorna e processa o pedido mais urgente da fila."""

    def get(self, request):
        """Consulta sem remover — O(1)."""
        topo = consultar_topo()
        if not topo:
            return Response({"mensagem": "Fila vazia."}, status=status.HTTP_204_NO_CONTENT)
        prioridade, _, pedido_id, dados = topo
        return Response({"prioridade": prioridade, "pedido": dados})

    def post(self, request):
        """Processa (remove) o pedido mais urgente — O(log n)."""
        item = proximo_pedido()
        if not item:
            return Response({"mensagem": "Fila vazia."}, status=status.HTTP_204_NO_CONTENT)

        prioridade, _, pedido_id, dados = item
        Pedido.objects.filter(id=pedido_id).update(status="em_rota")
        return Response({
            "processado": pedido_id,
            "prioridade": prioridade,
            "fila_restante": tamanho_fila()
        })


class StatusFilaView(APIView):
    """Retorna o tamanho atual da fila."""

    def get(self, request):
        return Response({"pedidos_na_fila": tamanho_fila()})
```

---

## URLs

**Arquivo:** `pedidos/urls.py`

```python
from django.urls import path
from .views import PedidoListView, ProximoPedidoView, StatusFilaView

urlpatterns = [
    path("", PedidoListView.as_view()),            # GET, POST
    path("proximo/", ProximoPedidoView.as_view()),  # GET (consulta), POST (processa)
    path("fila/status/", StatusFilaView.as_view()), # GET
]
```

**Registro em** `hublog/urls.py`:

```python
path("api/pedidos/", include("pedidos.urls")),
```

---

## Endpoints

| Método | URL | Descrição |
|---|---|---|
| GET | `/api/pedidos/` | Lista pedidos aguardando |
| POST | `/api/pedidos/` | Cria pedido e insere no heap O(log n) |
| GET | `/api/pedidos/proximo/` | Consulta mais urgente O(1) |
| POST | `/api/pedidos/proximo/` | Processa mais urgente O(log n) |
| GET | `/api/pedidos/fila/status/` | Tamanho atual da fila |

---

## Exemplo de Payload

**POST /api/pedidos/**
```json
{
  "cliente": 1,
  "prioridade": 1,
  "descricao": "Entrega urgente — cliente VIP"
}
```

**GET /api/pedidos/proximo/ — consulta o topo:**
```json
{
  "prioridade": 1,
  "pedido": {
    "id": 42,
    "cliente": 1,
    "prioridade": 1,
    "status": "aguardando"
  }
}
```

**POST /api/pedidos/proximo/ — processa:**
```json
{
  "processado": 42,
  "prioridade": 1,
  "fila_restante": 3199
}
```

---

## Análise de Complexidade

| Operação | Antes | Depois |
|---|---|---|
| Encontrar mais urgente | O(n) | O(1) |
| Extrair mais urgente | O(n) | O(log n) |
| Inserir novo pedido | O(1) | O(log n) |
| Reconstruir fila | O(n²) | O(n) via heapify |

**Para 3.200 pedidos:**
- Antes: 3.200 operações por extração
- Depois: ~12 operações por extração
- Ganho: **~275x**
