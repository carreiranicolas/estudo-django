# 5. Estudo de Caso: API de Reserva e Empréstimo de Livros

Este capítulo é o equivalente, no domínio de biblioteca, ao clássico exercício de "API de
carrinho de compras" — o cenário mais rico para praticar tudo dos capítulos 3 e 4 ao mesmo
tempo: recursos aninhados, serializers diferentes por ação, regras de negócio, transações e
permissões trabalhando juntas.

## 5.1 O fluxo de negócio

```
1. Um visitante (mesmo sem estar logado) cria uma "Reserva" (carrinho temporário)
2. Ele vai adicionando livros à reserva, com quantidade
3. Quando decide confirmar, ele precisa estar autenticado -> envia a reserva para
   virar um "Empréstimo" de verdade
4. O sistema valida disponibilidade, decrementa o estoque, registra o empréstimo
   e apaga a reserva temporária
5. Mais tarde, o livro é devolvido -> o estoque volta, o status do empréstimo muda
```

Esse desenho — um recurso "temporário e anônimo" (a `Reserva`) que se transforma em um
recurso "definitivo e vinculado a um usuário" (o `Emprestimo`) — é exatamente o mesmo
padrão usado por qualquer carrinho de compras que vira pedido. Tudo que você aprender aqui
se aplica sem alterações a esse cenário.

## 5.2 Desenhando os endpoints antes de codar

| Operação | Método + URL | Corpo enviado | Retorno |
|---|---|---|---|
| Criar reserva | `POST /api/reservas/` | *(vazio)* | reserva com `id` (UUID) |
| Obter reserva | `GET /api/reservas/{id}/` | — | reserva com itens e total |
| Apagar reserva | `DELETE /api/reservas/{id}/` | — | `204` |
| Listar itens | `GET /api/reservas/{id}/itens/` | — | lista de itens |
| Adicionar item | `POST /api/reservas/{id}/itens/` | `{"livro_id": 3, "quantidade": 2}` | item criado |
| Atualizar item | `PATCH /api/reservas/{id}/itens/{item_id}/` | `{"quantidade": 1}` | item atualizado |
| Remover item | `DELETE /api/reservas/{id}/itens/{item_id}/` | — | `204` |
| Confirmar empréstimo | `POST /api/emprestimos/` | `{"reserva_id": "..."}` | empréstimo criado |
| Listar meus empréstimos | `GET /api/emprestimos/` | — | próprios (ou todos, se admin) |
| Devolver | `POST /api/emprestimos/{id}/devolver/` | *(vazio)* | empréstimo atualizado |

Planejar a tabela acima **antes** de tocar em código é o que evita ficar redesenhando o
modelo de dados no meio do caminho — é basicamente a mesma disciplina de "modelar antes de
codar" da seção 1.10, só que aplicada à API em vez de ao banco.

## 5.3 Ajustando o modelo de dados

### Por que UUID como chave primária aqui

Uma `Reserva` é criada **sem autenticação**. Se a chave primária fosse um inteiro
autoincrementado (`1`, `2`, `3`...), seria trivial para qualquer pessoa "adivinhar" o ID de
uma reserva alheia e manipulá-la. Usar um **UUID** (identificador de 128 bits,
praticamente impossível de adivinhar) resolve isso sem precisar de autenticação nesse
ponto do fluxo.

```python
# reservas/models.py
import uuid
from django.db import models
from catalogo.models import Livro


class Reserva(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    criada_em = models.DateTimeField(auto_now_add=True)


class ItemReserva(models.Model):
    reserva = models.ForeignKey(Reserva, on_delete=models.CASCADE, related_name="itens")
    livro = models.ForeignKey(Livro, on_delete=models.CASCADE, related_name="+")
    quantidade = models.PositiveSmallIntegerField(default=1)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["reserva", "livro"], name="livro_unico_por_reserva"), #Dentro de uma mesma reserva, um determinado livro só pode aparecer uma vez.
        ]
```

Dois detalhes que valem destaque:

- `default=uuid.uuid4` — **sem parênteses**. Passar `uuid.uuid4()` (chamando a função)
  geraria o mesmo UUID fixo para todo registro criado a partir daquele ponto (o valor seria
  calculado uma única vez, no momento em que o Django lê `models.py`, não a cada
  `.save()`). O correto é passar a **referência** da função, para o Django chamá-la a cada
  novo registro.
- `models.UniqueConstraint` sobre `(reserva, livro)` impede duas linhas para o mesmo livro
  na mesma reserva — quando o cliente adiciona um livro que já está na reserva, a regra de
  negócio (seção 5.6) deve **somar** a quantidade, não duplicar a linha.

> **Nota sobre custo:** UUID ocupa mais espaço em disco que um inteiro (16 bytes contra 4/8)
> e é, em teoria, um pouco mais lento para indexar. Na prática, para uma tabela que — como
> `Reserva` — é **temporária** (o registro é apagado assim que vira `Emprestimo`, ou pode
> ser limpo por um job periódico depois de alguns dias sem uso), esse custo é irrelevante.
> Já para `Emprestimo`, que é um registro definitivo e nunca é acessado anonimamente, não
> há motivo para pagar esse custo — inteiro autoincrementado continua sendo a escolha certa.

```python
# emprestimos/models.py
from django.db import models
from catalogo.models import Livro
from core.models import Leitor  # perfil do usuário, ver capítulo 6


class Emprestimo(models.Model):
    STATUS_ATIVO = "A"
    STATUS_DEVOLVIDO = "D"
    STATUS_CHOICES = [
        (STATUS_ATIVO, "Ativo"),
        (STATUS_DEVOLVIDO, "Devolvido"),
    ]

    leitor = models.ForeignKey(Leitor, on_delete=models.PROTECT, related_name="emprestimos")
    criado_em = models.DateTimeField(auto_now_add=True)
    status = models.CharField(max_length=1, choices=STATUS_CHOICES, default=STATUS_ATIVO)

    class Meta:
        permissions = [
            ("cancelar_emprestimo", "Pode cancelar um empréstimo"),
        ]


class ItemEmprestimo(models.Model):
    emprestimo = models.ForeignKey(Emprestimo, on_delete=models.PROTECT, related_name="itens")
    livro = models.ForeignKey(Livro, on_delete=models.PROTECT, related_name="itens_emprestimo")
    quantidade = models.PositiveSmallIntegerField()
    preco_unitario = models.DecimalField(max_digits=8, decimal_places=2)
    data_devolucao_prevista = models.DateField()
    data_devolucao_real = models.DateField(null=True, blank=True)
```

Repare que `ItemEmprestimo` guarda `preco_unitario` **no momento do empréstimo**, mesmo já
existindo `livro.preco`. Isso não é redundância por acidente: o preço de um livro pode mudar
com o tempo, mas o **histórico** de um empréstimo já confirmado precisa continuar
mostrando o valor vigente naquela data. Esse é o mesmo motivo por trás de qualquer sistema
de pedidos guardar o preço no item do pedido, e não só uma referência ao produto.

Depois de ajustar os modelos:

```bash
python manage.py makemigrations reservas emprestimos
python manage.py migrate
```

## 5.4 Criar uma reserva

Serializer bem enxuto — o cliente não envia nada, só recebe o `id` de volta:

```python
# reservas/serializers.py
from rest_framework import serializers
from .models import Reserva


class ReservaSerializer(serializers.ModelSerializer):
    class Meta:
        model = Reserva
        fields = ["id"]
        read_only_fields = ["id"]
```

ViewSet montado só com os mixins realmente necessários — **não** usamos `ModelViewSet` aqui
de propósito, porque nunca deveria existir um `GET /api/reservas/` listando todas as
reservas do sistema (isso vazaria os IDs "secretos" de reservas alheias):

```python
# reservas/views.py
from rest_framework import mixins, viewsets
from .models import Reserva
from .serializers import ReservaSerializer


class ReservaViewSet(
    mixins.CreateModelMixin,
    mixins.RetrieveModelMixin,
    mixins.DestroyModelMixin,
    viewsets.GenericViewSet,
):
    queryset = Reserva.objects.prefetch_related("itens__livro")
    serializer_class = ReservaSerializer
```

