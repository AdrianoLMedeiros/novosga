# Manual de Deploy para Homologação/Testes — NovoSGA

Este manual cobre como subir o NovoSGA (backend + módulos) localmente para testes/homologação. Existem dois caminhos:

- **Opção A — Docker** (recomendado, é o fluxo oficial do projeto)
- **Opção B — Sem Docker** (alternativa nativa, usada quando o ambiente não permite rodar o daemon Docker — ex: sandboxes/containers restritos)

---

## Pré-requisitos

- PHP **8.2 ou 8.3** (o `composer.lock` trava dependências — ex: `league/oauth2-server` — que não suportam PHP 8.4 ainda; ver seção de Troubleshooting)
- Composer 2.x
- PostgreSQL 16 (via Docker ou instalação nativa)
- Extensões PHP: `pdo_pgsql`, `iconv`, `simplexml`, `tokenizer`, `xmlwriter`, `fileinfo`, `sodium`, `xsl`
- OpenSSL (para gerar as chaves JWT do OAuth)

---

## Opção A — Docker (fluxo oficial)

```bash
# 1. Subir Postgres + Mercure (hub de eventos em tempo real)
docker compose up -d

# 2. Instalar dependências PHP
composer install

# 3. Gerar as chaves JWT do OAuth
mkdir -p config/jwt
openssl genrsa -out config/jwt/private.pem 2048
openssl rsa -in config/jwt/private.pem -pubout -out config/jwt/public.pem

# 4. Rodar o instalador do NovoSGA (cria admin, unidade, prioridades, local, roda migrations)
php bin/console novosga:install

# 5. Subir o servidor
symfony server:start
# ou, sem o Symfony CLI:
php -S 127.0.0.1:8000 -t public
```

O `compose.override.yaml` já expõe as portas do Postgres (5432), Mercure (3000) e adiciona o **Mailpit** (captura de e-mails de teste em `http://127.0.0.1:8025`).

Acesse: `http://127.0.0.1:8000`

---

## Opção B — Sem Docker (ambientes sem suporte ao daemon Docker)

Use Postgres instalado nativamente no host.

```bash
# 1. Subir o Postgres local
sudo service postgresql start
# ou: pg_ctlcluster 16 main start

# 2. Criar usuário e banco (se ainda não existirem)
sudo -u postgres psql -c "CREATE ROLE app LOGIN PASSWORD 'app' CREATEDB;"
sudo -u postgres createdb -O app app

# 3. Apontar a aplicação para esse banco
cat > .env.local << 'EOF'
DATABASE_URL="postgresql://app:app@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
EOF

# 4. Instalar dependências PHP
#    (use --ignore-platform-reqs apenas se estiver em PHP 8.4+; ver Troubleshooting)
composer install --no-interaction

# 5. Gerar as chaves JWT do OAuth
mkdir -p config/jwt
openssl genrsa -out config/jwt/private.pem 2048
openssl rsa -in config/jwt/private.pem -pubout -out config/jwt/public.pem

# 6. Validar variáveis de ambiente antes de instalar
php bin/console novosga:check

# 7. Rodar o instalador
php bin/console novosga:install
```

> Sem o Mercure, funcionalidades de atualização em tempo real (ex: painel de chamada) não vão funcionar, mas o restante do sistema opera normalmente.

### Instalação não-interativa (CI / scripts)

O `novosga:install` aceita variáveis de ambiente para não pedir input:

```bash
NOVOSGA_ADMIN_USERNAME=admin \
NOVOSGA_ADMIN_PASSWORD=admin123 \
NOVOSGA_ADMIN_FIRSTNAME=Administrador \
NOVOSGA_ADMIN_LASTNAME=Sistema \
NOVOSGA_UNITY_NAME="Unidade Central" \
NOVOSGA_UNITY_DESCRIPTION=UNI1 \
NOVOSGA_UNITY_TIMEZONE="America/Sao_Paulo" \
NOVOSGA_NOPRIORITY_NAME=Normal \
NOVOSGA_NOPRIORITY_DESCRIPTION="Sem prioridade" \
NOVOSGA_PRIORITY_NAME=Prioridade \
NOVOSGA_PRIORITY_DESCRIPTION="Atendimento prioritário" \
NOVOSGA_PLACE_NAME="Guichê" \
php bin/console novosga:install --no-interaction
```

### Subindo o servidor

```bash
php -S 127.0.0.1:8000 -t public
```

Acesse: `http://127.0.0.1:8000` → redireciona para `/login`.

---

## Login e validação

