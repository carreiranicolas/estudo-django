# 6. Autenticação, Permissões e Signals

> Último capítulo. Aqui fechamos o ciclo: quem pode fazer o quê na API construída no
> capítulo 5, como identificar o usuário por trás de cada requisição, e como conectar
> partes do sistema sem acoplá-las diretamente.

## 6.1 Como o Django sabe quem é o usuário

O Django já vem com um sistema de autenticação completo no app `django.contrib.auth`:
modelo `User`, `Group`, `Permission`, hashing de senha, tudo pronto.

A peça que liga uma requisição HTTP a um usuário autenticado é um **middleware** —
`AuthenticationMiddleware`, listado em `settings.py`. A cada requisição, ele lê as
credenciais (sessão, token, o que estiver configurado) e popula `request.user`:

```python
# settings.py
MIDDLEWARE = [
    ...
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    ...
]
```

```python
def alguma_view(request):
    if request.user.is_authenticated:
        print(request.user.id)
    else:
        # request.user é uma instância de AnonymousUser, não None
        print("visitante anônimo")
```

`request.user` **nunca** é `None` — se ninguém estiver logado, ele é uma instância de
`AnonymousUser`, uma classe que responde `is_authenticated=False` e não tem `id` de banco.
Esse detalhe evita ter que checar `if request.user is not None` toda hora.

## 6.2 Customizando o usuário: duas estratégias

Existem duas formas de adicionar dados de usuário além do que o Django já oferece
(username, email, senha, nome). A escolha certa depende de uma pergunta: **esse dado afeta
o processo de autenticação em si, ou é só informação de perfil, específica de um domínio?**

### Estratégia 1 — Estender o modelo (`AbstractUser`)

Use quando o dado é sobre autenticação/identidade central do usuário (ex.: e-mail único,
exigir CPF para login). Isso **estende a própria tabela de usuários**.

```python
# core/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models


class Usuario(AbstractUser):
    email = models.EmailField(unique=True)
```

```python
# settings.py
AUTH_USER_MODEL = "core.Usuario"
```

> **Atenção crítica:** essa troca só é segura **no início do projeto**, antes de rodar a
> primeira `migrate`. Trocar `AUTH_USER_MODEL` no meio de um projeto com dados existentes
> exige recriar o banco do zero, porque toda migration anterior já ficou "amarrada" ao
> modelo antigo de usuário. Por isso a recomendação padrão é: **todo projeto Django deveria
> começar com um `Usuario` customizado (mesmo vazio, com `pass`)**, só para manter essa
> porta aberta caso um dia seja necessário.

### Estratégia 2 — Perfil (composição via `OneToOneField`)

Use para qualquer dado que seja **específico de um domínio do seu sistema**, e não do
processo de autenticação em si — telefone, endereço, preferências, e no nosso caso,
"quantos livros esse leitor pode ter emprestados ao mesmo tempo".

Já construímos isso no capítulo 1 (seção 1.12) com o modelo `Leitor`:

```python
# core/models.py
from django.conf import settings
from django.db import models


class Leitor(models.Model):
    usuario = models.OneToOneField(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE,
        primary_key=True, related_name="leitor",
    )
    telefone = models.CharField(max_length=20, blank=True)
    limite_emprestimos = models.PositiveSmallIntegerField(default=3)
```

A vantagem dessa abordagem é que cada app pode ter seu **próprio** conceito de "perfil": em
um sistema de RH, o perfil seria `Funcionario`; em um de vendas, `Cliente`. Nenhum desses
apps precisa saber como o outro modela usuário — todos só dependem de
`settings.AUTH_USER_MODEL`, nunca de um modelo concreto importado diretamente. Essa é a
mesma disciplina de "app reutilizável" da seção 1.5, aplicada a autenticação.

> **Regra prática:** na dúvida, comece pela estratégia 2 (perfil). Ela resolve a imensa
> maioria dos casos sem o risco de precisar recriar o banco depois.

## 6.3 Grupos e permissões

Toda vez que você cria um modelo e roda `migrate`, o Django cria automaticamente 4
permissões para ele: `add_<modelo>`, `change_<modelo>`, `delete_<modelo>`,
`view_<modelo>`. Você pode:

- atribuir permissões **diretamente** a um usuário (evite — vira difícil de auditar);
- criar um **grupo** (ex.: "Bibliotecários"), atribuir permissões a ele, e colocar usuários
  dentro do grupo (recomendado — centraliza a fonte da verdade).

Isso é feito pelo painel admin (`/admin/auth/group/`), sem precisar de código.

### Permissões customizadas (além das 4 automáticas)

Já usamos isso no capítulo 5 (`Emprestimo.Meta.permissions`):

```python
class Emprestimo(models.Model):
    ...
    class Meta:
        permissions = [
            ("cancelar_emprestimo", "Pode cancelar um empréstimo"),
        ]
```

Depois de `makemigrations` + `migrate`, essa permissão passa a existir no banco e pode ser
atribuída a um grupo/usuário, e checada em código com:

```python
request.user.has_perm("emprestimos.cancelar_emprestimo")
```

## 6.4 Autenticação em uma API: por que token, e não sessão

Para sites tradicionais (com templates), o Django usa **sessões**: o servidor grava um
consumidas por apps mobile, front-ends desacoplados ou outros serviços — o padrão de fato
para APIs REST é **autenticação por token**, e dentro disso, o formato mais usado hoje é o
**JSON Web Token (JWT)**.

### Anatomia de um JWT

Um JWT é uma string com três partes separadas por `.`:

```
cabecalho.payload.assinatura
```

- **Cabeçalho**: metadados (algoritmo usado).
- **Payload**: dados do token — tipicamente o ID do usuário, o tipo do token (`access` ou
  `refresh`) e a data de expiração. **Não é criptografado**, só codificado em Base64 —
  qualquer pessoa consegue *ler* o conteúdo de um JWT, só não consegue *forjar* um válido.
- **Assinatura**: gerada a partir do cabeçalho + payload usando uma chave secreta que só o
  servidor conhece. Se alguém alterar o payload (por exemplo, trocar o ID do usuário) sem
  ter a chave secreta, a assinatura deixa de bater, e o servidor rejeita o token.

Essa assinatura é o que permite ao servidor validar um token **sem precisar consultar o
banco de dados a cada requisição** (diferente de um token de sessão tradicional, que exige
uma consulta para conferir se o token ainda é válido) — um ganho real de performance em
APIs com muito tráfego.

### O ciclo de vida de dois tokens

- **Access token**: de vida curta (minutos), enviado em todo request para endpoints
  protegidos.
- **Refresh token**: de vida mais longa (dias), usado só para obter um novo access token
  quando o atual expira — sem forçar o usuário a digitar a senha de novo.

```
1. Login (usuário + senha) -> servidor devolve {access, refresh}
2. Cliente guarda os dois (localStorage, keychain do app, etc.)
3. Toda requisição protegida: header Authorization: Bearer <access>
4. Se o servidor responder 401 (access expirado) -> cliente chama /refresh/ com o refresh
5. Servidor devolve um novo access -> cliente repete a requisição original
```

"Logout", nesse modelo, é simplesmente **apagar os tokens do lado do cliente** — não existe
endpoint de logout no servidor, porque não há estado de sessão guardado lá (a não ser que
você implemente uma lista de tokens revogados, um refinamento fora do escopo deste
material).

## 6.5 Djoser + Simple JWT: montando os endpoints de autenticação

Reimplementar registro, login, troca de senha, etc. na mão é repetitivo — o pacote
**Djoser** fornece essas views prontas, delegando a geração/validação dos tokens em si a um
back-end de autenticação (aqui, `djangorestframework-simplejwt`).

```bash
pip install djoser djangorestframework-simplejwt
```

```python
# settings.py
INSTALLED_APPS += ["djoser"]

REST_FRAMEWORK = {
    ...
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
}

from datetime import timedelta

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=1),
}
```

```python
# biblioteca_config/urls.py
urlpatterns = [
    ...
    path("api/auth/", include("djoser.urls")),
    path("api/auth/", include("djoser.urls.jwt")),
]
```

Isso já disponibiliza, entre outros:

```
POST /api/auth/users/            -> registrar novo usuário
GET  /api/auth/users/me/         -> dados do usuário autenticado
POST /api/auth/jwt/create/       -> login (recebe username+password, devolve access+refresh)
POST /api/auth/jwt/refresh/      -> troca refresh por um novo access
```

