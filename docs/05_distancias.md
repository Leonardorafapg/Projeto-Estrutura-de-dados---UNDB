# Módulo 05 — Matriz de Distâncias

## Visão Geral

| Item | Detalhe |
|---|---|
| App Django | `distancias` |
| Estrutura de dados | Dicionário bidimensional — HashMap aninhado |
| Complexidade antes | O(n) — busca sequencial pelo par de pontos |
| Complexidade depois | O(1) — acesso direto por chave origem/destino |

---

## Model

**Arquivo:** `distancias/models.py`

```python
from django.db import models

class PontoEntrega(models.Model):
    codigo    = models.CharField(max_length=50, unique=True, db_index=True)
    nome      = models.CharField(max_length=200)
    latitude  = models.DecimalField(max_digits=9, decimal_places=6)
    longitude = models.DecimalField(max_digits=9, decimal_places=6)
    ativo     = models.BooleanField(default=True)

    def __str__(self):
        return f"{self.codigo} — {self.nome}"


class Distancia(models.Model):
    origem  = models.ForeignKey(
        PontoEntrega, on_delete=models.CASCADE, related_name="saidas"
    )
    destino = models.ForeignKey(
        PontoEntrega, on_delete=models.CASCADE, related_name="chegadas"
    )
    km      = models.DecimalField(max_digits=8, decimal_places=2)
    minutos = models.PositiveIntegerField(help_text="Tempo estimado em minutos")

    class Meta:
        # Garante que cada par origem/destino é único
        constraints = [
            models.UniqueConstraint(
                fields=["origem", "destino"],
                name="par_unico_origem_destino"
            )
        ]
        indexes = [
            models.Index(fields=["origem", "destino"]),
        ]

    def __str__(self):
        return f"{self.origem.codigo} → {self.destino.codigo}: {self.km}km"
```

---

## Serializer

**Arquivo:** `distancias/serializers.py`

```python
from rest_framework import serializers
from .models import PontoEntrega, Distancia

class PontoEntregaSerializer(serializers.ModelSerializer):
    class Meta:
        model  = PontoEntrega
        fields = ["id", "codigo", "nome", "latitude", "longitude", "ativo"]


class DistanciaSerializer(serializers.ModelSerializer):
    origem_codigo  = serializers.CharField(source="origem.codigo", read_only=True)
    destino_codigo = serializers.CharField(source="destino.codigo", read_only=True)

    class Meta:
        model  = Distancia
        fields = ["id", "origem", "origem_codigo",
                  "destino", "destino_codigo", "km", "minutos"]


class ConsultaDistanciaSerializer(serializers.Serializer):
    origem  = serializers.CharField()
    destino = serializers.CharField()
```

---

## HashMap de Distâncias

**Arquivo:** `distancias/matriz.py`

```python
# HashMap bidimensional — {origem: {destino: {km, minutos}}}
_matriz = {}

def registrar(origem: str, destino: str, km: float, minutos: int):
    """
    Registra distância entre dois pontos nos dois sentidos.
    Complexidade: O(1)
    """
    _matriz.setdefault(origem, {})[destino] = {"km": km, "minutos": minutos}
    _matriz.setdefault(destino, {})[origem] = {"km": km, "minutos": minutos}

def consultar(origem: str, destino: str):
    """
    Retorna distância diretamente pelo par de chaves.
    Complexidade: O(1)
    """
    return _matriz.get(origem, {}).get(destino)

def carregar_do_banco(distancias: list):
    """
    Constrói o HashMap a partir dos registros do banco.
    Chamado na inicialização do servidor.
    Complexidade: O(n)
    """
    for d in distancias:
        registrar(
            origem=d.origem.codigo,
            destino=d.destino.codigo,
            km=float(d.km),
            minutos=d.minutos
        )

def destinos_de(origem: str):
    """Retorna todos os destinos acessíveis a partir de uma origem."""
    return _matriz.get(origem, {})
```

---

## Views

