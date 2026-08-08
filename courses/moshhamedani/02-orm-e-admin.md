# 2. Django ORM Avançado e Django Admin

> Continuação direta do capítulo 1. Os exemplos usam os modelos `Autor`, `Categoria`,
> `Livro`, `Leitor` e `Emprestimo` definidos lá.

## 2.1 Managers e QuerySets: o coração do ORM

Todo modelo Django ganha automaticamente um atributo `objects`, que é um **Manager** — o
ponto de entrada para consultar e manipular aquela tabela.

```python
Livro.objects          # o Manager
Livro.objects.all()    # um QuerySet
```

A maioria dos métodos do Manager (`all`, `filter`, `exclude`, `order_by`...) **não
consulta o banco imediatamente** — eles devolvem um `QuerySet`, que é só uma representação
da consulta a ser feita. O `QuerySet` só é de fato **avaliado** (e o banco é consultado)
quando você:

- itera sobre ele (`for livro in queryset:`);
- converte para `list()`;
- acessa um índice ou faz *slicing* (`queryset[0]`, `queryset[:5]`);
- chama métodos que "materializam" resultado, como `len()`, `bool()`, `repr()`.

```python
queryset = Livro.objects.all()          # nenhuma query ainda
queryset = queryset.filter(ano_publicacao__gte=2020)  # ainda nenhuma query
livros = list(queryset)                 # AGORA a query é executada
```

Essa "preguiça" (*lazy evaluation*) é o que permite encadear `.filter().filter().order_by()`
sem custo de ir ao banco várias vezes — o Django só monta o SQL final quando você realmente
precisa dos dados.

Além disso, o `QuerySet` tem **cache interno**: se você avalia o mesmo queryset duas vezes,
a segunda vem do cache em memória, não do banco — **desde que você reaproveite a mesma
variável**:

```python
queryset = Livro.objects.all()
list(queryset)   # dispara 1 query, popula o cache
list(queryset)   # usa o cache, 0 queries novas

# Cuidado: isto dispara 2 queries, porque são dois querysets diferentes
list(Livro.objects.all())
list(Livro.objects.all())
```

## 2.2 Consultas básicas

```python
Livro.objects.all()                                   # todos
Livro.objects.get(pk=1)                                # um único (lança exceção se 0 ou 2+)
Livro.objects.filter(autor_id=3)                       # filtro simples
Livro.objects.exclude(quantidade_disponivel=0)         # negação
Livro.objects.filter(preco__gt=50).exclude(ano_publicacao__lt=2000)
```

`get()` **sempre** espera exatamente um resultado — se não encontrar nada, lança
`Livro.DoesNotExist`; se encontrar mais de um, lança `Livro.MultipleObjectsReturned`. Em
views, é comum tratar isso com o atalho:

```python
from django.shortcuts import get_object_or_404

livro = get_object_or_404(Livro, pk=livro_id)
```

## 2.3 Field lookups: o vocabulário de filtros

O padrão é `campo__lookup=valor` (dois underscores separam campo de operador):

| Lookup | Exemplo | Equivale a |
|---|---|---|
| `exact` (padrão) | `titulo__exact="Duna"` | `= 'Duna'` |
| `iexact` | `titulo__iexact="duna"` | igual, sem diferenciar maiúsc./minúsc. |
| `contains` / `icontains` | `titulo__icontains="rei"` | `LIKE '%rei%'` |
| `startswith` / `istartswith` | | `LIKE 'rei%'` |
| `gt`, `gte`, `lt`, `lte` | `preco__gte=30` | `>=` |
| `range` | `preco__range=(10, 50)` | `BETWEEN` |
| `in` | `id__in=[1, 2, 3]` | `IN (...)` |
| `isnull` | `capa_url__isnull=True` | `IS NULL` |
| `year`, `month`, `day` | `criado_em__year=2024` | extrai parte de data |

Você também navega relacionamentos com o mesmo `__`:

```python
# livros de autores brasileiros (supondo um campo 'pais' em Autor)
Livro.objects.filter(autor__pais="Brasil")

# livros com estoque baixo dentro de uma categoria específica
Livro.objects.filter(categorias__nome="Programação", quantidade_disponivel__lt=3)
```

