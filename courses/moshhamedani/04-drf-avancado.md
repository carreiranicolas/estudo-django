# 4. Django REST Framework — Recursos Avançados

## 4.1 De função para classe: `APIView`

As views baseadas em função funcionam bem, mas tendem a acumular `if request.method == ...`
conforme o endpoint cresce. `APIView` resolve isso mapeando cada verbo HTTP para um método
próprio da classe:

```python
# catalogo/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from django.shortcuts import get_object_or_404
from .models import Livro
from .serializers import LivroSerializer


class ListaLivrosView(APIView):
    def get(self, request):
        livros = Livro.objects.select_related("autor").all()
        return Response(LivroSerializer(livros, many=True).data)

    def post(self, request):
        serializer = LivroSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data, status=status.HTTP_201_CREATED)


class DetalheLivroView(APIView):
    def get(self, request, pk):
        livro = get_object_or_404(Livro, pk=pk)
        return Response(LivroSerializer(livro).data)

    def put(self, request, pk):
        livro = get_object_or_404(Livro, pk=pk)
        serializer = LivroSerializer(livro, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)

    def delete(self, request, pk):
        livro = get_object_or_404(Livro, pk=pk)
        livro.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

```python
urlpatterns = [
    path("livros/", ListaLivrosView.as_view()),
    path("livros/<int:pk>/", DetalheLivroView.as_view()),
]
```

`as_view()` é o que transforma a classe de volta em uma função compatível com o roteador de
URLs do Django — por baixo dos panos, sempre existe uma função no final da cadeia.

## 4.2 Mixins: reaproveitando o padrão de CRUD

Se você reparar, o corpo de `get()` em `ListaLivrosView` (montar queryset → serializar →
responder) é **idêntico** para qualquer outro modelo — só muda o queryset e o serializer.
O DRF já empacota esse padrão em **mixins** (mixins são classes que empacotam um padrão):

| Mixin | Implementa |
|---|---|
| `ListModelMixin` | `.list()` — listar coleção |
| `CreateModelMixin` | `.create()` — criar |
| `RetrieveModelMixin` | `.retrieve()` — buscar um item |
| `UpdateModelMixin` | `.update()` — atualizar (PUT/PATCH) |
| `DestroyModelMixin` | `.destroy()` — apagar |

Combinados com `GenericAPIView` (que fornece `get_queryset()`, `get_serializer_class()`,
etc.), eles formam a base das **generic views**:

```python
from rest_framework import mixins, generics


class ListaLivrosView(mixins.ListModelMixin, mixins.CreateModelMixin, generics.GenericAPIView):
    queryset = Livro.objects.select_related("autor").all()
    serializer_class = LivroSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

## 4.3 Generic views prontas: o caminho mais curto

Na prática, raramente você monta a combinação de mixins na mão — o DRF já entrega as
combinações mais comuns prontas:

```python
from rest_framework import generics


class ListaLivrosView(generics.ListCreateAPIView):
    queryset = Livro.objects.select_related("autor").all()
    serializer_class = LivroSerializer


class DetalheLivroView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Livro.objects.select_related("autor").all()
    serializer_class = LivroSerializer
```

| Classe | Operações |
|---|---|
| `ListAPIView` | listar |
| `CreateAPIView` | criar |
| `RetrieveAPIView` | buscar um |
| `ListCreateAPIView` | listar + criar |
| `RetrieveUpdateAPIView` | buscar + atualizar |
| `RetrieveDestroyAPIView` | buscar + apagar |
| `RetrieveUpdateDestroyAPIView` | buscar + atualizar + apagar |

Sobrescrever `get_queryset()`/`get_serializer_class()` em vez de usar os atributos fixos
permite injetar lógica condicional (baseada no usuário logado, em parâmetros da URL, etc.):