**Arquivo:** `distancias/views.py`

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import PontoEntrega, Distancia
from .serializers import (
    PontoEntregaSerializer, DistanciaSerializer, ConsultaDistanciaSerializer
)
from .matriz import registrar, consultar, destinos_de

class PontoListView(APIView):
    """Lista e cadastra pontos de entrega."""

    def get(self, request):
        pontos = PontoEntrega.objects.filter(ativo=True)
        return Response(PontoEntregaSerializer(pontos, many=True).data)

    def post(self, request):
        serializer = PontoEntregaSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class DistanciaView(APIView):
    """Registra e consulta distâncias entre pontos."""

    def post(self, request):
        """Registra nova distância entre dois pontos."""
        serializer = DistanciaSerializer(data=request.data)
        if serializer.is_valid():
            dist = serializer.save()
            # Popula o HashMap — O(1)
            registrar(
                origem=dist.origem.codigo,
                destino=dist.destino.codigo,
                km=float(dist.km),
                minutos=dist.minutos
            )
            return Response(DistanciaSerializer(dist).data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class ConsultaDistanciaView(APIView):
    """Consulta distância entre dois pontos — O(1)."""

    def get(self, request):
        serializer = ConsultaDistanciaSerializer(data=request.query_params)
        if not serializer.is_valid():
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        origem  = serializer.validated_data["origem"]
        destino = serializer.validated_data["destino"]

        # Acesso direto ao HashMap — O(1)
        resultado = consultar(origem, destino)

        if not resultado:
            return Response(
                {"erro": f"Distância entre {origem} e {destino} não cadastrada."},
                status=status.HTTP_404_NOT_FOUND
            )

        return Response({
            "origem": origem,
            "destino": destino,
            "km": resultado["km"],
            "minutos": resultado["minutos"]
        })


class DestinosView(APIView):
    """Lista todos os destinos acessíveis a partir de uma origem."""

    def get(self, request, origem):
        destinos = destinos_de(origem)
        if not destinos:
            return Response({"erro": "Origem não encontrada."}, status=status.HTTP_404_NOT_FOUND)
        return Response({"origem": origem, "destinos": destinos})
```

---

## URLs

**Arquivo:** `distancias/urls.py`

```python
from django.urls import path
from .views import PontoListView, DistanciaView, ConsultaDistanciaView, DestinosView

urlpatterns = [
    path("pontos/", PontoListView.as_view()),               # GET, POST
    path("", DistanciaView.as_view()),                       # POST
    path("consulta/", ConsultaDistanciaView.as_view()),      # GET ?origem=A&destino=B
    path("destinos/<str:origem>/", DestinosView.as_view()),  # GET
]
```

**Registro em** `hublog/urls.py`:

```python
path("api/distancias/", include("distancias.urls")),
```

---

## Endpoints

| Método | URL | Descrição |
|---|---|---|
| GET | `/api/distancias/pontos/` | Lista pontos de entrega |
| POST | `/api/distancias/pontos/` | Cadastra ponto de entrega |
| POST | `/api/distancias/` | Registra distância entre dois pontos |
| GET | `/api/distancias/consulta/?origem=A&destino=B` | Consulta distância O(1) |
| GET | `/api/distancias/destinos/<origem>/` | Lista destinos de uma origem |

---

## Exemplo de Payload

**POST /api/distancias/ — registrar:**
```json
{
  "origem": 1,
  "destino": 2,
  "km": 5.2,
  "minutos": 15
}
```

**GET /api/distancias/consulta/?origem=CENTRO&destino=COHAB:**
```json
{
  "origem": "CENTRO",
  "destino": "COHAB",
  "km": 5.2,
  "minutos": 15
}
```

---

## Análise de Complexidade

| Operação | Antes | Depois |
|---|---|---|
| Consultar distância A→B | O(n) | O(1) |
| Consultar distância B→A | O(n) | O(1) bidirecional |
| Registrar distância | O(1) | O(1) |
| Carregar do banco | O(n) | O(n) — apenas na inicialização |
