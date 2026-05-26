# Módulo 04 — Histórico de Ações (Desfazer)

## Visão Geral

| Item | Detalhe |
|---|---|
| App Django | `historico` |
| Estrutura de dados | Pilha — Stack LIFO (list Python) |
| Complexidade antes | Incorreto — ordem de reversão não respeitada |
| Complexidade depois | O(1) para push e pop, sempre correto |

---

## Model

**Arquivo:** `historico/models.py`

```python
from django.db import models

class Acao(models.Model):
    TIPO_CHOICES = [
        ("criar_pedido", "Criar Pedido"),
        ("cancelar_pedido", "Cancelar Pedido"),
        ("ajustar_estoque", "Ajustar Estoque"),
        ("atualizar_cliente", "Atualizar Cliente"),
        ("cadastrar_produto", "Cadastrar Produto"),
    ]

    usuario    = models.CharField(max_length=100)
    tipo       = models.CharField(max_length=50, choices=TIPO_CHOICES)
    descricao  = models.TextField()
    dados_antes = models.JSONField(null=True, blank=True,
                                   help_text="Estado anterior ao da ação")
    dados_apos  = models.JSONField(null=True, blank=True,
                                   help_text="Estado após a ação")
    revertida  = models.BooleanField(default=False)
    criado_em  = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=["usuario", "revertida"]),
            models.Index(fields=["criado_em"]),
        ]
        ordering = ["-criado_em"]  # mais recente primeiro

    def __str__(self):
        return f"[{self.tipo}] {self.usuario} — {self.criado_em}"
```

---

## Serializer

**Arquivo:** `historico/serializers.py`

```python
from rest_framework import serializers
from .models import Acao

class AcaoSerializer(serializers.ModelSerializer):
    class Meta:
        model  = Acao
        fields = ["id", "usuario", "tipo", "descricao",
                  "dados_antes", "dados_apos", "revertida", "criado_em"]
        read_only_fields = ["revertida", "criado_em"]
```

---

## Pilha LIFO

**Arquivo:** `historico/stack.py`

```python
# Pilha LIFO por usuário — {usuario: [acao1, acao2, ...]}
_pilhas = {}

def empilhar(usuario: str, acao: dict):
    """
    Registra ação na pilha do usuário.
    Complexidade: O(1)
    """
    if usuario not in _pilhas:
        _pilhas[usuario] = []
    _pilhas[usuario].append(acao)  # push — O(1)

def desempilhar(usuario: str):
    """
    Retorna e remove a última ação do usuário (LIFO).
    Complexidade: O(1)
    """
    pilha = _pilhas.get(usuario, [])
    if not pilha:
        return None
    return pilha.pop()  # pop — O(1), sempre a última ação

def consultar_topo(usuario: str):
    """
    Consulta a última ação sem remover.
    Complexidade: O(1)
    """
    pilha = _pilhas.get(usuario, [])
    return pilha[-1] if pilha else None

def tamanho_pilha(usuario: str):
    return len(_pilhas.get(usuario, []))

def limpar_pilha(usuario: str):
    """Limpa o histórico do usuário."""
    _pilhas.pop(usuario, None)
```

---

## Views

**Arquivo:** `historico/views.py`

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import Acao
from .serializers import AcaoSerializer
from .stack import empilhar, desempilhar, consultar_topo, tamanho_pilha

class AcaoListView(APIView):
    """Lista histórico de ações e registra nova ação."""

    def get(self, request):
        usuario = request.query_params.get("usuario", "")
        qs = Acao.objects.filter(usuario=usuario, revertida=False)
        return Response(AcaoSerializer(qs, many=True).data)

    def post(self, request):
        serializer = AcaoSerializer(data=request.data)
        if serializer.is_valid():
            acao = serializer.save()
            # Empilha na pilha LIFO — O(1)
            empilhar(acao.usuario, AcaoSerializer(acao).data)
            return Response(AcaoSerializer(acao).data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class DesfazerAcaoView(APIView):
    """Desfaz a última ação do usuário — LIFO garantido."""

    def get(self, request, usuario):
        """Consulta o que seria desfeito sem desfazer."""
        topo = consultar_topo(usuario)
        if not topo:
            return Response({"mensagem": "Nenhuma ação para desfazer."})
        return Response({
            "proxima_a_desfazer": topo,
            "acoes_na_pilha": tamanho_pilha(usuario)
        })

    def post(self, request, usuario):
        """Desfaz a última ação — O(1)."""
        acao_dados = desempilhar(usuario)  # pop — O(1)
        if not acao_dados:
            return Response(
                {"mensagem": "Nenhuma ação para desfazer."},
                status=status.HTTP_204_NO_CONTENT
            )

        # Marca como revertida no banco
        Acao.objects.filter(id=acao_dados["id"]).update(revertida=True)

        return Response({
            "desfeita": acao_dados,
            "acoes_restantes": tamanho_pilha(usuario)
        })
```

---

## URLs

**Arquivo:** `historico/urls.py`

```python
from django.urls import path
from .views import AcaoListView, DesfazerAcaoView

urlpatterns = [
    path("", AcaoListView.as_view()),                          # GET, POST
    path("desfazer/<str:usuario>/", DesfazerAcaoView.as_view()), # GET (consulta), POST (desfaz)
]
```

**Registro em** `hublog/urls.py`:

```python
path("api/historico/", include("historico.urls")),
```

---

## Endpoints

| Método | URL | Descrição |
|---|---|---|
| GET | `/api/historico/?usuario=joao` | Lista ações do usuário |
| POST | `/api/historico/` | Registra nova ação na pilha O(1) |
| GET | `/api/historico/desfazer/<usuario>/` | Consulta o que será desfeito O(1) |
| POST | `/api/historico/desfazer/<usuario>/` | Desfaz última ação LIFO O(1) |

---

## Exemplo de Payload

**POST /api/historico/ — registrar ação:**
```json
{
  "usuario": "joao",
  "tipo": "ajustar_estoque",
  "descricao": "Saída de 10 unidades de FRU001",
  "dados_antes": { "sku": "FRU001", "estoque": 100 },
  "dados_apos":  { "sku": "FRU001", "estoque": 90 }
}
```

**POST /api/historico/desfazer/joao/ — desfaz:**
```json
{
  "desfeita": {
    "id": 7,
    "tipo": "ajustar_estoque",
    "descricao": "Saída de 10 unidades de FRU001",
    "dados_antes": { "sku": "FRU001", "estoque": 100 }
  },
  "acoes_restantes": 2
}
```

---

## Análise de Complexidade

| Operação | Antes (estrutura errada) | Depois (Pilha LIFO) |
|---|---|---|
| Registrar ação | O(1) | O(1) |
| Desfazer última ação | Incorreto | O(1) — sempre correto |
| Consultar próxima | Incorreto | O(1) |
| Ordem de reversão | Não garantida | Garantida pela estrutura |
