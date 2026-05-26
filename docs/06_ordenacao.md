# Módulo 06 — Ordenação de Pedidos por Prioridade

## Visão Geral

| Item | Detalhe |
|---|---|
| App Django | `ordenacao` |
| Estrutura de dados | Tim Sort — sorted() nativo do Python |
| Complexidade antes | O(n²) — Bubble Sort recalculado em tempo real |
| Complexidade depois | O(n log n) garantido / O(n) quando parcialmente ordenado |

---

## Model

Este módulo não possui model próprio — opera sobre os dados do model `Pedido` do módulo 03. O foco está na lógica de ordenação e nos endpoints de consulta do painel.

---

## Serializer

**Arquivo:** `ordenacao/serializers.py`

```python
from rest_framework import serializers

class FiltroOrdenacaoSerializer(serializers.Serializer):
    CRITERIO_CHOICES = [
        ("prioridade", "Prioridade"),
        ("criado_em", "Data de criação"),
        ("status", "Status"),
    ]
    ORDEM_CHOICES = [
        ("asc", "Crescente"),
        ("desc", "Decrescente"),
    ]

    criterio = serializers.ChoiceField(
        choices=CRITERIO_CHOICES,
        default="prioridade"
    )
    ordem = serializers.ChoiceField(
        choices=ORDEM_CHOICES,
        default="asc"
    )
    limite = serializers.IntegerField(
        min_value=1,
        max_value=500,
        default=50
    )
```

---

## Lógica de Ordenação

**Arquivo:** `ordenacao/sorter.py`

```python
import time

def ordenar_pedidos(pedidos: list, criterio: str = "prioridade", ordem: str = "asc") -> dict:
    """
    Ordena pedidos usando Tim Sort — O(n log n) garantido.
    Estável: pedidos com mesma prioridade mantêm ordem de chegada (FIFO).

    Parâmetros:
        pedidos:  lista de dicts com dados dos pedidos
        criterio: campo para ordenação
        ordem:    "asc" ou "desc"

    Retorna dict com pedidos ordenados e métricas de performance.
    """
    reverso = (ordem == "desc")
    n = len(pedidos)

    inicio = time.perf_counter()

    # Tim Sort nativo — O(n log n) pior caso, O(n) se já ordenado
    pedidos_ordenados = sorted(
        pedidos,
        key=lambda p: (p[criterio], p.get("criado_em", "")),
        reverse=reverso
    )

    fim = time.perf_counter()
    tempo_ms = round((fim - inicio) * 1000, 4)

    # Estimativa de operações para fins didáticos
    import math
    ops_bubble   = n * n          # O(n²) — pior caso Bubble Sort
    ops_timsort  = int(n * math.log2(n)) if n > 1 else 1  # O(n log n)
    ganho = round(ops_bubble / ops_timsort, 1) if ops_timsort > 0 else 1

    return {
        "pedidos":         pedidos_ordenados,
        "total":           n,
        "criterio":        criterio,
        "ordem":           ordem,
        "tempo_ms":        tempo_ms,
        "algoritmo":       "Tim Sort",
        "complexidade":    "O(n log n)",
        "metricas": {
            "ops_bubble_sort": ops_bubble,
            "ops_tim_sort":    ops_timsort,
            "ganho_estimado":  f"{ganho}x"
        }
    }


def ordenacao_multipla(pedidos: list, criterios: list) -> list:
    """
    Ordenação por múltiplos critérios — ex: prioridade, depois tempo.
    Complexidade: O(n log n)
    """
    return sorted(
        pedidos,
        key=lambda p: tuple(p.get(c, "") for c in criterios)
    )


def inserir_ordenado(pedidos: list, novo_pedido: dict, criterio: str = "prioridade") -> list:
    """
    Insere novo pedido na posição correta sem reordenar tudo.
    Usa bisect para encontrar posição — O(log n) busca + O(n) inserção.
    Mais eficiente que reordenar tudo quando a lista já está ordenada.
    """
    import bisect
    chaves = [p[criterio] for p in pedidos]
    pos = bisect.bisect_left(chaves, novo_pedido[criterio])
    pedidos.insert(pos, novo_pedido)
    return pedidos
```

---

## Views

**Arquivo:** `ordenacao/views.py`

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from pedidos.models import Pedido
from pedidos.serializers import PedidoSerializer
from .serializers import FiltroOrdenacaoSerializer
from .sorter import ordenar_pedidos, ordenacao_multipla

