# 1. Fundamentos do Django e Modelagem de Dados

## 1.1 O que é Django e por que ele é tão usado

Django é um **framework web em Python**, focado em produtividade: ele entrega, prontos,
uma porção de recursos que normalmente você teria que construir na mão em qualquer projeto
web — por isso é chamado de framework *"baterias inclusas"* (*batteries included*):

- **ORM** (Object-Relational Mapper): você manipula o banco de dados escrevendo Python,
  não SQL.
- **Painel administrativo** pronto, gerado automaticamente a partir dos seus modelos.
- **Sistema de autenticação** completo (usuários, grupos, permissões, sessões).
- **Sistema de migrations** para versionar o esquema do banco.
- Proteções de segurança padrão (CSRF, SQL injection, XSS) já configuradas.

Isso não significa que Django seja "melhor" que outros frameworks (Flask, FastAPI, Express,
ASP.NET) em todos os cenários — significa que ele otimiza para *entregar rápido* projetos
que precisam de estrutura de dados relacional + admin + autenticação, que é o caso da
maioria dos sistemas de backend "CRUD-heavy" (e-commerce, sistemas internos, backends de
apps mobile, etc.).

> **Regra prática:** escolher framework não é sobre qual é "mais rápido" em benchmark, e
> sim sobre qual reduz mais a quantidade de código que *você* precisa escrever e manter
> para o problema que você tem. Framework "chato e maduro" que resolve 90% do seu problema
> de graça geralmente vence framework "na moda" que resolve 60%.

## 1.2 Conceitos de desenvolvimento web que todo backend dev precisa saber

Antes de tocar em código, vale alinhar vocabulário:

- **Front-end**: roda no navegador/app do cliente. Responsável por *apresentar* dados.
- **Back-end**: roda no servidor. Responsável por *processar* dados, aplicar regras de
  negócio e persistir informação.
- **URL** (Uniform Resource Locator): endereço que localiza um recurso na web
  (`https://biblioteca.com/livros/42`).
- **HTTP**: protocolo de requisição/resposta usado entre cliente e servidor. Toda
  interação é: o cliente manda uma **requisição** (request), o servidor devolve uma
  **resposta** (response).
- **HTML**: quando o servidor monta a página inteira e devolve pronta para o navegador
  exibir (renderização no servidor / *server-side rendering*).
- **JSON**: quando o servidor devolve só os *dados*, e um front-end separado (React, Vue,
  app mobile) é quem monta a interface (essa é a abordagem que este material usa a partir
  do capítulo 3, construindo uma **API**).

Nos projetos modernos, a tendência é o backend virar puramente um "provedor de dados": ele
expõe **endpoints** (pontos de entrada de uma API) e delega a montagem da tela para o
cliente. É o modelo que vamos seguir aqui: Django cuidando de modelos, regras de negócio e
API; qualquer front-end (que não é o foco deste material) consumindo essa API.

## 1.3 Preparando o ambiente

```bash
# 1. Confirme a versão do Python
python --version

# 2. Crie e ative um ambiente virtual (isola as dependências deste projeto)
python -m venv .venv
source .venv/bin/activate        # Linux/Mac
# .venv\Scripts\activate         # Windows

# 3. Instale o Django
pip install django

# 4. Confirme a instalação
python -m django --version
```

> **Por que ambiente virtual?** Sem ele, todos os pacotes Python instalados ficam globais
> na sua máquina. Isso vira um problema assim que você tem dois projetos que precisam de
> versões diferentes da mesma biblioteca. Cada projeto Django deveria ter o seu próprio
> ambiente isolado.

## 1.4 Criando o primeiro projeto

```bash
django-admin startproject biblioteca_config .
```

O `.` no final é importante: ele diz para o Django não criar uma pasta extra em volta do
projeto, evitando aquela duplicação de diretórios (`biblioteca/biblioteca/`) que só
atrapalha.

Estrutura gerada:

```
manage.py
biblioteca_config/
    __init__.py
    settings.py     # todas as configurações do projeto
    urls.py         # roteamento principal (URLconf raiz)
    wsgi.py         # ponto de entrada para servidores WSGI (produção)
    asgi.py         # ponto de entrada assíncrono (WebSockets, etc.)
```