Exemplo de uso do endpoint de login:

```
POST /api/auth/jwt/create/
{"username": "maria", "password": "senha-segura-123"}

-> 200 OK
{"access": "eyJhbGciOi...", "refresh": "eyJhbGciOi..."}
```

E de uma requisição autenticada:

```
GET /api/emprestimos/
Authorization: Bearer eyJhbGciOi...
```

### Customizando o serializer de registro do Djoser

Por padrão, o Djoser expõe só `username`, `email` e `password` no registro. Para incluir
campos adicionais (por exemplo, nome completo), sobrescrevemos o serializer:

```python
# core/serializers.py
from djoser.serializers import UserCreateSerializer as BaseUserCreateSerializer


class UserCreateSerializer(BaseUserCreateSerializer):
    class Meta(BaseUserCreateSerializer.Meta):
        fields = ["id", "username", "email", "password", "first_name", "last_name"]
```

```python
# settings.py
DJOSER = {
    "SERIALIZERS": {
        "user_create": "core.serializers.UserCreateSerializer",
        "current_user": "core.serializers.UserCreateSerializer",
    },
}
```

Repare que herdamos de `Meta` do serializer base, em vez de reescrever tudo do zero — assim,
se uma versão futura do Djoser mudar algo na implementação padrão, seu serializer continua
compatível, e você só está sobrescrevendo os `fields`.

> **O que *não* colocar aqui:** dados de perfil (como o `telefone` de `Leitor`) não deveriam
> ser capturados nesse mesmo serializer de registro. Misturar "criar usuário" com "criar
> perfil de domínio" no mesmo serializer viola a mesma ideia de responsabilidade única que
> discutimos na seção 5.9 — a solução correta é o cliente fazer duas chamadas (criar
> usuário, depois `PATCH` no próprio perfil), ou — melhor ainda — resolver isso
> automaticamente com um **signal**, que é o assunto da seção 6.7.

## 6.6 Permissões no DRF

Permissões controlam **quem pode acessar cada endpoint**, sendo verificadas antes de a
view processar a requisição.

### Classes prontas

| Classe | Efeito |
|---|---|
| `AllowAny` | qualquer um, autenticado ou não (padrão do DRF) |
| `IsAuthenticated` | só usuários logados |
| `IsAdminUser` | só usuários com `is_staff=True` |
| `IsAuthenticatedOrReadOnly` | leitura livre, escrita só para autenticados |
| `DjangoModelPermissions` | usa as permissões automáticas do modelo (`add_`, `change_`, etc.) |

Aplicando globalmente (todos os endpoints exigem login por padrão, a não ser que a view
diga o contrário):

```python
# settings.py
REST_FRAMEWORK = {
    ...
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
}
```

Ou por view/ViewSet:

```python
class LivroViewSet(ModelViewSet):
    ...

    def get_permissions(self):
        if self.request.method in ("GET", "HEAD", "OPTIONS"):
            return [AllowAny()]
        return [IsAdminUser()]
```

Repare que `get_permissions()` retorna uma **lista de instâncias** (`AllowAny()`, com
parênteses), diferente do atributo `permission_classes`, que é uma lista de **classes**
(sem parênteses) — um erro comum é esquecer os parênteses ao usar o método.

### Permissão customizada

Quando nenhuma classe pronta expressa a regra que você precisa, você escreve a sua,
herdando de `BasePermission`:

```python
# catalogo/permissions.py
from rest_framework.permissions import SAFE_METHODS, BasePermission


class EhAdminOuSomenteLeitura(BasePermission):
    def has_permission(self, request, view):
        if request.method in SAFE_METHODS:   # GET, HEAD, OPTIONS
            return True
        return bool(request.user and request.user.is_staff)
```

```python
class LivroViewSet(ModelViewSet):
    permission_classes = [EhAdminOuSomenteLeitura]
```

Para checar uma **permissão de objeto específico** (não só "pode acessar essa view",
mas "pode acessar *este* registro"), há também `has_object_permission`:

```python
class SoDonoOuAdmin(BasePermission):
    def has_object_permission(self, request, view, emprestimo):
        return request.user.is_staff or emprestimo.leitor.usuario == request.user
```

`has_object_permission` só é chamado quando a view usa `self.get_object()` (as ações de
detalhe: `retrieve`, `update`, `destroy`, e ações customizadas de `detail=True`) — ele não
entra em ação sozinho em `list`, por isso o filtro em `get_queryset()` (como fizemos no
capítulo 5, seção 5.7) continua sendo necessário para restringir listagens.

### Usando uma permissão customizada baseada em `has_perm`

Combinando com a permissão customizada de `Meta.permissions` (seção 6.3):

```python
from rest_framework.permissions import BasePermission


class PodeCancelarEmprestimo(BasePermission):
    def has_permission(self, request, view):
        return request.user.has_perm("emprestimos.cancelar_emprestimo")
```

```python
class EmprestimoViewSet(...):
    ...

    @action(detail=True, methods=["post"], permission_classes=[PodeCancelarEmprestimo])
    def cancelar(self, request, pk=None):
        ...
```

`permission_classes` também pode ser passado direto no decorador `@action`, sobrescrevendo
a permissão padrão da classe só para aquela ação específica — muito útil quando uma ação
tem uma regra de acesso mais restrita que o restante do ViewSet.

## 6.7 Signals: desacoplando apps

Um **signal** é uma notificação disparada em algum ponto do ciclo de vida de um modelo,
que qualquer parte do sistema pode "escutar" sem que o código que dispara o signal precise
saber quem está escutando. É o mecanismo do Django para dois módulos colaborarem **sem
depender um do outro diretamente**.

### Signals nativos do Django

| Signal | Disparado |
|---|---|
| `pre_save` / `post_save` | antes / depois de `.save()` |
| `pre_delete` / `post_delete` | antes / depois de `.delete()` |
| `m2m_changed` | quando um relacionamento `ManyToMany` é alterado |

### Caso de uso: criar `Leitor` automaticamente ao registrar um usuário

Voltando ao problema deixado em aberto na seção 6.5 (não misturar criação de usuário com
criação de perfil): a solução limpa é o app `core` (dono de `Usuario`) **não saber nada**
sobre `Leitor`, e o app `emprestimos` (dono de `Leitor`) escutar o evento "usuário criado":

```python
# emprestimos/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.conf import settings
from .models import Leitor


@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def criar_leitor_para_novo_usuario(sender, instance, created, **kwargs):
    if created:
        Leitor.objects.create(usuario=instance)
```

- `sender=settings.AUTH_USER_MODEL` — de novo, referenciamos pela *configuração*, não pelo
  modelo concreto, mantendo `emprestimos` livre de um `import` direto do modelo de usuário.
- `created` é um booleano que o `post_save` sempre envia: `True` só na primeira gravação
  (`INSERT`), `False` em atualizações subsequentes (`UPDATE`) — sem essa checagem, o
  handler tentaria criar um `Leitor` duplicado a cada vez que o usuário atualizasse o
  próprio cadastro.

Handlers de signal só são registrados se o módulo `signals.py` for de fato importado em
algum momento da inicialização — o lugar certo para isso é o método `ready()` da
configuração do app:

```python
# emprestimos/apps.py
from django.apps import AppConfig


class EmprestimosConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "emprestimos"

    def ready(self):
        import emprestimos.signals  # noqa: F401
```

### Signals customizados: eventos do seu próprio domínio

Além de escutar eventos do Django, você pode **criar** e **disparar** os seus próprios
eventos de negócio. Isso é útil quando uma ação (como "confirmar um empréstimo", do
capítulo 5) deveria acionar comportamentos em outros módulos — notificação por e-mail,
atualização de estatísticas, integração externa — sem que o módulo de empréstimos precise
conhecer nenhum desses módulos.

```python
# emprestimos/signals.py (continuação)
import django.dispatch

emprestimo_confirmado = django.dispatch.Signal()
```

Disparando o evento no ponto em que a operação de negócio realmente acontece (dentro do
`save()` de `ConfirmarEmprestimoSerializer`, do capítulo 5):