## 2.4 Combinando condições: `Q` e `F`

`filter()` com múltiplos argumentos, ou `.filter().filter()` encadeado, sempre combina
condições com **E** (`AND`). Para **OU** (`OR`) ou negação complexa, use `Q`:

```python
from django.db.models import Q

# livros de programação OU com preço abaixo de 20
Livro.objects.filter(Q(categorias__nome="Programação") | Q(preco__lt=20))

# livros que NÃO são de ficção
Livro.objects.filter(~Q(categorias__nome="Ficção"))
```

`F` permite comparar (ou operar sobre) o valor de **outro campo do mesmo registro**, em
vez de um valor fixo em Python:

```python
from django.db.models import F

# livros em que a quantidade disponível é igual à quantidade total (nenhum emprestado)
Livro.objects.filter(quantidade_disponivel=F("quantidade_total"))

# reajuste de preço de 10% direto no banco, sem trazer os objetos para o Python
Livro.objects.update(preco=F("preco") * 1.10)
```

Usar `F` para o `update()` acima é importante por um motivo prático: se você fizesse
`livro.preco = livro.preco * 1.10; livro.save()` em Python, cada linha exigiria ler +
gravar; com `F`, a operação inteira roda **dentro do banco**, em uma única instrução SQL,
evitando problemas de concorrência (dois processos lendo o mesmo valor "antigo" ao mesmo
tempo — *race condition*).

## 2.5 Ordenação e paginação manual

```python
Livro.objects.order_by("titulo")            # ascendente
Livro.objects.order_by("-preco")            # descendente
Livro.objects.order_by("autor__nome", "-ano_publicacao")   # múltiplos critérios

Livro.objects.order_by("preco")[:10]        # primeiros 10 (página 1)
Livro.objects.order_by("preco")[10:20]      # próximos 10 (página 2)
```

*Slicing* em um `QuerySet` vira `LIMIT`/`OFFSET` no SQL — não traz tudo para a memória
antes de cortar.

## 2.6 `select_related` x `prefetch_related`: resolvendo o problema N+1

Este é, disparado, o ponto de performance mais importante do ORM. Considere:

```python
for livro in Livro.objects.all():
    print(livro.autor.nome)   # PERIGO
```

Esse laço dispara **1 query** para pegar os livros e mais **1 query por livro** para pegar
o autor correspondente — se houver 500 livros, são 501 queries. Isso é o chamado
**problema N+1**, e é a causa nº 1 de APIs Django lentas em produção.

A solução depende do tipo de relação:

- **`select_related`** — para relações onde "do outro lado tem só um" (`ForeignKey`,
  `OneToOneField`). Internamente vira um `JOIN` SQL — tudo em uma única query.

```python
livros = Livro.objects.select_related("autor").all()
for livro in livros:
    print(livro.autor.nome)   # nenhuma query extra
```

- **`prefetch_related`** — para relações "do outro lado tem vários"
  (`ManyToManyField`, ou a ponta "muitos" de uma `ForeignKey` reversa). Internamente
  faz uma **segunda query separada** (com `IN (...)`) e junta os resultados em Python
  (um `JOIN` aqui multiplicaria linhas indevidamente).

```python
livros = Livro.objects.prefetch_related("categorias").all()
for livro in livros:
    print([c.nome for c in livro.categorias.all()])   # nenhuma query extra
```

Você pode combinar os dois e ainda "atravessar" relações com `__`:

```python
Emprestimo.objects.select_related("leitor__usuario", "livro").prefetch_related(
    "livro__categorias"
)
```

> **Hábito recomendado:** sempre que uma view acessa um relacionamento dentro de um laço
> (ou dentro de um `serializer_method_field`, como veremos no capítulo 3), pare e pergunte
> "isso vai gerar N+1?". Abra o Django Debug Toolbar e confirme o número de queries antes
> de considerar a view "pronta".

## 2.7 `values()`, `values_list()`, `only()` e `defer()`

Por padrão, uma consulta traz **instâncias completas** do modelo. Às vezes você só precisa
de alguns campos:

```python
Livro.objects.values("id", "titulo")
# [{'id': 1, 'titulo': 'Duna'}, {'id': 2, 'titulo': '1984'}, ...]   -> dicionários

Livro.objects.values_list("id", "titulo")
# [(1, 'Duna'), (2, '1984'), ...]   -> tuplas

Livro.objects.values_list("titulo", flat=True)
# ['Duna', '1984', ...]   -> lista simples (só funciona com 1 campo)
```

`only()` e `defer()` continuam devolvendo **instâncias do modelo** (não dicionários), mas
controlam quais campos são carregados de imediato:

```python
Livro.objects.only("id", "titulo")     # só carrega esses campos de cara
Livro.objects.defer("sinopse")         # carrega tudo, menos esse campo
```

**Cuidado:** se você usa `only()`/`defer()` e depois acessa, dentro de um laço, um campo
que não foi carregado, o Django dispara **uma query extra por objeto** para buscar aquele
campo — reintroduzindo o problema N+1 de outra forma. Use essas otimizações só quando tiver
certeza de quais campos serão realmente acessados.

## 2.8 Agregação x Anotação

- **Agregação** (`aggregate()`) resume o **queryset inteiro** em um único dicionário.
- **Anotação** (`annotate()`) calcula um valor **por registro**, adicionando um "campo
  virtual" a cada objeto do queryset.

```python
from django.db.models import Count, Avg, Sum, Min, Max

# agregação: um resumo só
Livro.objects.aggregate(total=Count("id"), preco_medio=Avg("preco"))
# {'total': 120, 'preco_medio': Decimal('42.50')}

# anotação: um valor por autor
Autor.objects.annotate(total_livros=Count("livros"))
for autor in Autor.objects.annotate(total_livros=Count("livros")):
    print(autor.nome, autor.total_livros)
```

`Value`, `F`, `Func` e `ExpressionWrapper` permitem montar expressões mais ricas dentro de
`annotate()`:

```python
from django.db.models import Value, F, ExpressionWrapper, DecimalField
from django.db.models.functions import Concat

Autor.objects.annotate(
    nome_completo_formatado=Concat(Value("Autor: "), F("nome")),
)

Livro.objects.annotate(
    preco_com_desconto=ExpressionWrapper(
        F("preco") * 0.9,
        output_field=DecimalField(max_digits=8, decimal_places=2),
    )
)
```

`ExpressionWrapper` é necessário sempre que o Django não consegue inferir sozinho o tipo do
resultado de uma expressão mista (aqui, `DecimalField * float`).

## 2.9 Criando, atualizando e apagando objetos

```python
# criar (forma explícita — recomendada para clareza e para IntelliSense)
livro = Livro(titulo="Duna", isbn="9780593099322", autor=autor, ano_publicacao=1965,
              preco=59.90, quantidade_total=5, quantidade_disponivel=5)
livro.save()

# atalho equivalente
livro = Livro.objects.create(
    titulo="Duna", isbn="9780593099322", autor=autor,
    ano_publicacao=1965, preco=59.90,
)

# atualizar UM objeto já carregado (sempre leia antes de atualizar campo a campo)
livro = Livro.objects.get(pk=1)
livro.preco = 64.90
livro.save()

# atualizar EM MASSA, direto no banco, sem instanciar objetos
Livro.objects.filter(categorias__nome="Ficção").update(quantidade_disponivel=F("quantidade_disponivel") - 1)

# apagar
livro.delete()
Livro.objects.filter(quantidade_disponivel=0, quantidade_total=0).delete()

# inserir vários de uma vez (uma única query, muito mais rápido que um loop com .create())
Categoria.objects.bulk_create([
    Categoria(nome="Fantasia"),
    Categoria(nome="Biografia"),
])
```

> **Armadilha clássica:** `Model(**kwargs); model.save()` grava **todos os campos**,
> inclusive os que não foram explicitamente alterados (usando o valor que já estava em
> memória). Se você monta um objeto parcialmente e chama `.save()`, pode sobrescrever
> campos com valores "vazios" sem querer. A forma segura de atualizar só um campo
> específico é ler o objeto do banco primeiro, ou usar `.update()` no queryset (que gera
> `UPDATE` só com os campos informados).