class PainelEntregadorView(APIView):
    """
    Painel principal do entregador com pedidos ordenados por prioridade.
    Substitui o Bubble Sort por Tim Sort.
    """

    def get(self, request):
        serializer = FiltroOrdenacaoSerializer(data=request.query_params)
        if not serializer.is_valid():
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        criterio = serializer.validated_data["criterio"]
        ordem    = serializer.validated_data["ordem"]
        limite   = serializer.validated_data["limite"]

        # Busca do banco já com índice
        pedidos_qs = Pedido.objects.filter(
            status="aguardando"
        ).values("id", "prioridade", "status", "descricao", "criado_em")[:limite * 2]

        pedidos_lista = list(pedidos_qs)

        # Tim Sort — O(n log n)
        resultado = ordenar_pedidos(pedidos_lista, criterio, ordem)
        resultado["pedidos"] = resultado["pedidos"][:limite]

        return Response(resultado)


class ComparativoAlgoritmosView(APIView):
    """
    Endpoint didático — compara performance dos algoritmos.
    Demonstra o ganho do Tim Sort sobre o Bubble Sort.
    """

    def get(self, request):
        import math

        casos = [100, 500, 1000, 3200, 10000]
        comparativo = []

        for n in casos:
            ops_bubble  = n * n
            ops_merge   = int(n * math.log2(n)) if n > 1 else 1
            ops_timsort = int(n * math.log2(n)) if n > 1 else 1

            comparativo.append({
                "n_pedidos":    n,
                "bubble_sort":  ops_bubble,
                "merge_sort":   ops_merge,
                "tim_sort":     ops_timsort,
                "ganho_vs_bubble": f"{round(ops_bubble / ops_timsort, 0):.0f}x"
            })

        return Response({
            "tabela": comparativo,
            "recomendado": "Tim Sort",
            "motivo": (
                "O(n log n) garantido no pior caso, estável, "
                "nativo do Python via sorted(), otimizado para "
                "dados parcialmente ordenados."
            )
        })
```

---

## URLs

**Arquivo:** `ordenacao/urls.py`

```python
from django.urls import path
from .views import PainelEntregadorView, ComparativoAlgoritmosView

urlpatterns = [
    path("painel/", PainelEntregadorView.as_view()),          # GET
    path("comparativo/", ComparativoAlgoritmosView.as_view()), # GET
]
```

**Registro em** `hublog/urls.py`:

```python
path("api/ordenacao/", include("ordenacao.urls")),
```

---

## Endpoints

| Método | URL | Descrição |
|---|---|---|
| GET | `/api/ordenacao/painel/` | Painel ordenado por prioridade — Tim Sort |
| GET | `/api/ordenacao/painel/?criterio=criado_em&ordem=asc` | Ordena por data |
| GET | `/api/ordenacao/painel/?limite=100` | Limita resultados |
| GET | `/api/ordenacao/comparativo/` | Comparativo Bubble vs Tim Sort |

---

## Exemplo de Resposta

**GET /api/ordenacao/painel/**
```json
{
  "pedidos": [
    { "id": 42, "prioridade": 1, "descricao": "VIP urgente" },
    { "id": 17, "prioridade": 2, "descricao": "Cliente premium" },
    { "id": 91, "prioridade": 3, "descricao": "Entrega padrão" }
  ],
  "total": 3200,
  "criterio": "prioridade",
  "algoritmo": "Tim Sort",
  "complexidade": "O(n log n)",
  "tempo_ms": 1.24,
  "metricas": {
    "ops_bubble_sort": 10240000,
    "ops_tim_sort": 35200,
    "ganho_estimado": "290.9x"
  }
}
```

**GET /api/ordenacao/comparativo/**
```json
{
  "tabela": [
    { "n_pedidos": 100,   "bubble_sort": 10000,     "tim_sort": 664,   "ganho_vs_bubble": "15x" },
    { "n_pedidos": 500,   "bubble_sort": 250000,    "tim_sort": 4482,  "ganho_vs_bubble": "56x" },
    { "n_pedidos": 3200,  "bubble_sort": 10240000,  "tim_sort": 35200, "ganho_vs_bubble": "291x" },
    { "n_pedidos": 10000, "bubble_sort": 100000000, "tim_sort": 132877,"ganho_vs_bubble": "753x" }
  ],
  "recomendado": "Tim Sort"
}
```

---

## Análise de Complexidade

| Algoritmo | Melhor | Médio | Pior | Estável | Recomendado |
|---|---|---|---|---|---|
| Bubble Sort (atual) | O(n) | O(n²) | O(n²) | Sim | Não |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | Sim | Bom |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | Não | Arriscado |
| **Tim Sort** | **O(n)** | **O(n log n)** | **O(n log n)** | **Sim** | **Recomendado** |

**Para 3.200 pedidos:**
- Bubble Sort: ~10.240.000 operações
- Tim Sort: ~35.200 operações
- Ganho: **~290x**