```python
from emprestimos.signals import emprestimo_confirmado

class ConfirmarEmprestimoSerializer(serializers.Serializer):
    ...
    def save(self, **kwargs):
        with transaction.atomic():
            ...
            emprestimo_confirmado.send_robust(sender=self.__class__, emprestimo=emprestimo)

        self.instance = emprestimo
        return self.instance
```

Um módulo totalmente independente (ex.: `notificacoes`) escuta esse evento sem que
`emprestimos` saiba que `notificacoes` sequer existe:

```python
# notificacoes/signals.py
from django.dispatch import receiver
from emprestimos.signals import emprestimo_confirmado


@receiver(emprestimo_confirmado)
def enviar_email_confirmacao(sender, emprestimo, **kwargs):
    print(f"[e-mail simulado] Empréstimo #{emprestimo.id} confirmado.")
    # aqui entraria a integração real de e-mail/push
```

`send_robust()` (em vez de `send()`) garante que, se um dos handlers lançar uma exceção
(ex.: falha ao enviar e-mail), os **outros** handlers registrados para o mesmo signal ainda
são executados — uma falha de notificação não deveria derrubar o fluxo principal de
confirmar o empréstimo.

> **Quando usar signals, e quando não usar:** signals são ótimos para desacoplar
> comportamentos verdadeiramente **secundários** ao fluxo principal (notificar, logar,
> atualizar um cache). Não são um bom lugar para lógica **essencial** ao resultado da
> operação (como decrementar o estoque do livro) — essa lógica deveria continuar explícita
> e visível no mesmo lugar onde a operação principal acontece (como fizemos no capítulo 5),
> porque handlers de signal são "invisíveis" a quem lê o código do chamador, o que torna
> mais difícil entender o fluxo completo de uma operação crítica só olhando um arquivo.

---

## Checklist de conceitos deste capítulo

- [ ] Entendo o papel do `AuthenticationMiddleware` e o que é `request.user`
- [ ] Sei escolher entre estender o usuário (`AbstractUser`) e usar perfil (`OneToOneField`)
- [ ] Sei criar grupos/permissões e permissões customizadas via `Meta.permissions`
- [ ] Entendo a estrutura de um JWT (cabeçalho, payload, assinatura) e o papel de access x
  refresh token
- [ ] Sei configurar Djoser + Simple JWT e customizar o serializer de registro
- [ ] Sei aplicar permissões prontas do DRF e escrever uma `BasePermission` customizada
- [ ] Sei a diferença entre `has_permission` (a view) e `has_object_permission` (um registro)
- [ ] Sei registrar um handler de signal nativo (`post_save`) e criar/disparar um signal
  customizado, e sei justificar quando **não** usar signals

## Exercícios propostos

1. Implemente o signal que cria um `Leitor` automaticamente ao registrar um novo usuário
   via Djoser, e confirme (criando um usuário pela API) que o perfil aparece no admin.
2. Crie uma permissão customizada `EhDonoDoEmprestimoOuAdmin` usando
   `has_object_permission`, e aplique-a nas ações de detalhe de `EmprestimoViewSet`.
3. Adicione a permissão customizada `cancelar_emprestimo` a um grupo "Bibliotecários" pelo
   admin, crie um usuário nesse grupo e confirme que só ele consegue acessar a ação
   `cancelar` implementada no capítulo 5.
4. Crie um signal customizado `emprestimo_devolvido`, disparado dentro da ação `devolver`
   (capítulo 5), e um handler em um app separado que apenas imprime uma mensagem — sem que
   o app `emprestimos` precise importar nada desse novo app.

---

## Fechando o material

Com os seis capítulos, você tem o ciclo completo de uma API Django "de produção": modelagem
→ ORM/admin → serialização → arquitetura de views/rotas → um estudo de caso real com regra
de negócio e transações → autenticação e permissões → desacoplamento via signals. A partir
daqui, os próximos passos naturais (alinhados com o que costuma vir depois de dominar
Django/DRF) são: escrever testes automatizados para cada endpoint construído no capítulo 5,
adicionar cache em consultas pesadas, e — quando fizer sentido comparar abordagens — revisar
esses mesmos conceitos (serialização, validação, rotas, dependências) no vocabulário do
FastAPI, que resolve os mesmos problemas com uma filosofia mais explícita e tipada.