```python
class DetalheLivroView(generics.RetrieveUpdateDestroyAPIView):
    serializer_class = LivroSerializer

    def get_queryset(self): # define o **QuerySet que a View irá utilizar para buscar seus objetos. Podemos sobrescrevê-lo para aplicar filtros, regras condicionais ou otimizações
        return Livro.objects.select_related("autor").prefetch_related("categorias")

    def destroy(self, request, *args, **kwargs): # Nas Generic Views, o método destroy() é responsável pelo comportamento do DELETE. Podemos sobrescrevê-lo para adicionar regras de negócio antes da exclusão:
    
        livro = self.get_object()
        if livro.emprestimos.filter(data_devolucao_real__isnull=True).exists():
            return Response(
                {"erro": "Não é possível remover um livro com empréstimos em aberto."},
                status=status.HTTP_405_METHOD_NOT_ALLOWED,
            )
        
        #No caso acima, self.get_object() obtém o objeto que está sendo requisitado. O DRF utiliza o pk capturado pela URL para localizar esse objeto dentro do queryset. Verificamos se existem empréstimos em aberto. Se existir, interrompemos a exclusão. 
        return super().destroy(request, *args, **kwargs)
```

## 4.4 ViewSets: agrupando views relacionadas

Repare que `ListaLivrosView` e `DetalheLivroView` compartilham o mesmo `queryset` e o mesmo
`serializer_class` — só a operação HTTP muda. Um **ViewSet** junta tudo isso em uma única
classe:

```python
from rest_framework.viewsets import ModelViewSet


class LivroViewSet(ModelViewSet):
    queryset = Livro.objects.select_related("autor").prefetch_related("categorias")
    serializer_class = LivroSerializer
```

Isso sozinho já dá `list`, `create`, `retrieve`, `update`, `partial_update` e `destroy`.
Quando você **não** quer todas as operações (por exemplo, um recurso que nunca deveria ser
listado por completo, só criado/consultado individualmente), monte o ViewSet a partir dos
mixins específicos + `GenericViewSet`:

```python
from rest_framework import mixins, viewsets


class LeitorViewSet(
    mixins.CreateModelMixin,
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    viewsets.GenericViewSet,
):
    queryset = Leitor.objects.select_related("usuario")
    serializer_class = LeitorSerializer
```

Existe ainda `viewsets.ReadOnlyModelViewSet`, que só permite `list` e `retrieve` — útil para recursos consultáveis por qualquer um, mas gerenciáveis apenas pelo admin. Além disso, podemos ter o atributo http_method_names (exemplo: http_method_names = ["get", "delete"])

## 4.5 Routers: gerando as URLs automaticamente

Com ViewSets, você não escreve `path()` na mão — um **router** gera as rotas convencionais
a partir do ViewSet registrado:

```python
# biblioteca_config/urls.py
from rest_framework.routers import DefaultRouter
from catalogo.views import LivroViewSet, AutorViewSet, CategoriaViewSet

router = DefaultRouter()
router.register("livros", LivroViewSet, basename="livro")
router.register("autores", AutorViewSet, basename="autor")
router.register("categorias", CategoriaViewSet, basename="categoria")

urlpatterns = [
    path("api/", include(router.urls)), # Nesse caso mais simples, também poderia ser urlpatterns = router.urls
]
```

Isso gera automaticamente:

```
GET/POST      /api/livros/
GET/PUT/PATCH/DELETE   /api/livros/{pk}/
```

`DefaultRouter` (comparado a `SimpleRouter`) ainda adiciona uma página raiz da API
(`/api/`) listando os endpoints disponíveis, e permite pedir a resposta em formato
específico anexando `.json` na URL.

`basename` é necessário sempre que o ViewSet **não** define `queryset` como atributo fixo
(por exemplo, quando você sobrescreve `get_queryset()`), porque nesses casos o router não
tem como inferir sozinho o nome base das rotas.

## 4.6 Nested routers: recursos aninhados

Para um sub-recurso de verdade (ex.: as avaliações **de um livro específico**), usa-se o
pacote `drf-nested-routers`:

```bash
pip install drf-nested-routers
```

