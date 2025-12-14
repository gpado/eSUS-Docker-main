# eSUS-Docker

Implantação do **e-SUS APS PEC** em containers Docker.

> Importante: este projeto **faz o build do `webserver` conectando no Postgres durante o build** (o instalador do e-SUS roda dentro do `Dockerfile`).
> Por isso, em geral, **não dá para usar “Stack + build” direto no Portainer** sem preparar o banco antes.

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

## Subindo na VPS com Portainer (Hostinger)

### Visão geral (recomendado)

1) Subir o Postgres na VPS
2) Buildar a imagem do `webserver` na VPS **com a rede do compose**
3) No Portainer, criar uma Stack **sem `build:`** (somente `image:`)

### 1) Coloque o projeto na VPS

Exemplos:

```bash
# opção A: via git
git clone <seu-repositorio> /opt/eSUS-Docker
cd /opt/eSUS-Docker

# opção B: via upload (SFTP/WinSCP)
cd /opt/eSUS-Docker
```

### 2) Suba o banco primeiro

```bash
docker compose up -d --build database
docker compose ps
```

### 3) Build do `webserver` na VPS (apontando para o serviço `database`)

Esse comando permite que o `docker build` enxergue o Postgres pela rede `esus_network`.

```bash
docker build --network esus_network -t esus_webserver:5.4.22 \
  --build-arg APP_DB_URL="jdbc:postgresql://database:5432/esus" \
  --build-arg APP_DB_USER="postgres" \
  --build-arg APP_DB_PASSWORD="esus" \
  --build-arg URL_DOWNLOAD_ESUS="https://arquivos.esusaps.ufsc.br/PEC/e05d0f216b67efe0/5.4.22/eSUS-AB-PEC-5.4.22-Linux64.jar" \
  ./webserver
```

(Opcional) Build da imagem do banco:

```bash
docker build -t esus_database:1.0.0 ./database
```

### 4) Portainer: Stack (sem `build:`) + volume do Postgres

No Portainer: **Stacks → Add stack → Web editor** e cole:

```yaml
services:
  database:
    container_name: esus_database
    image: esus_database:1.0.0
    environment:
      POSTGRES_DB: esus
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: esus
    volumes:
      - esus_pgdata:/var/lib/postgresql/data
    networks:
      - esus_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      retries: 5
      start_period: 30s
      timeout: 10s

  webserver:
    container_name: esus_webserver
    image: esus_webserver:5.4.22
    ports:
      - '8080:8080'
    networks:
      - esus_network
    depends_on:
      database:
        condition: service_healthy

networks:
  esus_network:
    driver: bridge

volumes:
  esus_pgdata:
```

Clique em **Deploy the stack**.

### 5) Firewall / segurança

- Libere **apenas** a porta `8080` (ou coloque atrás de Nginx/Traefik com HTTPS).
- **Não publique** `5432` para a internet. Deixe o Postgres só na rede Docker.

---

## Customização

- **Versão do e-SUS**: altere o `URL_DOWNLOAD_ESUS` (link do `.jar`) no `docker-compose.yml` ou no comando de `docker build`.
- **Fuso horário**: o `webserver/Dockerfile` define `TZ=Etc/GMT+4`. Ajuste para sua região.

---

## Troubleshooting

- **Build do webserver falha dizendo que não conecta no banco**:
  - na VPS, suba o serviço `database` primeiro e faça o build com `--network esus_network`.
- **Portainer Stack não builda**:
  - use Stack sem `build:` (só `image:`) e faça o build das imagens na VPS antes.
- **Ver logs**:

```bash
docker compose logs -f database
docker compose logs -f webserver
```

---

## Observações

- Nomes de imagens/containers/rede e portas podem ser alterados conforme sua necessidade.
- O script `build-service.sh` foi feito para ambiente Linux (bash). Em Windows, use os comandos do Docker diretamente.