1. Acesse `/login`.
2. Use o usuário/senha definidos no instalador (ex: `admin` / `admin123`).
3. **Atenção ao formulário**: os campos do form de login são `username` e `password` (não `_username`/`_password`), e exigem o token `_csrf_token` presente no HTML da página — relevante se for testar via `curl`/script em vez do navegador.

### Acessando os módulos instalados

Cada bundle de módulo é montado sob um prefixo derivado do namespace (ver `src/Loader/RouterLoader.php`):

| Papel | Bundle | Incluso no `composer.json`? | Prefixo de rota |
|---|---|---|---|
| Totem/auto-atendimento (público escolhe serviço) | `triage-bundle` | ✅ Sim | `/novosga.triage/` |
| Atendente (chamar/atender senha no guichê) | `attendance-bundle` | ✅ Sim | `/novosga.attendance/` |
| Monitor/supervisão (acompanhar fila, transferir senhas) | `monitor-bundle` | ✅ Sim | `/novosga.monitor/` |
| Painel de TV (exibição pública da senha chamada) | `panel-bundle` | ❌ **Não incluso** — ver limitação abaixo | — |
| Admin (nativo, não é módulo) | — | — | `/admin` |

Exemplo: `http://127.0.0.1:8000/novosga.triage/`

#### ⚠️ Limitação conhecida: painel de TV (`panel-bundle`) não pode ser instalado sem atualizar o Symfony

O `novosga/panel-bundle` (que mostra a senha chamada em telas de TV nos guichês/salas) **não está nas dependências deste projeto** e, ao tentar adicioná-lo, o Composer falha:

```
novosga/panel-bundle v2.3.x-dev requires symfony/form 7.4.* -> conflicts with root composer.json (7.1.*)
novosga/core v2.3.x-dev requires symfony/http-kernel 7.4.* -> conflicts
```

A causa raiz: este `composer.json` fixa **mais de 25 pacotes Symfony** em `7.1.*`, enquanto a branch `2.3.x-dev` do `novosga/core` (dependência transitiva de quase todos os bundles) já exige Symfony `7.4.*`. Ou seja, o ecossistema de bundles avançou para 7.4 e este backend ainda não.

Resolver isso exige uma **atualização de framework em escala** (editar todas as constraints `7.1.*` → `7.4.*` neste `composer.json`, rodar `composer update -W` e validar toda a aplicação contra possíveis breaking changes do Symfony 7.1→7.4), não um simples `composer require`. Foi deliberadamente deixado como tarefa separada, a ser planejada e testada com mais cuidado (suíte de testes completa, revisão de changelog do Symfony) antes de ser executada.

> A tela de Triagem só mostra serviços se houver `Servico` + `ServicoUnidade` cadastrados para a unidade do usuário logado. Recém-instalado, ela aparecerá vazia — é esperado.

---

## Customização de aparência (sem alterar código)

Acesse `/admin` logado como administrador → seção "Aparência":

- Upload de logo da navbar
- Upload de logo da tela de login
- Escolha de tema (26 temas Bootswatch)
- Cor da navbar (Light/Dark/Primary/Tertiary)

---

## Troubleshooting

### `composer install` falha com erro de `league/oauth2-server` e versão de PHP

```
- league/oauth2-server is locked to version 9.0.1 ... your php version (8.4.x) does not satisfy that requirement.
```

O `composer.lock` atual trava em compatibilidade até PHP 8.3 (o `Dockerfile` oficial usa PHP 8.2 via `trafex/php-nginx`). Em ambiente de teste com PHP 8.4, contorne com:

```bash
composer install --ignore-platform-reqs
```

**Isso não deve ser usado em produção** — para produção, use PHP 8.2/8.3 (igual ao `Dockerfile`) ou atualize a dependência (`composer update league/oauth2-server league/oauth2-server-bundle`) e valide compatibilidade.

### Docker não sobe (`failed to connect to the docker API ... no such file or directory`)

O daemon Docker não está disponível/rodando no host (comum em sandboxes/CI restritos). Use a **Opção B** deste manual (Postgres nativo).

### Login retorna "Credenciais inválidas" mesmo com usuário/senha corretos

Confira se o POST está usando os nomes de campo certos: `username` e `password` (não `_username`/`_password`), e se o `_csrf_token` enviado é o mesmo presente no HTML da página de login carregada na mesma sessão/cookie.

### Tela de Triagem aparece vazia

Cadastre ao menos um `Serviço` (Admin → Serviços) e configure-o para a `Unidade` (Admin → Unidades → Serviços da unidade) antes de testar a triagem.

---

## Resetando o ambiente

```bash
# Limpar cache da aplicação
php bin/console cache:clear

# Resetar dados (existe comando dedicado no projeto)
php bin/console novosga:reset
```