```python
from rest_framework_nested import routers
from catalogo.views import LivroViewSet, AvaliacaoViewSet

router = routers.DefaultRouter()
router.register("livros", LivroViewSet, basename="livro")

livros_router = routers.NestedDefaultRouter(router, "livros", lookup="livro")
livros_router.register("avaliacoes", AvaliacaoViewSet, basename="livro-avaliacoes")

urlpatterns = [
    path("api/", include(router.urls)),
    path("api/", include(livros_router.urls)),
]
```

Isso gera `/api/livros/{livro_pk}/avaliacoes/`. Dentro do ViewSet aninhado, você usa o
parâmetro de URL para filtrar (e, ao criar, para associar automaticamente ao pai), uma observação é que caso não existisse isso, ao criar uma review, teriamos que passar o produto, o que não faz muito sentido, é melhor que isso seja pego de maneira automática:

```python
class AvaliacaoViewSet(ModelViewSet):
    serializer_class = AvaliacaoSerializer

    def get_queryset(self):
        return Avaliacao.objects.filter(livro_id=self.kwargs["livro_pk"]) # Esse self.kwargs vem da url que chamou o viewset

    def get_serializer_context(self): # Método que serve para passar contexto adicional para o Serializer, caso necessárip
        return {**super().get_serializer_context(), "livro_id": self.kwargs["livro_pk"]} #o super.get_serializer_context() é necessário para passar outros contextos, como da requisição, por exemplo
```

```python
class AvaliacaoSerializer(serializers.ModelSerializer):
    class Meta:
        model = Avaliacao
        fields = ["id", "nota", "comentario"]

    def create(self, dados_validados):
        return Avaliacao.objects.create(
            livro_id=self.context["livro_id"], **dados_validados
        )
```



## 4.7 Filtros, busca e ordenação genéricos

Suponhamos que na nossa url nos quisessemos filtrar um livro por sua categoria (?categoria_id=x). Uma forma de fazer isso é:

### Filtro básico manual (sem biblioteca)

```python
class LivroViewSet(ModelViewSet):
    serializer_class = LivroSerializer

    def get_queryset(self):
        queryset = Livro.objects.select_related("autor")
        categoria_id = self.request.query_params.get("categoria_id")
        if categoria_id is not None:
            queryset = queryset.filter(categorias__id=categoria_id)
        return queryset
```

Isso funciona, mas escala mal se você precisar filtrar por vários campos e vários
operadores (`preco__gte`, `preco__lte`, etc.). É aí que entra o `django-filter`.

### `django-filter`: filtros declarativos

```bash
pip install django-filter
```

```python
# settings.py
INSTALLED_APPS += ["django_filters"]
```

```python
# catalogo/filters.py
import django_filters
from .models import Livro


class LivroFilter(django_filters.FilterSet):
    class Meta:
        model = Livro
        fields = {
            "categorias__id": ["exact"],
            "preco": ["gt", "lt"],
            "ano_publicacao": ["exact", "gt", "lt"],
        }
```

```python
# catalogo/views.py
from django_filters.rest_framework import DjangoFilterBackend


class LivroViewSet(ModelViewSet):
    queryset = Livro.objects.select_related("autor")
    serializer_class = LivroSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_class = LivroFilter
```

Isso habilita, sem nenhuma linha extra de lógica na view:

```
GET /api/livros/?categorias__id=2
GET /api/livros/?preco__gt=20&preco__lt=100
```

### Busca textual (`SearchFilter`)

```python
from rest_framework.filters import SearchFilter


class LivroViewSet(ModelViewSet):
    ...
    filter_backends = [DjangoFilterBackend, SearchFilter]
    filterset_class = LivroFilter
    search_fields = ["titulo", "isbn", "autor__nome"]
```

```
GET /api/livros/?search=duna
```

### Ordenação (`OrderingFilter`)

```python
from rest_framework.filters import OrderingFilter


class LivroViewSet(ModelViewSet):
    ...
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    ordering_fields = ["preco", "ano_publicacao", "titulo"]
```