## 5.5 Devolvendo a reserva com itens e total

O que o cliente realmente quer ver ao consultar uma reserva não é só o `id` — é a lista de
itens (com o livro já "resolvido", não só o ID) e o valor total.

```python
# reservas/serializers.py
from rest_framework import serializers
from catalogo.models import Livro
from .models import Reserva, ItemReserva


class LivroResumidoSerializer(serializers.ModelSerializer):
    """Representação enxuta do livro, só com o que a reserva precisa mostrar."""
    class Meta:
        model = Livro
        fields = ["id", "titulo", "preco"]


class ItemReservaSerializer(serializers.ModelSerializer):
    livro = LivroResumidoSerializer(read_only=True)
    subtotal = serializers.SerializerMethodField()

    class Meta:
        model = ItemReserva
        fields = ["id", "livro", "quantidade", "subtotal"]

    def get_subtotal(self, item: ItemReserva) -> float:
        return item.quantidade * item.livro.preco


class ReservaSerializer(serializers.ModelSerializer):
    itens = ItemReservaSerializer(many=True, read_only=True)
    total = serializers.SerializerMethodField()

    class Meta:
        model = Reserva
        fields = ["id", "itens", "total"]

    def get_total(self, reserva: Reserva) -> float:
        return sum(item.quantidade * item.livro.preco for item in reserva.itens.all())
```

O `queryset` do ViewSet já usa `prefetch_related("itens__livro")` (seção 5.4) — assim, tanto
`ItemReservaSerializer` quanto `get_total()` (que itera `reserva.itens.all()`) reaproveitam
o mesmo cache do QuerySet, sem disparar uma query por item.

## 5.6 Sub-recurso: itens da reserva

Registrando o roteador aninhado:

```python
# biblioteca_config/urls.py
from rest_framework.routers import DefaultRouter
from rest_framework_nested import routers
from reservas.views import ReservaViewSet, ItemReservaViewSet

router = DefaultRouter()
router.register("reservas", ReservaViewSet, basename="reserva")

reservas_router = routers.NestedDefaultRouter(router, "reservas", lookup="reserva")
reservas_router.register("itens", ItemReservaViewSet, basename="reserva-itens")

urlpatterns = [
    path("api/", include(router.urls)),
    path("api/", include(reservas_router.urls)),
]
```

O ponto mais delicado aqui é que **o formato do corpo enviado ao criar um item é diferente
do formato devolvido**: para criar, o cliente manda `livro_id` (um inteiro) — para ler, o
cliente recebe o objeto `livro` completo (seção 5.5). Isso pede **dois serializers**:

```python
# reservas/serializers.py (continuação)

class AdicionarItemReservaSerializer(serializers.ModelSerializer):
    livro_id = serializers.IntegerField()

    class Meta:
        model = ItemReserva
        fields = ["id", "livro_id", "quantidade"]

    def validate_livro_id(self, valor):
        if not Livro.objects.filter(pk=valor).exists():
            raise serializers.ValidationError("Nenhum livro encontrado com esse ID.")
        return valor

    def save(self, **kwargs):
        reserva_id = self.context["reserva_id"]
        livro_id = self.validated_data["livro_id"]
        quantidade = self.validated_data["quantidade"]

        try:
            item = ItemReserva.objects.get(reserva_id=reserva_id, livro_id=livro_id)
            item.quantidade += quantidade
            item.save()
        except ItemReserva.DoesNotExist:
            item = ItemReserva.objects.create(
                reserva_id=reserva_id, livro_id=livro_id, quantidade=quantidade
            )

        self.instance = item
        return self.instance


class AtualizarItemReservaSerializer(serializers.ModelSerializer):
    class Meta:
        model = ItemReserva
        fields = ["quantidade"]
```

`livro_id` é declarado explicitamente porque, embora `ItemReserva.livro_id` exista em
tempo de execução (é o sufixo automático que o Django cria para toda `ForeignKey`), ele
**não é um campo "de verdade" do modelo** no sentido que o `ModelSerializer` reconheceria
sozinho — precisa ser declarado à mão.