- `manage.py` é um atalho para `django-admin` que já sabe qual `settings.py` usar. A partir
  de agora, **toda interação com o projeto passa por ele**: `python manage.py <comando>`.
- `settings.py` é o "painel de controle" do projeto: apps instalados, banco de dados,
  idioma, fuso horário, middlewares, etc.

Subindo o servidor de desenvolvimento:

```bash
python manage.py runserver 8000
```

Se aparecer um aviso de *"you have unapplied migrations"*, não se preocupe ainda — isso é
assunto da seção 1.9.

## 1.5 Apps: a unidade de organização de um projeto Django

Um projeto Django é composto por **apps** — módulos independentes, cada um responsável por
uma fatia de funcionalidade. Pense em apps como os aplicativos do seu celular: cada um faz
uma coisa e faz bem.

```bash
python manage.py startapp catalogo
```

Isso gera:

```
catalogo/
    __init__.py
    admin.py         # customização do painel admin para este app
    apps.py          # configuração do app
    migrations/       # histórico de mudanças no esquema do banco
    models.py         # classes que representam tabelas
    tests.py
    views.py          # request handlers deste app
```

Depois de criar o app, é **obrigatório** registrá-lo em `settings.py`:

```python
# biblioteca_config/settings.py
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",

    # apps do projeto
    "catalogo",
]
```

### Como decidir o tamanho de um app (design)

Esse é um ponto sutil, mas importante, e que costuma ser negligenciado por quem está
começando:

- **App gigante** ("monolito"): um único app com todos os modelos do sistema. Fácil no
  começo, vira um pesadelo de manutenção conforme o projeto cresce — tudo depende de tudo.
- **Apps micro demais**: cada tabelinha em um app separado. O resultado costuma ser
  **acoplamento em cadeia** (app A depende de B, que depende de C), o que destrói a
  possibilidade de reaproveitar qualquer um deles isoladamente.
- **Equilíbrio (recomendado):** agrupe em um mesmo app tudo que **sempre** anda junto e faz
  sentido ser reutilizado em bloco. No nosso domínio, por exemplo:
  - `catalogo` → `Autor`, `Categoria`, `Livro` (o catálogo não faz sentido existir pela
    metade).
  - `emprestimos` → `Leitor`, `Emprestimo` (depende do catálogo, mas é um domínio à parte).
  - `core` → tudo que é **específico deste projeto** e não deveria ser reaproveitado em
    outro (usuário customizado, integrações entre apps, configurações gerais).

> **Heurística:** "será que eu conseguiria copiar esta pasta `app/` para outro projeto e
> ela continuar fazendo sentido sozinha, sem eu ter que arrastar dez outras pastas junto?"
> Se a resposta for não, provavelmente o corte dos apps está errado.

## 1.6 Views: a peça que responde a uma requisição

No vocabulário do Django, uma **view** não é a tela — é a função (ou classe) que recebe uma
`request` e devolve uma `response`. É, na prática, um *request handler*. Isso costuma
confundir quem vem de outros frameworks, onde "view" normalmente é o HTML.

```python
# catalogo/views.py
from django.http import HttpResponse

def listar_livros(request):
    return HttpResponse("Lista de livros (ainda estático)")
```

## 1.7 Roteamento: ligando URLs a views

Cada app pode ter seu próprio arquivo `urls.py` (URLconf local), que depois é incluído no
`urls.py` raiz do projeto.

```python
# catalogo/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path("livros/", views.listar_livros, name="listar_livros"),
]
```

```python
# biblioteca_config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("catalogo/", include("catalogo.urls")),
]
```

Com isso, `GET /catalogo/livros/` cai em `listar_livros`. Repare no padrão: o `include()`
"corta" o prefixo `catalogo/` e repassa o resto da URL para o URLconf do app — é assim que
mantemos cada app dono das suas próprias rotas, sem o projeto principal precisar conhecer
os detalhes internos de cada um.

Parâmetros na URL usam conversores de tipo:

```python
urlpatterns = [
    path("livros/", views.listar_livros, name="listar_livros"),
    path("livros/<int:livro_id>/", views.detalhe_livro, name="detalhe_livro"),
]
```

