# Django do Zero ao Avançado — Material de Estudo

> Material didático elaborado a partir das anotações de um curso completo de Django (~10h),
> reescrito com explicações próprias, tabelas de referência rápida e exemplos de código
> **originais**, usando como fio condutor um projeto único: um **Sistema de Biblioteca**
> (cadastro de livros, autores, categorias, leitores e empréstimos).

A escolha do domínio "biblioteca" não é acidental: ela usa exatamente os mesmos padrões
de relacionamento (1:N, N:N, 1:1, entidade "carrinho/pedido") que aparecem em qualquer
e-commerce, então tudo que você aprender aqui se transfere 100% para APIs de produtos,
pedidos, assinaturas, etc. — e ainda conversa diretamente com o seu próprio projeto de
"Book Rental System".

## Como o material está organizado

| Arquivo | Conteúdo |
|---|---|
| `01-fundamentos-e-modelagem.md` | O que é Django, como a web funciona, primeiro projeto, apps, views, URLs, templates, debug, modelagem de dados, `models.py`, migrations |
| `02-orm-e-admin.md` | QuerySets, lookups, `Q`/`F`, agregação x anotação, `select_related`/`prefetch_related`, transações, SQL bruto, Django Admin completo |
| `03-drf-fundamentos.md` | O que é uma API REST, Django REST Framework, serializers, `@api_view`, validação, relacionamentos em serializers |
| `04-drf-avancado.md` | Class-based views, mixins, generic views, ViewSets, Routers, nested routers, filtros, busca, paginação |
| `05-estudo-de-caso-api-emprestimos.md` | Projeto guiado: API completa de reserva/empréstimo de livros (equivalente a um carrinho de compras) |
| `06-autenticacao-permissoes-signals.md` | Sistema de autenticação, customização de usuário, JWT com Djoser, permissões (built-in e customizadas), signals |

## Pré-requisitos

- Python (funções, classes, herança, decoradores, list comprehensions)
- Noções de banco de dados relacional (tabelas, chaves primárias/estrangeiras, `JOIN`)
- Terminal básico e um editor (VS Code é o usado nos exemplos, mas qualquer um serve)

## Como estudar isso na prática

O curso original insiste bastante em um ponto que vale a pena repetir aqui: **não basta
assistir/ler passivamente**. Para fixar de verdade:

1. Recrie o projeto "biblioteca" no seu ambiente, capítulo por capítulo.
2. Depois de cada seção, feche o material e tente reescrever o trecho de código de memória.
3. Faça os exercícios propostos ao final de cada arquivo **antes** de olhar qualquer solução.
4. Rode o `python manage.py runserver` e o Django Debug Toolbar sempre que quiser confirmar
   quantas queries uma view está disparando — isso cria o hábito de pensar em performance
   desde cedo (problema N+1, por exemplo).

## Stack usada nos exemplos

```
Python 3.11+
Django 5.x
djangorestframework
django-filter
djoser + djangorestframework-simplejwt
PostgreSQL (os exemplos funcionam igual em SQLite/MySQL, só muda a configuração em settings.py)
```

## Modelo de domínio usado em todo o material

```
Autor 1───N Livro N───N Categoria
                │
                │ 1
                │
                N
             Emprestimo N───1 Leitor (perfil do usuário)
                                  │
                                  1
                                  │
                                  1
                               Usuario (autenticação)
```

- **Autor**: quem escreveu o livro.
- **Categoria**: gênero/assunto (Ficção, Programação, História...).
- **Livro**: título, autor, categorias, ISBN, quantidade em estoque.
- **Leitor**: perfil de quem empresta livros (ligado 1‑para‑1 a um usuário autenticado).
- **Empréstimo**: registro de que um leitor pegou um livro emprestado, com datas e status.

Esse modelo aparece, evolui e ganha campos novos ao longo dos seis arquivos — exatamente
como aconteceria em um projeto real.

---

Bons estudos! Sempre que terminar um arquivo, volte a este índice para saber qual é o
próximo passo.