Note também que `save()` foi sobrescrito por completo (em vez de só `create()`): a regra
"se o livro já está na reserva, some a quantidade; senão, crie" não se encaixa no
fluxo padrão de `create`/`update` do `ModelSerializer` — ela decide **qual das duas coisas
fazer** dependendo do estado atual do banco. Seguindo a mesma convenção interna do DRF,
sempre terminamos atribuindo o resultado a `self.instance` e retornando-o.

```python
# reservas/views.py (continuação)
from rest_framework import mixins, viewsets


class ItemReservaViewSet(viewsets.ModelViewSet):
    http_method_names = ["get", "post", "patch", "delete", "head", "options"]

    def get_queryset(self):
        return ItemReserva.objects.filter(
            reserva_id=self.kwargs["reserva_pk"]
        ).select_related("livro")

    def get_serializer_class(self):
        if self.request.method == "POST":
            return AdicionarItemReservaSerializer
        if self.request.method == "PATCH":
            return AtualizarItemReservaSerializer
        return ItemReservaSerializer

    def get_serializer_context(self):
        return {**super().get_serializer_context(), "reserva_id": self.kwargs["reserva_pk"]}
```

`http_method_names` bloqueia `PUT` de propósito: não faz sentido substituir um item da
reserva por completo — só faz sentido ajustar a `quantidade` (por isso `PATCH`, e não
`PUT`, no desenho da tabela da seção 5.2).

### Validando a quantidade

```python
from django.core.validators import MinValueValidator

class ItemReserva(models.Model):
    ...
    quantidade = models.PositiveSmallIntegerField(
        default=1, validators=[MinValueValidator(1)]
    )
```

## 5.7 Confirmando o empréstimo: a parte mais delicada

Ao contrário dos endpoints anteriores, aqui `POST /api/emprestimos/` recebe um objeto
(`{"reserva_id": "..."}`) que **não corresponde a nenhum campo direto** de `Emprestimo` —
então usamos um `Serializer` "puro", não um `ModelSerializer`:

```python
# emprestimos/serializers.py
from datetime import date, timedelta
from django.db import transaction
from rest_framework import serializers
from reservas.models import Reserva, ItemReserva
from core.models import Leitor
from .models import Emprestimo, ItemEmprestimo

PRAZO_PADRAO_DIAS = 14


class ConfirmarEmprestimoSerializer(serializers.Serializer):
    reserva_id = serializers.UUIDField()

    def validate_reserva_id(self, valor):
        if not Reserva.objects.filter(pk=valor).exists():
            raise serializers.ValidationError("Nenhuma reserva encontrada com esse ID.")
        if not ItemReserva.objects.filter(reserva_id=valor).exists():
            raise serializers.ValidationError("A reserva está vazia.")
        return valor

    def save(self, **kwargs):
        usuario = self.context["usuario"]
        reserva_id = self.validated_data["reserva_id"]

        with transaction.atomic():
            leitor, _ = Leitor.objects.get_or_create(usuario=usuario)
            emprestimo = Emprestimo.objects.create(leitor=leitor)

            itens_reserva = ItemReserva.objects.select_related("livro").filter(
                reserva_id=reserva_id
            )

            for item in itens_reserva:
                if item.livro.quantidade_disponivel < item.quantidade:
                    raise serializers.ValidationError(
                        f"Estoque insuficiente para '{item.livro.titulo}'."
                    )

            itens_emprestimo = [
                ItemEmprestimo(
                    emprestimo=emprestimo,
                    livro=item.livro,
                    quantidade=item.quantidade,
                    preco_unitario=item.livro.preco,
                    data_devolucao_prevista=date.today() + timedelta(days=PRAZO_PADRAO_DIAS),
                )
                for item in itens_reserva
            ]
            ItemEmprestimo.objects.bulk_create(itens_emprestimo)

            for item in itens_reserva:
                item.livro.quantidade_disponivel -= item.quantidade
                item.livro.save(update_fields=["quantidade_disponivel"])

            Reserva.objects.filter(pk=reserva_id).delete()

        self.instance = emprestimo
        return self.instance
```

Pontos importantes desse trecho, que resumem boa parte do capítulo 2:

- **Toda a operação está dentro de `transaction.atomic()`.** Se a validação de estoque
  falhar no meio do laço, **nada** do que já foi feito antes (criar o empréstimo, criar
  alguns itens) fica gravado — o banco volta ao estado anterior.
- A checagem de estoque é feita **antes** de qualquer `bulk_create`, evitando ter que
  desfazer gravações manualmente.
- `bulk_create` insere todos os itens em uma única instrução SQL, em vez de um `INSERT`
  por item dentro de um laço.
- A reserva é apagada só depois que tudo o mais deu certo.

### Devolvendo um `Emprestimo`, não a `reserva_id` recebida

Assim como fizemos na seção 4.3 do capítulo anterior, a view usa **dois serializers**: um
para ler os dados de entrada, outro para montar a resposta — porque devolver
`{"reserva_id": "..."}` de volta ao cliente seria inútil; ele quer ver o empréstimo criado.

```python
# emprestimos/serializers.py (continuação)

class ItemEmprestimoSerializer(serializers.ModelSerializer):
    livro = LivroResumidoSerializer(read_only=True)   # reaproveitado do app 'reservas'

    class Meta:
        model = ItemEmprestimo
        fields = ["id", "livro", "quantidade", "preco_unitario", "data_devolucao_prevista",
                  "data_devolucao_real"]


class EmprestimoSerializer(serializers.ModelSerializer):
    itens = ItemEmprestimoSerializer(many=True, read_only=True)

    class Meta:
        model = Emprestimo
        fields = ["id", "leitor", "criado_em", "status", "itens"]
        read_only_fields = ["leitor", "criado_em", "status"]
```

```python
# emprestimos/views.py
from rest_framework import mixins, viewsets, status
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from .models import Emprestimo
from .serializers import ConfirmarEmprestimoSerializer, EmprestimoSerializer


class EmprestimoViewSet(
    mixins.CreateModelMixin,
    mixins.ListModelMixin,
    mixins.RetrieveModelMixin,
    viewsets.GenericViewSet,
):
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        queryset = Emprestimo.objects.select_related("leitor__usuario").prefetch_related(
            "itens__livro"
        )
        if self.request.user.is_staff:
            return queryset
        return queryset.filter(leitor__usuario=self.request.user)

    def get_serializer_class(self):
        if self.request.method == "POST":
            return ConfirmarEmprestimoSerializer
        return EmprestimoSerializer

    def get_serializer_context(self):
        return {**super().get_serializer_context(), "usuario": self.request.user}

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        emprestimo = serializer.save()

        resposta = EmprestimoSerializer(emprestimo, context=self.get_serializer_context())
        return Response(resposta.data, status=status.HTTP_201_CREATED)
```

`create()` foi sobrescrito por completo, em vez de reaproveitar `CreateModelMixin.create()`,
justamente para poder trocar de serializer entre "o que valida a entrada"
(`ConfirmarEmprestimoSerializer`) e "o que monta a resposta" (`EmprestimoSerializer`) — sem
isso, o mixin padrão devolveria de volta exatamente o que foi enviado (`reserva_id`).

Repare também em `get_queryset()`: um leitor comum só enxerga os **próprios** empréstimos;
um `staff` (administrador) enxerga todos. Essa regra fica isolada em um único lugar — o
método que toda operação de leitura e escrita da view passa a usar como base.

## 5.8 Uma ação customizada: devolução

Nem toda operação de negócio se encaixa perfeitamente em `list`/`create`/`retrieve`/
`update`/`destroy`. "Devolver um livro" é uma operação própria, com sua própria regra
(incrementar estoque, marcar data de devolução, eventualmente mudar o status do
`Emprestimo` se todos os itens já tiverem voltado). Para isso o DRF oferece o decorador
`@action`:

```python
# emprestimos/views.py (continuação)
from datetime import date
from django.db import transaction
from rest_framework.decorators import action
from .models import ItemEmprestimo


class EmprestimoViewSet(...):
    ...

    @action(detail=True, methods=["post"])
    def devolver(self, request, pk=None):
        emprestimo = self.get_object()

        with transaction.atomic():
            itens_pendentes = emprestimo.itens.filter(data_devolucao_real__isnull=True)
            for item in itens_pendentes:
                item.data_devolucao_real = date.today()
                item.save(update_fields=["data_devolucao_real"])

                item.livro.quantidade_disponivel += item.quantidade
                item.livro.save(update_fields=["quantidade_disponivel"])

            emprestimo.status = Emprestimo.STATUS_DEVOLVIDO
            emprestimo.save(update_fields=["status"])

        serializer = EmprestimoSerializer(emprestimo, context=self.get_serializer_context())
        return Response(serializer.data)
```

- `detail=True` diz que essa ação atua sobre **um** empréstimo específico, então a URL
  gerada inclui o `pk`: `POST /api/emprestimos/{id}/devolver/`. Com `detail=False`, a ação
  ficaria disponível na coleção (`/api/emprestimos/devolver/`, sem `{id}`).
- `self.get_object()` já aplica automaticamente o `get_queryset()` (e, portanto, o filtro
  "só os próprios empréstimos, a não ser que seja staff" da seção 5.7) — se o usuário tentar
  devolver um empréstimo que não é dele, `get_object()` já devolve `404` sozinho.

## 5.9 Um princípio para revisar seu próprio código: separação consulta/comando

Vale nomear explicitamente um princípio que guiou várias decisões deste capítulo (e do
capítulo 2): **Command-Query Separation** — um método deveria ser uma **consulta** (responde
uma pergunta, sem alterar nada) **ou** um **comando** (altera o estado do sistema), nunca as
duas coisas ao mesmo tempo.

Um exemplo do que **evitar**:

```python
# EVITE: get_queryset() é, pelo nome, uma consulta — mas aqui ela também
# cria um Leitor no banco de dados como efeito colateral.
def get_queryset(self):
    leitor, _ = Leitor.objects.get_or_create(usuario=self.request.user)
    return Emprestimo.objects.filter(leitor=leitor)
```

Isso funciona, mas é surpreendente: qualquer `GET` nesse endpoint passaria a gravar dados no
banco, o que ninguém esperaria só de olhar o nome do método. A alternativa (como fizemos na
seção 5.7, usando `leitor__usuario=self.request.user` diretamente, e deixando a criação do
`Leitor` só dentro de `save()`, que já é claramente um comando) mantém cada método fazendo
uma coisa só — o que torna o comportamento do sistema muito mais previsível conforme o
projeto cresce.

---

## Checklist de conceitos deste capítulo

- [ ] Sei justificar quando usar UUID como chave primária (e quando não vale a pena)
- [ ] Sei implementar um ViewSet "customizado", só com os mixins que fazem sentido
- [ ] Sei usar serializers diferentes para entrada e para saída no mesmo endpoint
- [ ] Sei implementar uma regra de "somar se já existe, criar se não existe" em um `save()`
- [ ] Sei encadear múltiplas gravações relacionadas dentro de `transaction.atomic()`
- [ ] Sei criar uma ação customizada com `@action` (`detail=True` x `detail=False`)
- [ ] Consigo reconhecer uma violação do princípio de separação consulta/comando

## Exercícios propostos

1. Implemente a remoção de item da reserva (`DELETE /api/reservas/{id}/itens/{item_id}/`)
   e teste o que acontece ao remover o último item — a reserva deveria continuar existindo
   vazia, ou ser removida automaticamente? Justifique sua escolha e implemente.
2. Adicione uma validação em `ConfirmarEmprestimoSerializer` que impede um mesmo leitor de
   ter mais de 3 empréstimos ativos simultaneamente.
3. Implemente uma ação `@action(detail=False)` em `EmprestimoViewSet` chamada
   `atrasados`, que lista (para o próprio usuário, ou todos se for staff) os empréstimos
   com algum item cuja `data_devolucao_prevista` já passou e `data_devolucao_real` ainda
   é nula.
4. (Desafio) Crie um job (comando de management, `python manage.py limpar_reservas`) que
   apaga reservas com mais de 3 dias sem confirmação — o equivalente ao "abandono de
   carrinho" em um e-commerce.
