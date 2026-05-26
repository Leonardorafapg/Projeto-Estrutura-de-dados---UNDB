# HubLog — Documentação Geral do Projeto

## Contexto

O HubLog é o sistema interno de gestão de pedidos e rotas da **RotaVerde**, startup maranhense de logística do último quilômetro especializada na entrega de produtos orgânicos em São Luís. Com mais de 8.000 clientes ativos, 200 entregadores cadastrados e um catálogo de aproximadamente 1.500 itens, o sistema entrou em colapso durante o Festival Orgânico da Primavera ao receber 3.200 pedidos em menos de 40 minutos — gerando um prejuízo estimado de R$ 180.000.

A investigação revelou que todos os problemas tinham raiz comum: nenhuma decisão sobre como os dados seriam armazenados, organizados e acessados foi tomada com critério técnico. Este projeto documenta a reestruturação completa do núcleo de dados do HubLog.

---

## Objetivo

Substituir as decisões inadequadas de organização de dados por estruturas tecnicamente corretas para cada contexto operacional, eliminando os gargalos de performance e garantindo que a plataforma suporte o crescimento da RotaVerde com eficiência, consistência e escalabilidade.

---

## Stack Tecnológica

| Camada | Tecnologia | Função |
|---|---|---|
| Backend | Django 4.x + Django REST Framework | API REST, models, regras de negócio |
| Frontend | Next.js 14 | Interface do operador e entregador |
| Banco de dados | SQLite (desenvolvimento) | Persistência dos dados |
| Linguagem | Python 3.11+ | Implementação das estruturas de dados |

---

## Arquitetura Geral

O projeto é dividido em dois serviços independentes que se comunicam via API REST:

```
┌─────────────────────────────────────────────┐
│                  FRONTEND                   │
│              Next.js 14                     │
│   /clientes  /catalogo  /pedidos            │
│   /historico /distancias /painel            │
└──────────────────┬──────────────────────────┘
                   │ HTTP / JSON
┌──────────────────▼──────────────────────────┐
│                  BACKEND                    │
│         Django + Django REST Framework      │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │clientes  │  │catalogo  │  │ pedidos  │  │
│  │ HashMap  │  │HashSet   │  │ Min-Heap │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │historico │  │distancias│  │ordenacao │  │
│  │  Stack   │  │HashMap2D │  │ TimSort  │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│              SQLite / ORM Django            │
└─────────────────────────────────────────────┘
```

---

## Estrutura de Pastas

```
hublog/
├── backend/
│   ├── manage.py
│   ├── hublog/
│   │   ├── settings.py
│   │   └── urls.py
│   ├── clientes/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── catalogo/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── urls.py
│   ├── pedidos/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── priority_queue.py
│   ├── historico/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── stack.py
│   ├── distancias/
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── matriz.py
│   └── ordenacao/
│       ├── serializers.py
│       ├── views.py
│       ├── urls.py
│       └── sorter.py
│
└── frontend/
    ├── app/
    │   ├── page.tsx
    │   ├── clientes/
    │   ├── catalogo/
    │   ├── pedidos/
    │   ├── historico/
    │   ├── distancias/
    │   └── painel/
    └── components/
```

---

## Módulos do Sistema

| # | Módulo | App Django | Estrutura Adotada | Antes | Depois |
|---|---|---|---|---|---|
| 01 | [Cadastro de Clientes](./01_clientes.md) | `clientes` | HashMap | O(n) | O(1) |
| 02 | [Catálogo de Produtos](./02_catalogo.md) | `catalogo` | HashSet + HashMap | O(n) | O(1) |
| 03 | [Fila de Pedidos](./03_pedidos.md) | `pedidos` | Min-Heap (heapq) | O(n) | O(log n) |
| 04 | [Histórico / Desfazer](./04_historico.md) | `historico` | Pilha LIFO | Incorreto | O(1) |
| 05 | [Matriz de Distâncias](./05_distancias.md) | `distancias` | HashMap 2D | O(n) | O(1) |
| 06 | [Ordenação de Pedidos](./06_ordenacao.md) | `ordenacao` | Tim Sort | O(n²) | O(n log n) |

---

## Mapa de Endpoints

