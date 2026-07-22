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

## Deploy no EasyPanel

> **Sem isso, toda atualização apaga mídia, páginas publicadas e backups
> locais**, mesmo sem nenhuma mudança de conteúdo — o `Dockerfile` só marca
> `/app/public` como ponto de montagem (`VOLUME /app/public`); ele **não**
> tem como forçar nenhuma plataforma a manter um volume persistente ali.
> Quem garante isso é a configuração do deploy, não a imagem.

Existem dois jeitos de subir este projeto no EasyPanel — escolha um:

### Opção recomendada: App do tipo Compose

1. Crie o App como **Compose** (não "App a partir de um repositório/Dockerfile").
2. Cole o conteúdo deste `docker-compose.yml` (ou aponte para este
   repositório).
3. Suba o app.

Nesse modo o EasyPanel lê o `docker-compose.yml` e cria os volumes nomeados
(`kastir-public`, `kastir-postgres`, `kastir-redis`) automaticamente — eles
persistem entre atualizações **sem nenhum passo manual**, contanto que você
sempre atualize o mesmo Compose App (não crie um novo do zero).

> **Aviso "ports is used in app. It might cause conflicts with other
> services"** — esperado, pode ignorar. O serviço `app` publica a porta
> direto no host (`ports: - "${HTTP_PORT:-8080}:80"`) em vez de depender da
> aba Domínios do EasyPanel, de propósito: é o jeito que funciona igual em
> qualquer host Docker (VPS puro, Coolify, EasyPanel, etc.), sem exigir
> nenhuma configuração específica de plataforma. Só vira problema de verdade
> se outro serviço no **mesmo host** já estiver usando a mesma porta —
> nesse caso, mude `HTTP_PORT` no `.env` para uma porta livre.

### Alternativa: App a partir do repositório/Dockerfile (GitHub)

Se preferir apontar o EasyPanel direto para o Dockerfile (build a partir do
GitHub, sem usar Compose), o EasyPanel **não lê** o `docker-compose.yml` —
ele não sabe que `/app/public` precisa ser persistente. Você precisa
configurar isso manualmente **uma vez**, antes do primeiro upload de mídia
ou publicação de página:

1. No app, abra a aba **Mounts** (ou "Volumes", dependendo da versão).
2. Adicione um mount do tipo **Volume** (não "Bind" para um path temporário
   do host) com **Mount Path**: `/app/public`.
3. Salve e faça o primeiro deploy.
4. Depois de qualquer atualização futura ("Rebuild"/"Deploy"), confirme que
   o mount continua listado na aba Mounts — se o EasyPanel recriar o app do
   zero em vez de atualizá-lo, o volume pode não ser reanexado.

Essa mesma regra vale para **qualquer** host Docker, não só EasyPanel
(Coolify, VPS própria, etc.): o volume precisa sobreviver a um *redeploy*,
não só a um *restart* do mesmo container.

## Variáveis de ambiente

| Variável | Obrigatória | Uso |
| --- | --- | --- |
| `DATABASE_URL` | Sim | Connection string do Postgres que o app usa. Default no `.env.example` já aponta pro `postgres` deste compose — só troque se for reaproveitar um Postgres já existente |
| `REDIS_URL` | Sim | Connection string do Redis que o app usa. Mesma ideia: default aponta pro `redis` deste compose |
| `NEXTAUTH_SECRET` | Sim | Secret do Better Auth (≥ 32 caracteres aleatórios) |
| `BETTER_AUTH_URL` | Sim | Origem pública (cookies/CSRF/redirects) — deve bater com a URL do navegador |
| `MASTER_EMAIL` | Sim | E-mail do Administrador Master no primeiro boot |
| `MASTER_PASSWORD` | Sim | Senha do Master no primeiro boot |
| `AUTH_TRUST_PROXY` | Não | Default `true` (Nginx do container sobrescreve headers de IP) |
| `MASTER_FORCE_RESET` | Não | `true` só para recuperação controlada de credenciais |
| `SITE_TITLE` / `PANEL_LANGUAGE` | Não | Defaults do painel |
| `HTTP_PORT` | Não | Porta do host → Nginx; default `8080` |
| `POSTGRES_*` / `REDIS_PASSWORD` | Não | Só inicializam os containers `postgres`/`redis` **deste** compose — se `DATABASE_URL`/`REDIS_URL` apontarem pra fora, essas variáveis não têm efeito nenhum |
| `KASTIR_IMAGE` | Não | Override da imagem; default `ghcr.io/korflux/kastir:latest` |
| `BACKUP_MAX_BYTES` | Não | Teto do pacote de backup/import (bytes). Default na app se omitido |

Já tem Postgres/Redis rodando (outro serviço no mesmo projeto EasyPanel, uma
instância gerenciada)? Aponte `DATABASE_URL`/`REDIS_URL` pra eles e remova os
serviços `postgres`/`redis` (+ seus volumes) deste `docker-compose.yml` — não
precisa reconstruir a URL a partir de usuário/senha separados.

## Persistência

Se você usa EasyPanel, veja primeiro [Deploy no EasyPanel](#deploy-no-easypanel)
acima — fora do `docker-compose.yml` puro, a plataforma precisa saber manter
esses volumes entre atualizações.

Volumes nomeados:

- `kastir-public` — **todo** o conteúdo sob `/app/public` no container:
  - uploads de mídia (`uploads/`)
  - HTML publicado das páginas
  - `robots.txt`, `sitemap.xml`
  - **backups locais** do painel (`_kastir/backups/`) — export/import/restore
    da fase 22; o Nginx da imagem **não** serve esse path publicamente
- `kastir-postgres` — dados do Postgres
- `kastir-redis` — dados do Redis (filas / rate limit)

O `docker-compose.yml` **não** precisa de volume extra para backups: eles
já vivem no volume `kastir-public`. Apagar esse volume apaga mídia, site
publicado **e** histórico de backup.

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

Features novas (ex.: Backup no painel) só aparecem depois que a **imagem**
no GHCR for republicada com o código novo. Este repositório fino só puxa a
imagem — não embute o app.

## Imagem privada / login no GHCR

Se o pacote no GitHub Container Registry estiver privado, autentique antes do pull:

```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u SEU_USER --password-stdin
docker compose pull
```

## Licença

Ver `LICENSE` neste repositório.
