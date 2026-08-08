# 3. Django REST Framework — Fundamentos

## 3.1 O que é uma API REST

REST (*Representational State Transfer*) é um conjunto de convenções para expor dados pela
web de forma previsível. Três ideias carregam praticamente tudo:

1. **Recursos**: cada "coisa" do seu domínio (um livro, um autor, um empréstimo) é
   endereçada por uma URL própria. Coleções ficam no plural, itens individuais recebem um
   identificador:

   ```
   GET /api/livros/          -> coleção de livros
   GET /api/livros/42/       -> um livro específico
   GET /api/livros/42/avaliacoes/   -> sub-recurso aninhado (no máximo 2 níveis, por clareza)
   ```

2. **Representações**: o servidor nunca devolve o objeto Python "puro" — ele devolve uma
   *representação* dele (quase sempre **JSON** hoje em dia). É perfeitamente normal (e às
   vezes desejável) que essa representação externa seja diferente da estrutura interna do
   banco: você pode omitir campos sensíveis, renomear campos, ou incluir campos calculados
   que não existem como coluna.

3. **Métodos HTTP como verbos de ação**:

   | Método | Uso | Idempotente? |
   |---|---|---|
   | `GET` | ler um recurso ou coleção | sim |
   | `POST` | criar um recurso | não |
   | `PUT` | substituir um recurso **por completo** | sim |
   | `PATCH` | atualizar **parte** de um recurso | não, tecnicamente, mas na prática costuma ser |
   | `DELETE` | remover um recurso | sim |

   *Idempotente* significa "repetir a mesma requisição várias vezes tem o mesmo efeito que
   fazê-la uma vez" — importante para decidir, por exemplo, se um endpoint de atualização de
   quantidade deveria ser `PATCH` (troca o valor absoluto) e não algo que soma a cada
   chamada.

### JSON, rapidamente

```json
{
  "id": 42,
  "titulo": "Duna",
  "preco": 59.90,
  "disponivel": true,
  "autor": {"id": 3, "nome": "Frank Herbert"},
  "categorias": ["Ficção", "Ficção Científica"]
}
```

Chaves sempre entre aspas duplas; valores podem ser string, número, booleano, objeto
aninhado, array, ou `null`.

## 3.2 Instalando e configurando o DRF

```bash
pip install djangorestframework
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    "rest_framework",
    "catalogo",
    "emprestimos",
]
```

## 3.3 A forma mais simples: `@api_view`

O DRF fornece uma versão "turbinada" de `HttpRequest`/`HttpResponse`, junto com o decorador
`@api_view`, que já cuida de negociação de conteúdo (JSON por padrão) e gera aquela
interface navegável no navegador (*browsable API*) — muito útil para testar endpoints
manualmente durante o desenvolvimento.

```python
# catalogo/views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from .models import Livro
from .serializers import LivroSerializer


@api_view(["GET", "POST"])
def lista_livros(request):
    if request.method == "GET":
        livros = Livro.objects.select_related("autor").all()
        serializer = LivroSerializer(livros, many=True)
        return Response(serializer.data)

    serializer = LivroSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    serializer.save()
    return Response(serializer.data, status=status.HTTP_201_CREATED)


@api_view(["GET", "PUT", "DELETE"])
def detalhe_livro(request, pk):
    livro = get_object_or_404(Livro, pk=pk)

    if request.method == "GET":
        return Response(LivroSerializer(livro).data)

    if request.method == "PUT":
        serializer = LivroSerializer(livro, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)

    if request.method == "DELETE":
        livro.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

```python
# catalogo/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path("livros/", views.lista_livros),
    path("livros/<int:pk>/", views.detalhe_livro),
]
```

Note os **códigos de status** usados de propósito: `201 Created` ao criar,
`204 No Content` ao apagar (sem corpo na resposta). Usar o código certo não é
"frescura" — é o que permite que qualquer cliente HTTP genérico (não só o seu front-end)
entenda o resultado da operação sem precisar interpretar o corpo da resposta.

## 3.4 Serializers: a ponte entre modelo e JSON

Um `Serializer` faz dois trabalhos opostos:

- **Serialização**: modelo Python → estrutura simples (dict) → JSON.
- **Deserialização**: JSON recebido → dict validado → modelo Python.

### Serializer manual (`Serializer`)

Útil quando a representação externa **não corresponde 1:1** a nenhum modelo (ex.: um
formulário de login, ou o "criar reserva" que veremos no capítulo 5):

```python
# catalogo/serializers.py
from rest_framework import serializers


class LivroSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    titulo = serializers.CharField(max_length=255)
    isbn = serializers.CharField(max_length=13)
    preco = serializers.DecimalField(max_digits=8, decimal_places=2)
```

### `ModelSerializer`: o caminho recomendado no dia a dia

Quando a representação externa é (quase) um espelho do modelo, `ModelSerializer` evita
duplicar a definição de cada campo e suas validações:

```python
class LivroSerializer(serializers.ModelSerializer):
    class Meta:
        model = Livro
        fields = ["id", "titulo", "isbn", "ano_publicacao", "preco", "autor"]
```

> **Nunca use `fields = "__all__"`.** Parece um atalho inofensivo, mas significa que
> **qualquer** campo novo adicionado ao modelo no futuro passa a ser exposto pela API
> automaticamente, sem revisão — inclusive campos sensíveis. Liste sempre os campos
> explicitamente.

Campos que não existem no modelo (calculados) são declarados à parte e referenciados na
`Meta.fields`:

```python
class LivroSerializer(serializers.ModelSerializer):
    preco_com_desconto = serializers.SerializerMethodField()

    class Meta:
        model = Livro
        fields = ["id", "titulo", "preco", "preco_com_desconto"]

    def get_preco_com_desconto(self, livro: Livro):
        return round(livro.preco * Decimal("0.9"), 2)
```

## 3.5 Validação de dados

**Nível de campo** (usa a convenção `validate_<nome_do_campo>`):

```python
class LivroSerializer(serializers.ModelSerializer):
    class Meta:
        model = Livro
        fields = ["id", "titulo", "isbn", "ano_publicacao"]

    def validate_isbn(self, valor):
        if not valor.isdigit():
            raise serializers.ValidationError("O ISBN deve conter apenas dígitos.")
        return valor
```

**Nível de objeto** (quando a regra envolve mais de um campo ao mesmo tempo):

```python
    def validate(self, dados):
        if dados["quantidade_disponivel"] > dados["quantidade_total"]:
            raise serializers.ValidationError(
                "Quantidade disponível não pode ser maior que a total."
            )
        return dados
```

**Validators reaproveitáveis do próprio modelo** (ex.: `MinValueValidator`) já são
aplicados automaticamente quando você usa `ModelSerializer` — não é preciso repeti-los.

## 3.6 Customizando `create()` e `update()`

Por padrão, `ModelSerializer.save()` já sabe criar/atualizar um objeto a partir dos dados
validados. Você sobrescreve esse comportamento quando a gravação envolve lógica além de
"um `INSERT`/`UPDATE` direto":

```python
class LivroSerializer(serializers.ModelSerializer):
    class Meta:
        model = Livro
        fields = ["id", "titulo", "isbn", "quantidade_total", "quantidade_disponivel"]

    def create(self, dados_validados):
        # ao cadastrar, a quantidade disponível começa igual à quantidade total
        dados_validados["quantidade_disponivel"] = dados_validados["quantidade_total"]
        return Livro.objects.create(**dados_validados)
```

O importante é sempre **retornar a instância** criada/atualizada ao final — é isso que o
restante do framework (a view, a resposta) espera receber de volta.

## 3.7 Representando relacionamentos em serializers

Esse é um dos pontos de maior impacto no design de uma API: como representar
`livro.autor` na saída JSON? O DRF oferece quatro estratégias:

**1) Chave primária** (mais compacto, exige uma chamada extra do cliente para ver detalhes):

```python
autor = serializers.PrimaryKeyRelatedField(queryset=Autor.objects.all())
```

```json
{"id": 1, "titulo": "Duna", "autor": 3}
```

**2) String** (usa o `__str__` do modelo relacionado — só leitura):

```python
autor = serializers.StringRelatedField()
```

```json
{"id": 1, "titulo": "Duna", "autor": "Frank Herbert"}
```

**3) Objeto aninhado** (mais completo, mais "pesado"):

```python
class AutorResumidoSerializer(serializers.ModelSerializer):
    class Meta:
        model = Autor
        fields = ["id", "nome"]