```python
def detalhe_livro(request, livro_id):
    return HttpResponse(f"Detalhes do livro {livro_id}")
```

Se alguém acessar `livros/abc/` (não numérico), o Django já devolve 404 automaticamente,
porque `abc` não bate com o conversor `int`.

## 1.8 Templates (visão geral, breve)

*Templates* são a forma "tradicional" do Django de gerar HTML no servidor. Como este
material tem foco em **construir APIs** (capítulos 3 em diante), templates aparecem aqui
só para você reconhecer o padrão quando encontrar em outros projetos:

```
catalogo/templates/catalogo/detalhe_livro.html
```

```html
<h1>{{ livro.titulo }}</h1>
{% if livro.quantidade_disponivel > 0 %}
  <p>Disponível</p>
{% else %}
  <p>Indisponível</p>
{% endif %}
```

```python
from django.shortcuts import render

def detalhe_livro(request, livro_id):
    livro = {"titulo": "Duna", "quantidade_disponivel": 3}
    return render(request, "catalogo/detalhe_livro.html", {"livro": livro})
```

Na prática, quando o objetivo é servir dados para um front-end separado (React, app
mobile) ou para outros sistemas, você não usa templates — você usa **serializers** e
devolve JSON. É para lá que este material caminha a partir do capítulo 3.

## 1.9 Debug: duas ferramentas essenciais

**1) Debugger integrado ao editor.** No VS Code, crie um `launch.json` de tipo "Django",
coloque um breakpoint clicando à esquerda da linha e rode em modo debug. Você consegue:

- `Step Over` (executar linha atual e ir para a próxima, sem entrar em funções chamadas);
- `Step Into` (entrar dentro da função chamada na linha atual);
- `Step Out` (sair da função atual e voltar para quem a chamou);
- inspecionar variáveis locais em tempo real.

**2) Django Debug Toolbar.** Um pacote que, quando ativo, mostra uma barra lateral com:
tempo de processamento, configurações ativas, e — o mais importante no dia a dia — a **aba
SQL**, que lista todas as queries disparadas para renderizar aquela página. Esse hábito de
olhar quantas queries uma view dispara é o que evita, lá na frente, o clássico
**problema N+1** (seção 2.6).

```bash
pip install django-debug-toolbar
```

```python
# settings.py
INSTALLED_APPS += ["debug_toolbar"]
MIDDLEWARE += ["debug_toolbar.middleware.DebugToolbarMiddleware"]
INTERNAL_IPS = ["127.0.0.1"]
```

```python
# urls.py
from django.conf import settings

if settings.DEBUG:
    import debug_toolbar
    urlpatterns += [path("__debug__/", include(debug_toolbar.urls))]
```

---

## 1.10 Modelagem de dados: pensando antes de codar

Antes de escrever qualquer `models.py`, vale desenhar (no papel mesmo) as entidades do
domínio e como elas se relacionam. Para o sistema de biblioteca:

| Entidade | Atributos principais | Relacionamentos |
|---|---|---|
| `Autor` | nome, biografia, data de nascimento | 1 autor → N livros |
| `Categoria` | nome | N categorias ↔ N livros |
| `Livro` | título, isbn, ano, preço, estoque | pertence a 1 autor, tem N categorias |
| `Leitor` | telefone, data de nascimento | 1:1 com o usuário do sistema |
| `Emprestimo` | data de retirada, data prevista, data de devolução, status | pertence a 1 leitor e a 1 livro |

Tipos de relacionamento que você vai usar o tempo todo:

- **Um-para-um (1:1)**: cada instância de A se relaciona com no máximo uma de B (ex.:
  `Leitor` ↔ `Usuario`).
- **Um-para-muitos (1:N)**: uma instância de A pode se relacionar com várias de B, mas cada
  B pertence a um único A (ex.: `Autor` → `Livro`).
- **Muitos-para-muitos (N:N)**: cada A pode se relacionar com várias B e vice-versa (ex.:
  `Livro` ↔ `Categoria`). Internamente o banco resolve isso com uma **tabela associativa**.
  Quando essa relação em si precisa carregar dados próprios (não só "A está ligado a B",
  mas "A está ligado a B *com quantidade X*"), você promove a tabela associativa a um
  modelo de verdade — é exatamente o papel de `Emprestimo` mais adiante, e também de um
  eventual `ItemEmprestimo` se um empréstimo puder conter vários livros de uma vez.

