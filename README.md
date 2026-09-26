# Django + Gunicorn + Nginx + PostgreSQL com Docker Compose

Atividade de Computação em Nuvem: aplicação Django com upload de arquivos, rodando em 3 containers orquestrados pelo Docker Compose.

## Defesa da atividade

A documentação técnica completa, com a explicação e a justificativa de cada decisão de arquitetura e implementação, está em [**Django conteinerizado com Docker Compose.md**](<Django conteinerizado com Docker Compose.md>) (também disponível em [PDF](<Django conteinerizado com Docker Compose.pdf>)). Esse documento é a resposta usada para a defesa da atividade.

## Arquitetura

```
             :8080
Navegador ─────────► [ nginx ] ──(frontend)──► [ web: Django + Gunicorn ] ──(backend)──► [ db: PostgreSQL ]
                        │                              │                                        │
                        ├── /static/ ◄── static_data ──┤                                        │
                        └── /media/  ◄── media_data ───┘                                  postgres_data
```

| Serviço | Imagem | Função |
|---------|--------|--------|
| `web`   | **própria** (`Dockerfile`, `python:3.12-slim`) | Django servido pelo Gunicorn na porta 8000 (interna) |
| `nginx` | `nginx:1.27-alpine` | Proxy reverso: repassa as requisições ao Gunicorn e serve `/static/` e `/media/` direto dos volumes |
| `db`    | `postgres:16-alpine` | Banco de dados |

### Volumes (persistência)

| Volume | Montado em | Conteúdo |
|--------|-----------|----------|
| `media_data` | `web:/app/media` e `nginx:/app/media` (somente leitura) | **Arquivos enviados pelo upload** |
| `static_data` | `web:/app/staticfiles` e `nginx:/app/staticfiles` | Arquivos estáticos (`collectstatic`) |
| `postgres_data` | `db:/var/lib/postgresql/data` | Dados do banco |

Os volumes são nomeados, então existem independentemente dos containers: `docker compose down` remove os containers, mas os arquivos e o banco continuam lá. Só `docker compose down -v` apaga os volumes.

### Redes

- `frontend`: nginx ↔ web
- `backend`: web ↔ db

O nginx não enxerga o banco, e o banco não expõe porta para o host. A única porta publicada é a `8080` do nginx.

## Como rodar

```bash
cp .env.example .env        # ajuste as variáveis se quiser
docker compose up -d --build
```

Acesse **http://localhost:8080**.

Comandos úteis:

```bash
docker compose ps                                   # status dos containers
docker compose logs -f                              # logs
docker compose exec web python manage.py createsuperuser   # usuário para o /admin
docker compose down                                 # para (mantém volumes)
docker compose down -v                              # para e APAGA os volumes
docker volume ls                                    # lista os volumes
```

## Testando a persistência do upload

1. Envie um arquivo em http://localhost:8080.
2. Rode `docker compose down` e depois `docker compose up -d`.
3. Volte à página: o arquivo continua listado e abrindo, porque está no volume `media_data` e o registro está no `postgres_data`.

Também dá para ver o arquivo dentro do volume:

```bash
docker compose exec web ls -R /app/media
```

## Estrutura

```
.
├── docker-compose.yml     # orquestra os 3 serviços, volumes e redes
├── Dockerfile             # imagem própria do Django/Gunicorn
├── entrypoint.sh          # migrate + collectstatic antes de iniciar o Gunicorn
├── requirements.txt
├── .env.example
├── nginx/
│   └── default.conf       # configuração do proxy reverso
└── app/
    ├── manage.py
    ├── config/            # settings, urls, wsgi
    └── arquivos/          # app de upload (model, form, views, template)
```

## Detalhes da implementação

- **Dockerfile**: instala as dependências antes de copiar o código, para aproveitar o cache de camadas, e roda a aplicação com um usuário sem privilégios (`appuser`).
- **entrypoint.sh**: aplica as migrações e coleta os estáticos toda vez que o container sobe, depois chama o Gunicorn com `exec`.
- **depends_on + healthcheck**: o `web` só sobe depois que o Postgres responde ao `pg_isready`.
- **Upload**: o model `Arquivo` usa `FileField(upload_to="uploads/%Y/%m/%d/")`. O arquivo vai para `MEDIA_ROOT` (`/app/media`, o volume) e o banco guarda só o caminho.
- **nginx**: `client_max_body_size 20M` limita o tamanho do upload. Os arquivos de `/media/` e `/static/` são entregues pelo nginx sem passar pelo Django.
- **Configuração** via variáveis de ambiente (`.env`), então o código não tem senhas fixas.