class LivroSerializer(serializers.ModelSerializer):
    autor = AutorResumidoSerializer(read_only=True)

    class Meta:
        model = Livro
        fields = ["id", "titulo", "autor"]
```

```json
{"id": 1, "titulo": "Duna", "autor": {"id": 3, "nome": "Frank Herbert"}}
```

**4) Hyperlink** (o cliente segue links, no espírito mais "puro" de REST — exige que a
view correspondente tenha `name=` registrado no roteador):

```python
autor = serializers.HyperlinkedRelatedField(
    queryset=Autor.objects.all(), view_name="autor-detail"
)
```

```json
{"id": 1, "titulo": "Duna", "autor": "http://api.exemplo.com/autores/3/"}
```

> Para relacionamentos N:N, o mesmo raciocínio vale usando `many=True` no serializer
> aninhado:
> ```python
> categorias = CategoriaSerializer(many=True, read_only=True)
> ```

### Um princípio importante de design de API

A representação de **entrada** (o que o cliente envia para criar/atualizar) nem sempre
deveria ser a mesma da representação de **saída** (o que ele recebe de volta). Um exemplo
concreto: ao **criar** um livro, faz sentido o cliente enviar `autor_id` (um inteiro); ao
**ler** um livro, faz mais sentido devolver o objeto `autor` completo aninhado. Quando essa
diferença fica grande, a solução mais limpa é ter **serializers diferentes por operação**
(voltamos a isso com profundidade no capítulo 4, na seção sobre `get_serializer_class`).

```python
class CriarLivroSerializer(serializers.ModelSerializer):
    class Meta:
        model = Livro
        fields = ["id", "titulo", "isbn", "ano_publicacao", "preco", "autor"]
        # 'autor' aqui vira PrimaryKeyRelatedField automaticamente


class LivroSerializer(serializers.ModelSerializer):
    autor = AutorResumidoSerializer(read_only=True)

    class Meta:
        model = Livro
        fields = ["id", "titulo", "isbn", "ano_publicacao", "preco", "autor"]
```

---

## Checklist de conceitos deste capítulo

- [ ] Entendo o que caracteriza uma API REST (recursos, representações, verbos HTTP)
- [ ] Sei escrever uma view com `@api_view` tratando `GET`/`POST`/`PUT`/`DELETE`
- [ ] Sei a diferença entre `Serializer` e `ModelSerializer`, e quando usar cada um
- [ ] Sei validar campos individuais (`validate_<campo>`) e o objeto inteiro (`validate`)
- [ ] Sei customizar `create()`/`update()` quando a gravação exige lógica extra
- [ ] Conheço as 4 formas de representar relacionamentos (PK, string, aninhado, hyperlink)
  e sei justificar qual usar em cada situação

## Exercícios propostos

1. Crie `AutorSerializer`, `CategoriaSerializer` e `LivroSerializer` (este último com
   `autor` como objeto aninhado somente leitura e `categorias` como lista de strings).
2. Escreva as views `lista_livros`/`detalhe_livro` com `@api_view`, incluindo validação
   que impede `quantidade_disponivel` maior que `quantidade_total`.
3. Adicione um campo calculado `esgotado` (booleano) ao `LivroSerializer`, usando
   `SerializerMethodField`.
4. Crie um serializer específico `CriarEmprestimoSerializer` que recebe apenas `livro_id`
   (o `leitor` viria futuramente do usuário autenticado — assunto do capítulo 6) e, no
   `create()`, decremente `quantidade_disponivel` do livro dentro de uma
   `transaction.atomic()`.