> Não modele campos "porque pode ser útil" — modele em função de um requisito real. Um
> exercício simples e eficaz: para cada campo que você for adicionar, pergunte "que
> pergunta de negócio esse campo responde?". Se não houver resposta clara, ele provavelmente
> não deveria existir ainda.

## 1.11 Definindo os models

```python
# catalogo/models.py
from django.db import models


class Autor(models.Model):
    nome = models.CharField(max_length=255)
    biografia = models.TextField(blank=True)
    data_nascimento = models.DateField(null=True, blank=True)

    class Meta:
        ordering = ["nome"]

    def __str__(self):
        return self.nome


class Categoria(models.Model):
    nome = models.CharField(max_length=100, unique=True)

    class Meta:
        verbose_name_plural = "categorias"
        ordering = ["nome"]

    def __str__(self):
        return self.nome


class Livro(models.Model):
    titulo = models.CharField(max_length=255)
    isbn = models.CharField(max_length=13, unique=True)
    ano_publicacao = models.PositiveIntegerField()
    preco = models.DecimalField(max_digits=8, decimal_places=2)
    quantidade_total = models.PositiveIntegerField(default=1)
    quantidade_disponivel = models.PositiveIntegerField(default=1)
    sinopse = models.TextField(blank=True)
    capa_url = models.URLField(blank=True)
    criado_em = models.DateTimeField(auto_now_add=True)
    atualizado_em = models.DateTimeField(auto_now=True)

    autor = models.ForeignKey(
        Autor,
        on_delete=models.PROTECT,
        related_name="livros",
    )
    categorias = models.ManyToManyField(
        Categoria,
        related_name="livros",
        blank=True,
    )

    class Meta:
        ordering = ["titulo"] # Serve para ordenar no painel administrativo

    def __str__(self):
        return self.titulo
```

### Tipos de campo mais usados

| Campo | Uso típico |
|---|---|
| `CharField(max_length=...)` | strings curtas/médias — `max_length` é obrigatório |
| `TextField()` | textos longos, sem limite prático de tamanho |
| `IntegerField` / `PositiveIntegerField` | números inteiros |
| `DecimalField(max_digits, decimal_places)` | **sempre** para valores monetários (nunca `FloatField`, que tem erro de arredondamento) |
| `BooleanField(default=...)` | verdadeiro/falso |
| `DateField` / `DateTimeField` | data / data e hora |
| `EmailField`, `URLField`, `SlugField` | variações de `CharField` com validação/uso específico |
| `ForeignKey(Modelo, on_delete=...)` | relação 1:N |
| `OneToOneField(Modelo, on_delete=...)` | relação 1:1 |
| `ManyToManyField(Modelo)` | relação N:N |

### Opções de campo comuns

| Opção | Efeito |
|---|---|
| `null=True` | permite `NULL` **no banco de dados** |
| `blank=True` | permite valor vazio **na validação de formulários/admin/serializers** |
| `default=...` | valor padrão se nada for informado |
| `unique=True` | cria constraint de unicidade |
| `db_index=True` | cria índice na coluna |
| `choices=[...]` | restringe os valores possíveis |
| `auto_now_add=True` | preenche automaticamente na criação (não editável depois) |
| `auto_now=True` | atualiza automaticamente a cada `save()` |

> **`null` vs `blank` é a confusão nº 1 de quem está começando.** `null` é sobre o banco de
> dados; `blank` é sobre validação de entrada. Um campo de texto (`CharField`) quase nunca
> precisa de `null=True`, porque "vazio" já é representado por uma string vazia `""` — usar
> `null=True` ali só cria duas formas diferentes de representar "sem valor" (`""` e
> `NULL`), o que é uma fonte clássica de bugs.

### `choices`: restringindo valores possíveis

```python
class Emprestimo(models.Model):
    STATUS_ATIVO = "A"
    STATUS_DEVOLVIDO = "D"
    STATUS_ATRASADO = "R"
    STATUS_CHOICES = [
        (STATUS_ATIVO, "Ativo"),
        (STATUS_DEVOLVIDO, "Devolvido"),
        (STATUS_ATRASADO, "Em atraso"),
    ]

    status = models.CharField(
        max_length=1,
        choices=STATUS_CHOICES,
        default=STATUS_ATIVO,
    )
```