| Módulo | Método | Endpoint | Descrição |
|---|---|---|---|
| Clientes | GET | `/api/clientes/` | Lista clientes |
| Clientes | POST | `/api/clientes/` | Cria cliente |
| Clientes | GET | `/api/clientes/<cpf>/` | Busca por CPF — O(1) |
| Clientes | PUT | `/api/clientes/<cpf>/` | Atualiza cliente |
| Clientes | DELETE | `/api/clientes/<cpf>/` | Remove cliente |
| Catálogo | GET | `/api/produtos/` | Lista produtos |
| Catálogo | POST | `/api/produtos/` | Cria produto — dup. O(1) |
| Catálogo | GET | `/api/produtos/<sku>/` | Busca por SKU |
| Catálogo | PUT | `/api/produtos/<sku>/` | Atualiza produto |
| Catálogo | DELETE | `/api/produtos/<sku>/` | Desativa produto |
| Catálogo | PATCH | `/api/produtos/<sku>/estoque/` | Ajusta estoque |
| Pedidos | GET | `/api/pedidos/` | Lista pedidos aguardando |
| Pedidos | POST | `/api/pedidos/` | Cria pedido — insere no heap |
| Pedidos | GET | `/api/pedidos/proximo/` | Consulta mais urgente — O(1) |
| Pedidos | POST | `/api/pedidos/proximo/` | Processa mais urgente — O(log n) |
| Pedidos | GET | `/api/pedidos/fila/status/` | Tamanho da fila |
| Histórico | GET | `/api/historico/` | Lista ações do usuário |
| Histórico | POST | `/api/historico/` | Registra ação na pilha |
| Histórico | GET | `/api/historico/desfazer/<usuario>/` | Consulta próxima a desfazer |
| Histórico | POST | `/api/historico/desfazer/<usuario>/` | Desfaz última ação — LIFO |
| Distâncias | GET | `/api/distancias/pontos/` | Lista pontos de entrega |
| Distâncias | POST | `/api/distancias/pontos/` | Cadastra ponto |
| Distâncias | POST | `/api/distancias/` | Registra distância entre pontos |
| Distâncias | GET | `/api/distancias/consulta/` | Consulta distância — O(1) |
| Distâncias | GET | `/api/distancias/destinos/<origem>/` | Destinos de uma origem |
| Ordenação | GET | `/api/ordenacao/painel/` | Painel ordenado — Tim Sort |
| Ordenação | GET | `/api/ordenacao/comparativo/` | Comparativo de algoritmos |

---

## Resumo do Ganho de Performance

| Módulo | Problema Original | Ganho Obtido |
|---|---|---|
| Clientes | Busca sequencial O(n) em 8.000 registros | Acesso direto O(1) via HashMap |
| Catálogo | Duplicatas sem controle — inconsistência de estoque | Unicidade automática O(1) via HashSet |
| Pedidos | Varredura O(n) a cada priorização | Extração O(log n) — ganho de ~275x para 3.200 pedidos |
| Histórico | Reversão na ordem errada | O(1) com ordem LIFO garantida por estrutura |
| Distâncias | Acesso sem indexação — O(n) por par de pontos | Acesso direto O(1) via HashMap 2D |
| Ordenação | Bubble Sort O(n²) em tempo real | Tim Sort O(n log n) — ganho de ~290x para 3.200 pedidos |

---

## Decisões de Projeto

**Por que Django REST Framework?**
Permite construir APIs REST com autenticação, serialização e validação de dados com pouco código, o que reduz a curva de aprendizado para um time iniciante e acelera o desenvolvimento dentro do prazo de 2 a 3 semanas.

**Por que Next.js?**
Oferece roteamento automático por pastas, suporte nativo a TypeScript e consumo simplificado de APIs externas via `fetch`, tornando a integração com o backend Django direta e sem configuração complexa.

**Por que SQLite no desenvolvimento?**
Não requer instalação ou configuração de servidor de banco de dados. O ORM do Django abstrai as diferenças entre bancos, permitindo migrar para PostgreSQL em produção sem alterações no código.

**Por que as estruturas de dados são mantidas em memória?**
Para os módulos onde a latência é crítica — como a fila de pedidos e o histórico de desfazer — manter a estrutura em memória (HashMap, Heap, Stack) elimina o custo de I/O do banco em cada operação. O banco é usado apenas para persistência e recuperação na inicialização do servidor.

