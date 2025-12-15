# eSUS-Docker

Implantação do **e-SUS APS PEC** em containers Docker.

> Importante: este projeto **faz o build do `webserver` conectando no Postgres durante o build** (o instalador do e-SUS roda dentro do `Dockerfile`).
> Por isso, é necessário subir o banco primeiro antes de fazer o build do webserver.

---

## Requisitos

- Docker Engine + Docker Compose (plugin)
- Acesso à internet para baixar o instalador (`.jar`) do e-SUS

### Portas

- **Web (PEC)**: `8080` (mapeada como `8080:8080`)
- **PostgreSQL**: `5432` (recomendado **não expor na internet**)

---

## Subindo localmente (Linux)

No diretório raiz do projeto:

```bash
docker compose up -d --build
docker compose ps
```

Se preferir usar o script (Linux):

```bash
sudo sh build-service.sh
```

O script existe porque o build do `webserver` precisa de um banco já rodando e saudável.

---

## Subindo na VPS

### 1) Coloque o projeto na VPS

```bash
git clone <seu-repositorio> /opt/eSUS-Docker
cd /opt/eSUS-Docker
```

### 2) Suba o banco primeiro

```bash
docker compose up -d --build database
docker compose ps
```

**Aguarde o banco ficar healthy.**

### 3) Defina a variável `APP_DB_URL`

O `docker-compose.yml` usa `${APP_DB_URL}` no build/execução. Você pode definir de duas formas:

**Opção A: Criar arquivo `.env` (recomendado)**

```bash
cd /opt/eSUS-Docker

cat > .env <<'EOF'
APP_DB_URL=jdbc:postgresql://database:5432/esus
EOF

cat .env
```

**Opção B: Exportar como variável de ambiente**

```bash
export APP_DB_URL="jdbc:postgresql://database:5432/esus"
```

### 4) Crie o `application.properties` (config do banco)

O `startup.sh` pode ler as configurações do arquivo `application.properties`. Crie no host e monte no container:

```bash
cd /opt/eSUS-Docker
mkdir -p config

cat > config/application.properties <<'EOF'
spring.datasource.url=jdbc:postgresql://database:5432/esus
spring.datasource.username=postgres
spring.datasource.password=esus
spring.datasource.driverClassName=org.postgresql.Driver
EOF

cat config/application.properties
```

**Importante**: use `driverClassName` (sem hífen). `driver-class-name` pode causar erro no export do shell (`bad variable name`) dependendo do script.

### 5) Garanta que o `docker-compose.yml` passa env + monta o properties

No serviço `webserver`, garanta que existam:

**environment:**
```yaml
environment:
  APP_DB_URL: ${APP_DB_URL}
  APP_DB_USER: postgres
  APP_DB_PASSWORD: esus
```

**volumes:**
```yaml
volumes:
  - ./config/application.properties:/opt/e-SUS/webserver/config/application.properties
```

**Recomendação**: não expor `5432` publicamente. Deixe apenas interno (sem `ports:` no `database`).

### 6) Build do `webserver` na VPS (BuildKit/buildx)

Em Docker moderno, o BuildKit pode não suportar `--network` com rede custom do compose no `docker build`.
A abordagem mais estável é usar `buildx` com um builder preso na rede do compose.

Descubra o nome da rede criada pelo compose (geralmente `<pasta>_esus_network`):

```bash
docker network ls | grep esus
```

Exemplo: `esus-docker_esus_network`

Crie/ative um builder buildx usando essa rede:

```bash
docker buildx create --use --name esusbuilder \
  --driver docker-container \
  --driver-opt "network=esus-docker_esus_network"

docker buildx inspect --bootstrap
```

Build da imagem do webserver (com `--load` para ficar disponível localmente):

```bash
cd /opt/eSUS-Docker

docker buildx build --no-cache --load -t esus_webserver:5.4.22 \
  --build-arg APP_DB_URL="jdbc:postgresql://database:5432/esus" \
  --build-arg APP_DB_USER="postgres" \
  --build-arg APP_DB_PASSWORD="esus" \
  --build-arg URL_DOWNLOAD_ESUS="https://arquivos.esusaps.ufsc.br/PEC/e05d0f216b67efe0/5.4.22/eSUS-AB-PEC-5.4.22-Linux64.jar" \
  ./webserver
```

### 7) Suba o webserver (sem build)

```bash
cd /opt/eSUS-Docker
docker compose up -d --no-build --force-recreate webserver
docker compose ps
```

**Logs:**

```bash
docker compose logs -n 200 webserver
```

**Teste da porta:**

```bash
ss -tulpn | grep 8080 || true
```

**Acesso:**

```
http://IP_DA_VPS:8080
```

---

## Customização

- **Versão do e-SUS**: altere o `URL_DOWNLOAD_ESUS` (link do `.jar`) no compose ou no comando de build.
- **Fuso horário**: o `webserver/Dockerfile` define `TZ=Etc/GMT+4`. Ajuste para sua região.

---

## Troubleshooting

### 1) BuildKit não suporta rede custom no `docker build --network ...`

**Sintoma**: erro de network mode "not supported by buildkit"

**Solução**: usar `docker buildx create ... --driver-opt network=<rede>` e `docker buildx build --load ...`.

### 2) Tag `openjdk:17-jdk-bullseye` não encontrada

**Sintoma**: `openjdk:17-jdk-bullseye: not found`

**Solução**: usar imagem base `eclipse-temurin:17-jdk`.

### 3) `apt update` falha com GPG / NO_PUBKEY (Debian bullseye)

**Sintoma**: erros GPG ao atualizar pacotes no build

**Causa comum**: sobrescrever `sources.list` do container com Debian em uma base Ubuntu

**Solução**: manter base e repos consistentes (não misturar).

### 4) Webserver sobe, mas DB URL/Username/Password ficam vazios

**Sintoma**: logs mostram `Database URL = vazio` e erro de conexão

**Solução**: garantir:
- `env APP_DB_URL/USER/PASSWORD` no `docker-compose.yml`
- volume do `application.properties` montado no caminho esperado.

### 5) `export: spring_datasource_driver-class-name: bad variable name`

**Sintoma**: erro de variável inválida

**Solução**: usar `spring.datasource.driverClassName` (sem hífen) no `application.properties`.

### Logs úteis

```bash
docker compose logs -f database
docker compose logs -f webserver
```

---

## Segurança

- Exponha publicamente apenas `8080` (ou prefira proxy com HTTPS).
- Não publique `5432` na internet. Deixe o Postgres apenas na rede Docker.

---

## Observações

- Nomes de imagens/containers/rede e portas podem ser alterados conforme necessidade.
- O `build-service.sh` foi feito para Linux (bash). Em Windows, use os comandos do Docker diretamente.