Guardar as opções como constantes (`STATUS_ATIVO = "A"`) em vez de espalhar a letra `"A"`
pelo código evita erro de digitação e centraliza a fonte da verdade — se um dia o código
mudar de `"A"` para `"ATIVO"`, você só ajusta em um lugar.

## 1.12 Relacionamentos em detalhe

### `ForeignKey` (1:N) e a opção `on_delete`

```python
autor = models.ForeignKey(Autor, on_delete=models.PROTECT, related_name="livros")
```

`on_delete` define o que acontece com o registro filho quando o pai é apagado:

| Opção | Comportamento |
|---|---|
| `CASCADE` | apaga os filhos junto (ex.: apagar um `Emprestimo` ao apagar o `Leitor`) |
| `PROTECT` | **impede** a exclusão do pai se existir algum filho (ex.: não deixar apagar um `Autor` que tem livros) |
| `SET_NULL` | zera o campo (exige `null=True`) |
| `SET_DEFAULT` | volta para um valor padrão |
| `RESTRICT` | parecido com `PROTECT`, mas com nuances em cascatas complexas |

`related_name` define como você acessa a relação **inversa** (do lado "1"). Sem
`related_name`, o Django cria automaticamente `<modelo>_set` (ex.: `autor.livro_set.all()`).
Com `related_name="livros"`, fica `autor.livros.all()` — muito mais legível.

### `OneToOneField` (1:1) — o padrão de "perfil"

Um dos usos mais comuns de 1:1 é estender o usuário do sistema com dados adicionais sem
modificar a tabela de autenticação (mais detalhes no capítulo 6):

```python
# emprestimos/models.py
from django.conf import settings
from django.db import models


class Leitor(models.Model):
    usuario = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        primary_key=True,     # a PK do Leitor É a PK do Usuario
        related_name="leitor",
    )
    telefone = models.CharField(max_length=20, blank=True)
    data_nascimento = models.DateField(null=True, blank=True)

    def __str__(self):
        return self.usuario.get_full_name() or self.usuario.username
```

Repare em dois detalhes:

- `settings.AUTH_USER_MODEL` em vez de importar o modelo de usuário diretamente — isso
  mantém o app `emprestimos` reutilizável em qualquer projeto, independente de qual modelo
  de usuário aquele projeto usa (voltamos a isso no capítulo 6).
- `primary_key=True` no `OneToOneField` faz a PK de `Leitor` **ser** a PK do `Usuario`
  associado — sem isso, o Django cria um `id` próprio e a relação vira, na prática, um
  1:N disfarçado (nada impediria múltiplos `Leitor` para o mesmo `Usuario`).

### `ManyToManyField` (N:N)

```python
categorias = models.ManyToManyField(Categoria, related_name="livros", blank=True)
```

Por trás dos panos, o Django cria uma tabela intermediária (`catalogo_livro_categorias`)
com duas `ForeignKey`. Você não precisa (nem deve) criar essa tabela na mão — a não ser
que a relação em si precise guardar dados extras (nesse caso, você declara explicitamente
uma tabela intermediária com `through=`).

### Relações genéricas (avançado)

Quando você quer relacionar um modelo com **qualquer outro modelo do sistema**, sem
acoplar o app a um modelo específico (o mesmo princípio de "app reutilizável" citado na
seção 1.5), usa-se o *ContentType framework*:

```python
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
from django.db import models


class Avaliacao(models.Model):
    nota = models.PositiveSmallIntegerField()
    comentario = models.TextField(blank=True)

    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    conteudo = GenericForeignKey("content_type", "object_id")
```

Com isso, `Avaliacao` pode apontar tanto para um `Livro` quanto, no futuro, para um
`Autor` ou qualquer outro modelo — sem o app de avaliações precisar `import` nenhum
modelo do app de catálogo. É o mesmo raciocínio usado em sistemas de "tags" genéricas.

## 1.13 Metadados de modelo (`class Meta`)