```
GET /api/livros/?ordering=-preco
GET /api/livros/?ordering=preco,titulo
```

## 4.8 Paginação

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
} 

# OBS: Fazendo essa configuração via settings.py, não precisamos ir em cada viewset e colocar o atributo pagination_class = PageNumberPagination. Ele aplica para todos. O unico problema é que aplica para todos os ViewSets. Se quiser algo mais especifico, talvez seja interessante a abordagem que trarei mais a frente
```

Resposta paginada:

```json
{
  "count": 143,
  "next": "http://api.exemplo.com/livros/?page=2",
  "previous": null,
  "results": [ ... ]
}
```

Existem outras estratégias, escolhidas conforme o caso de uso:

| Classe | Comportamento | Bom para |
|---|---|---|
| `PageNumberPagination` | `?page=2` | listagens com navegação por página (mais comum) |
| `LimitOffsetPagination` | `?limit=10&offset=20` | clientes que controlam o próprio "scroll" |
| `CursorPagination` | cursor opaco, sem número de página | grandes volumes, dados que mudam com frequência (evita duplicar/pular itens entre páginas) |

Uma classe de paginação customizada (útil para fixar `page_size` sem precisar de uma
configuração global no settings.py, ou para reaproveitar em um único ViewSet):

```python
# catalogo/pagination.py
from rest_framework.pagination import PageNumberPagination


class PaginacaoPadrao(PageNumberPagination):
    page_size = 10
    page_size_query_param = "page_size"
    max_page_size = 50
```

```python
class LivroViewSet(ModelViewSet):
    ...
    pagination_class = PaginacaoPadrao
```

## 4.9 Serializer e contexto: passando dados extras

Serializers frequentemente precisam de informação que **não está** nos dados enviados pelo
cliente nem no modelo — como o usuário autenticado, ou um parâmetro de URL. O mecanismo
correto para isso é o `context`, populado por `get_serializer_context()`:

```python
class LivroViewSet(ModelViewSet):
    serializer_class = LivroSerializer

    def get_serializer_context(self):
        return {**super().get_serializer_context(), "usuario": self.request.user}
```

```python
class LivroSerializer(serializers.ModelSerializer):
    def create(self, dados_validados):
        usuario = self.context["usuario"]
        # ... usar 'usuario' na lógica de criação ...
        return Livro.objects.create(**dados_validados)
```

`context` **sempre** inclui automaticamente `request`, `view` e `format` — por isso
`HyperlinkedRelatedField` (seção 3.7) funciona sem configuração extra: ele lê a `request`
do contexto para montar URLs absolutas.

---

## Checklist de conceitos deste capítulo

- [ ] Entendo a evolução função → `APIView` → mixins → generic views → ViewSets
- [ ] Sei quando montar um `ViewSet` a partir de mixins específicos (nem sempre `ModelViewSet`)
- [ ] Sei registrar rotas com `DefaultRouter` e entendo quando preciso de `basename`
- [ ] Sei montar um recurso aninhado com `drf-nested-routers`
- [ ] Sei aplicar `django-filter`, `SearchFilter` e `OrderingFilter`
- [ ] Sei configurar e escolher entre as estratégias de paginação
- [ ] Sei usar `context` para passar dados (usuário, parâmetros de URL) ao serializer

## Exercícios propostos

1. Transforme as views de `Livro` (capítulo 3) em um `LivroViewSet` registrado via
   `DefaultRouter`.
2. Implemente `LivroFilter` permitindo filtrar por `categorias__id`, faixa de `preco` e
   `ano_publicacao`, e adicione `search_fields` para `titulo` e `autor__nome`.
3. Implemente um endpoint aninhado `/api/livros/{livro_pk}/avaliacoes/` usando
   `drf-nested-routers`, garantindo (via `context`) que uma avaliação criada nesse endpoint
   seja sempre associada ao livro da URL.
4. Configure `PageNumberPagination` com `page_size=10` só para `LivroViewSet`, sem afetar
   os demais endpoints do projeto.
