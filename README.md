# ecosol-infra

Orquestração do ambiente de desenvolvimento da **Plataforma de Rede da Economia
Solidária de Niterói** — Centro Público de Referência em Economia Solidária
(Casa Paul Singer) · ITES / IFRJ Campus Niterói.

Um comando sobe os três serviços: Postgres, o backend Django e o frontend
Next.js.

**Este repositório não tem código de aplicação.** Ele só descreve como as peças
se ligam. O código vive em `ecosol-backend` e `ecosol-frontend`.

## Requisitos

- Docker Desktop (ou Docker Engine + Compose v2.20 ou superior, pelo `include`)
- Os dois repositórios de código clonados ao lado deste

## Árvore esperada

```
ecosol-fullstack/
├── apps/
│   ├── ecosol-backend/     # repo Git → Django + DRF
│   └── ecosol-frontend/    # repo Git → Next.js
└── ecosol-infra/           # ESTE repositório
    └── docker-compose.yml
```

O nome da pasta é o que `git clone` cria por padrão — não o renomeie por
gosto, mas saiba que ele **não** afeta nada: os caminhos do compose são
relativos ao próprio arquivo, e o nome do projeto Docker está fixado no YAML
(`name: infra`), não vem da pasta.

Se a sua árvore for diferente, copie `.env.example` para `.env` e ajuste
`CAMINHO_BACKEND` e `CAMINHO_FRONTEND`. Sem `.env`, os padrões acima já valem.

## Subir tudo

```bash
git clone <url-do-ecosol-backend>  ../apps/ecosol-backend
git clone <url-do-ecosol-frontend> ../apps/ecosol-frontend

docker compose up --build
```

| Serviço | Endereço |
|---|---|
| Site público | <http://localhost:3000> |
| API | <http://localhost:8001/api/coletivos/> |
| Área administrativa | <http://localhost:8001/admin/> |
| Postgres | `localhost:5432` (`ecosol` / `ecosol` / `ecosol`) |

Para criar a primeira conta de administrador:

```bash
docker compose exec backend python manage.py createsuperuser
```

## O ambiente é fechado

Nada aqui toca o Supabase. O banco é um Postgres 16 em container, as imagens
enviadas ficam em `media/` dentro do repositório do backend, e o `.env` da
máquina — que aponta para o Supabase real — é ignorado dentro dos containers
(`DJANGO_IGNORE_DOTENV=True`). Dá para quebrar tudo à vontade.

## Duas decisões que explicam o arquivo

**O banco e o backend não são declarados aqui.** Eles vêm por `include` do
compose que já existe em `ecosol-backend`. Manter duas cópias das mesmas
definições foi exatamente como a pasta `infra/` anterior envelheceu em silêncio,
até alguém subir o ambiente errado e vazar o `.env` de produção para dentro do
container. Cada serviço é descrito uma vez, no repositório do código que ele
executa.

**O frontend recebe dois endereços da API, e eles são diferentes.** O servidor
do Next fala com o Django pela rede interna do Docker (`http://backend:8001`);
o navegador de quem acessa, não — para ele, `backend` não existe. Como o Django
monta a URL das imagens a partir do host da requisição, sem a segunda variável
(`API_URL_PUBLICA`) as imagens voltariam apontando para um endereço que só
existe dentro do Docker, e quebrariam no navegador sem erro nenhum no log.

## Rodar só o backend

O `ecosol-backend` tem um compose próprio, com Postgres e Django apenas — útil
para trabalho de backend, sem esperar a imagem do Next:

```bash
cd ../apps/ecosol-backend/infra && docker compose up
```

Os dois usam o **mesmo nome de projeto** (`infra`) — o do backend porque o
Compose adota o nome da pasta, e este porque o declara explicitamente no YAML.
Isso é de propósito: os dois compartilham
containers e volume de banco. Se fossem projetos separados, existiriam dois
Postgres, e quem cadastrasse um coletivo pelo Admin de um não o veria no site do
outro — sem nenhuma mensagem explicando por quê.

O preço: subir só o backend depois de ter subido tudo faz o Compose avisar que
`frontend` é um container órfão. É só um aviso — **não use `--remove-orphans`**
nessa situação.

## Integração contínua

Não há código para testar, então o CI valida o que este repositório pode
quebrar: clona os dois repos de código, resolve o compose (`docker compose
config`), confere que os três serviços continuam declarados e que o Django
aceita o `Host` que o frontend usa. Esse último é um erro que só aparece em
tempo de execução, como um `400` que não menciona a causa.

## Quando houver deploy real

O compose constrói as imagens a partir dos repositórios ao lado. Isso resolve o
ambiente de desenvolvimento, mas amarra este repositório a uma árvore de pastas.
No dia em que os CIs do backend e do frontend publicarem imagens num registry,
a migração é trocar `build:` por `image:` — sem reescrever o resto.

## Fluxo Git

Branches por tarefa saindo de `staging`, Conventional Commits em português,
`main` e `staging` protegidas. Tudo em português — é requisito de Tecnologia
Social do projeto.

Licença: GPLv3.