```python
class Livro(models.Model):
    ...
    class Meta:
        ordering = ["titulo"]
        db_table = "livros"                       # nome customizado da tabela
        indexes = [models.Index(fields=["isbn"])]  # índice explícito
        constraints = [
            models.UniqueConstraint(fields=["isbn"], name="isbn_unico"),
        ]
        permissions = [
            ("pode_dar_baixa_estoque", "Pode dar baixa manual no estoque"),
        ]
```

`db_table` deveria ser evitado na maioria dos casos: seguir a convenção padrão do Django
(`app_modelo`) mantém o projeto previsível — só vale sobrescrever quando há uma razão de
peso (por exemplo, integrar com um banco legado).

## 1.14 Migrations: versionando o esquema do banco

Migrations são o "Git do seu banco de dados": cada mudança em `models.py` vira um arquivo
que descreve, em Python, como transformar o esquema do banco de um estado para o próximo.

```bash
# 1. Gera o arquivo de migration a partir das mudanças em models.py
python manage.py makemigrations

# 2. Aplica as migrations pendentes no banco
python manage.py migrate

# 3. Mostra o SQL que uma migration específica vai executar (sem rodar)
python manage.py sqlmigrate catalogo 0001

# 4. Reverte para uma migration anterior
python manage.py migrate catalogo 0002
```

Boas práticas com migrations:

- **Uma mudança lógica por migration.** Não misture "renomear campo" com "criar índice
  novo" na mesma migration — fica difícil de reverter seletivamente depois.
- **Rode `makemigrations` a cada mudança de modelo**, mesmo que pequena. Deixar acumular
  gera migrations gigantes e confusas.
- **Dê nomes descritivos**: `python manage.py makemigrations catalogo -n adiciona_isbn`.
- Ao adicionar um campo obrigatório em uma tabela que já tem dados, o Django vai pedir um
  valor padrão para preencher as linhas existentes — decida conscientemente entre um
  "valor único agora" (só usado naquela migration) e um `default=` fixo no próprio modelo.
- Para operações que o ORM não modela diretamente (criar uma *stored procedure*, popular
  dados), é possível escrever uma migration vazia e usar `migrations.RunSQL`:

```python
from django.db import migrations

def popular_categorias(apps, schema_editor):
    Categoria = apps.get_model("catalogo", "Categoria")
    Categoria.objects.bulk_create([
        Categoria(nome="Ficção"),
        Categoria(nome="Programação"),
        Categoria(nome="História"),
    ])

class Migration(migrations.Migration):
    dependencies = [("catalogo", "0001_initial")]
    operations = [
        migrations.RunPython(popular_categorias, reverse_code=migrations.RunPython.noop),
    ]
```

Note o uso de `apps.get_model(...)` em vez de `import` direto do modelo — dentro de uma
migration, você deve sempre pegar uma versão "congelada" do modelo daquele ponto no
histórico, e não a versão atual de `models.py` (que pode ter mudado desde então).

---

## Checklist de conceitos deste capítulo

- [ ] Sei explicar a diferença entre front-end e back-end, e o que é uma URL
- [ ] Consigo criar um projeto e um app Django do zero
- [ ] Entendo por que "view" no Django é o *request handler*, não o HTML
- [ ] Sei registrar rotas com `path()` e `include()`
- [ ] Sei quando usar `null=True` vs `blank=True`
- [ ] Entendo `on_delete` (`CASCADE`, `PROTECT`, `SET_NULL`) e sei quando usar cada um
- [ ] Sei modelar 1:N, 1:1 e N:N e sei o que é `related_name`
- [ ] Sei o ciclo `makemigrations` → `migrate`

## Exercícios propostos

1. Crie os apps `catalogo` e `emprestimos`, com os modelos `Autor`, `Categoria`, `Livro`,
   `Leitor` e `Emprestimo` (pode usar os exemplos deste capítulo como base, mas tente
   escrever de memória primeiro).
2. Adicione ao modelo `Livro` um campo `editora` (um novo modelo `Editora` com relação 1:N)
   e gere a migration correspondente.
3. Crie uma migration de dados (`RunPython`) que popule 5 categorias padrão.
4. Rode `sqlmigrate` na sua primeira migration e leia o SQL gerado — identifique os tipos
   de coluna que o Django escolheu para cada campo do seu modelo.