## 2.10 Transações: garantindo atomicidade

Quando uma operação de negócio envolve **múltiplas gravações relacionadas**, você precisa
garantir que ou tudo é salvo, ou nada é — senão o banco fica em estado inconsistente se
algo falhar no meio do caminho.

```python
from django.db import transaction

@transaction.atomic
def registrar_emprestimo(leitor, livro):
    if livro.quantidade_disponivel < 1:
        raise ValueError("Livro indisponível")

    livro.quantidade_disponivel = F("quantidade_disponivel") - 1
    livro.save(update_fields=["quantidade_disponivel"])

    return Emprestimo.objects.create(leitor=leitor, livro=livro)
```

Também é possível usar como *context manager*, controlando apenas um trecho da função:

```python
def alguma_view(request):
    # ... código que não precisa estar na transação ...
    with transaction.atomic():
        pedido = Pedido.objects.create(...)
        ItemPedido.objects.bulk_create([...])
    # ... resto do código ...
```

Se qualquer exceção não tratada acontecer dentro do bloco `atomic`, o Django **desfaz**
(*rollback*) tudo o que foi feito ali dentro.

## 2.11 SQL bruto (quando o ORM não é suficiente)

Para consultas muito complexas, onde o ORM geraria SQL ineficiente ou incompreensível, é
válido escrever SQL manualmente — **como exceção, não como regra**:

```python
# mapeando resultado de volta para instâncias do modelo
livros = Livro.objects.raw("SELECT * FROM catalogo_livro WHERE preco > %s", [50])

# acesso total ao banco, sem passar pelo ORM
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("SELECT autor_id, COUNT(*) FROM catalogo_livro GROUP BY autor_id")
    resultado = cursor.fetchall()
```

> **Regra de ouro:** premature optimization é a raiz de muitos problemas. Escreva primeiro
> com o ORM, meça (com o Debug Toolbar ou `EXPLAIN`), e só desça para SQL bruto se houver
> um gargalo comprovado que o ORM não resolve bem.

---

## 2.12 Django Admin: o painel gerado automaticamente

Registrar um modelo já dá um CRUD completo pronto:

```python
# catalogo/admin.py
from django.contrib import admin
from .models import Autor, Categoria, Livro


@admin.register(Autor)
class AutorAdmin(admin.ModelAdmin):
    list_display = ["nome", "data_nascimento"]
    search_fields = ["nome"]


admin.site.register(Categoria)
```

### Customizando a listagem

```python
@admin.register(Livro)
class LivroAdmin(admin.ModelAdmin):
    list_display = ["titulo", "autor", "preco", "status_estoque"]
    list_editable = ["preco"]              # editável direto na listagem
    list_filter = ["categorias", "ano_publicacao"]
    search_fields = ["titulo", "isbn"]
    list_per_page = 20
    list_select_related = ["autor"]        # evita N+1 na coluna 'autor'
    autocomplete_fields = ["autor", "categorias"]

    @admin.display(ordering="quantidade_disponivel", description="Estoque")
    def status_estoque(self, livro):
        return "Baixo" if livro.quantidade_disponivel < 3 else "OK"
```

Pontos-chave:

- `list_editable` exige que o campo também esteja em `list_display`.
- `list_select_related` é o equivalente, no admin, do `select_related` da seção 2.6 —
  sem ele, cada linha da listagem dispararia uma query extra para buscar o autor.
- Colunas calculadas (que não existem como campo do modelo) são métodos decorados com
  `@admin.display(...)`; o parâmetro `ordering` é o que permite que a coluna também seja
  clicável para ordenar.
- `autocomplete_fields` troca um `<select>` gigante (ruim para milhares de registros) por
  uma busca com autocomplete — só funciona se o `ModelAdmin` do modelo relacionado (aqui,
  `Autor`) também definir `search_fields`.

### Inlines: editando "filhos" dentro do "pai"

