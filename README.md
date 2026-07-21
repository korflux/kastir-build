# Kastir (self-hosted)

Este repositório distribui o **runtime** do Kastir para deploy via Docker.
O código-fonte de desenvolvimento **não** está aqui — a aplicação vem de uma
imagem pré-construída (`ghcr.io/korflux/kastir`).

## Requisitos

- Docker Engine + Docker Compose
- Uma porta livre no host (padrão `8080`)

Postgres e Redis já vêm no `docker-compose.yml` como serviços internos
(não expostos no host).

## Subir

```bash
cp .env.example .env
# edite .env: NEXTAUTH_SECRET, BETTER_AUTH_URL, MASTER_EMAIL, MASTER_PASSWORD

docker compose up -d
docker compose logs -f app
```

Quando os logs mostrarem migrations e bootstrap concluídos, abra:

`http://SEU_HOST:8080/admin/login` (ou a porta em `HTTP_PORT`).

## Variáveis de ambiente

| Variável | Obrigatória | Uso |
| --- | --- | --- |
| `NEXTAUTH_SECRET` | Sim | Secret do Better Auth (≥ 32 caracteres aleatórios) |
| `BETTER_AUTH_URL` | Sim | Origem pública (cookies/CSRF/redirects) — deve bater com a URL do navegador |
| `MASTER_EMAIL` | Sim | E-mail do Administrador Master no primeiro boot |
| `MASTER_PASSWORD` | Sim | Senha do Master no primeiro boot |
| `AUTH_TRUST_PROXY` | Não | Default `true` (Nginx do container sobrescreve headers de IP) |
| `MASTER_FORCE_RESET` | Não | `true` só para recuperação controlada de credenciais |
| `SITE_TITLE` / `PANEL_LANGUAGE` | Não | Defaults do painel |
| `HTTP_PORT` | Não | Porta do host → Nginx; default `8080` |
| `POSTGRES_*` / `REDIS_PASSWORD` | Não | Credenciais dos serviços internos; **troque em produção** |
| `KASTIR_IMAGE` | Não | Override da imagem; default `ghcr.io/korflux/kastir:latest` |

## Persistência

Volumes nomeados:

- `kastir-public` — uploads, páginas publicadas, `robots.txt`, `sitemap.xml`
- `kastir-postgres` — dados do Postgres
- `kastir-redis` — dados do Redis

Ao atualizar, **não** apague esses volumes — só puxe a nova imagem e suba de novo:

```bash
docker compose pull
docker compose up -d
```

Migrations e o bootstrap do Master são idempotentes a cada boot.

## Atualizações

1. `docker compose pull` (ou defina `KASTIR_IMAGE` com a tag desejada)
2. `docker compose up -d`
3. Mantenha o mesmo `.env` e os volumes

## Imagem privada / login no GHCR

Se o pacote no GitHub Container Registry estiver privado, autentique antes do pull:

```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u SEU_USER --password-stdin
docker compose pull
```

## Licença

Ver `LICENSE` neste repositório.