```python
class LivroInline(admin.TabularInline):
    model = Livro
    extra = 0
    fields = ["titulo", "isbn"]


@admin.register(Autor)
class AutorAdmin(admin.ModelAdmin):
    inlines = [LivroInline]
```

`TabularInline` mostra os registros filhos em formato de tabela (compacto);
`StackedInline` mostra cada um como um formulário separado (melhor quando há muitos
campos). `extra = 0` remove as linhas em branco extras que o Django mostra por padrão.

### Ações customizadas

```python
@admin.register(Livro)
class LivroAdmin(admin.ModelAdmin):
    actions = ["zerar_estoque"]

    @admin.action(description="Zerar estoque disponível dos livros selecionados")
    def zerar_estoque(self, request, queryset):
        atualizados = queryset.update(quantidade_disponivel=0)
        self.message_user(request, f"{atualizados} livro(s) atualizados.")
```

### Filtros customizados

```python
class EstoqueBaixoFilter(admin.SimpleListFilter):
    title = "estoque"
    parameter_name = "estoque"

    def lookups(self, request, model_admin):
        return [("baixo", "Baixo (< 3)")]

    def queryset(self, request, queryset):
        if self.value() == "baixo":
            return queryset.filter(quantidade_disponivel__lt=3)
        return queryset


@admin.register(Livro)
class LivroAdmin(admin.ModelAdmin):
    list_filter = [EstoqueBaixoFilter]
```

### Sobrescrevendo o queryset base (para colunas calculadas mais complexas)

```python
@admin.register(Autor)
class AutorAdmin(admin.ModelAdmin):
    list_display = ["nome", "total_livros"]

    def get_queryset(self, request):
        return super().get_queryset(request).annotate(total_livros=Count("livros"))

    @admin.display(ordering="total_livros", description="Livros")
    def total_livros(self, autor):
        return autor.total_livros
```

### Validação de dados no admin

Validações declaradas no `models.py` (via `validators=[...]`, `blank=False`, etc.) já são
respeitadas automaticamente pelos formulários do admin — não é preciso reescrevê-las lá.

```python
from django.core.validators import MinValueValidator

preco = models.DecimalField(
    max_digits=8, decimal_places=2,
    validators=[MinValueValidator(0.01)],
)
```

### Mantendo apps reutilizáveis também no admin

Assim como discutimos na seção 1.5 sobre não acoplar apps entre si, o mesmo cuidado vale
para customizações do admin: se um app "genérico" (como um sistema de avaliações
reutilizável) precisar customizar o admin de um modelo de **outro** app específico do
projeto, essa customização deveria morar no app `core` (específico do projeto), e não
dentro do app genérico — assim ele continua podendo ser copiado para outro projeto sem
carregar dependências desnecessárias.

---

## Checklist de conceitos deste capítulo

- [ ] Entendo *lazy evaluation* de QuerySets e quando eles são avaliados
- [ ] Sei usar os principais *lookups* (`__gt`, `__contains`, `__in`, etc.)
- [ ] Sei quando usar `Q` (OR/NOT) e `F` (comparar/operar campos)
- [ ] Sei identificar e resolver problema N+1 com `select_related`/`prefetch_related`
- [ ] Sei a diferença entre agregação e anotação
- [ ] Sei envolver operações relacionadas em `transaction.atomic`
- [ ] Consigo customizar list_display, filtros, buscas, inlines e ações no Admin

## Exercícios propostos

1. Escreva uma consulta que retorne, para cada `Autor`, o número de livros e o preço médio
   dos seus livros, ordenado do autor com mais livros para o com menos.
2. Escreva uma consulta que traga todos os `Emprestimo` ativos junto com `leitor__usuario`
   e `livro` em uma única query (use o Debug Toolbar para confirmar).
3. Implemente, no `LivroAdmin`, uma ação customizada "Marcar como esgotado" que zera
   `quantidade_disponivel` dos livros selecionados e mostra uma mensagem de sucesso.
4. Implemente um filtro customizado no Admin de `Emprestimo` para listar só os empréstimos
   com `data_devolucao_prevista` no passado e ainda sem `data_devolucao_real` (atrasados).
