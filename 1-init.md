# Guia de Instalação Local do Jenkins com Docker

Baseado na documentação oficial do Jenkins: [Installing Jenkins with Docker](https://www.jenkins.io/doc/book/installing/docker/).

---

## Pré-requisitos

- **Hardware mínimo:** 256 MB de RAM e 1 GB de disco (mínimo recomendado em Docker: **10 GB** de disco).
- **Recomendado para estudo/equipe pequena:** 4 GB+ de RAM e 50 GB+ de disco.
- **Software:** Docker instalado e em execução.
  - **No Windows:** o Docker Desktop **precisa estar configurado para Linux Containers** (o projeto Jenkins fornece imagens Linux, não imagens Windows).
  - **No Linux:** configure o Docker para ser gerenciado por usuário não-root (ou execute com `sudo`, se aplicável).

---

## Arquitetura do Ambiente Local

A imagem oficial `jenkins/jenkins:lts-jdk21` (versão **LTS** — *Long-Term Support*, estável para produção, com Java 21) **não inclui o Docker CLI**.  
Para que seus pipelines consigam construir e rodar containers, a arquitetura oficial usa **dois containers** na mesma rede bridge (`jenkins`), comunicando-se via **TLS** (*criptografia por certificados*) na porta `2376`:

1. `jenkins-docker` (`docker:dind`): container *Docker-in-Docker* que executa o daemon Docker interno.
2. `jenkins` (`myjenkins:lts-1`): imagem customizada do Jenkins com `docker-ce-cli` e plugins de Docker pré-instalados.

---

## Passo 1 — Criar a rede isolada no Docker

```bash
docker network create jenkins
```

---

## Passo 2 — Subir o container Docker-in-Docker (`docker:dind`)

### Linux / macOS (Bash/Zsh)

```bash
docker run --name jenkins-docker --rm --detach \
  --privileged --network jenkins --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind --storage-driver overlay2
```

### Windows (PowerShell)

```powershell
docker run --name jenkins-docker --rm --detach `
  --privileged --network jenkins --network-alias docker `
  --env DOCKER_TLS_CERTDIR=/certs `
  --volume jenkins-docker-certs:/certs/client `
  --volume jenkins-data:/var/jenkins_home `
  --publish 2376:2376 `
  docker:dind --storage-driver overlay2
```

### Windows (CMD)

```bat
docker run --name jenkins-docker --rm --detach ^
  --privileged --network jenkins --network-alias docker ^
  --env DOCKER_TLS_CERTDIR=/certs ^
  --volume jenkins-docker-certs:/certs/client ^
  --volume jenkins-data:/var/jenkins_home ^
  --publish 2376:2376 ^
  docker:dind
```

**O que cada opção faz:**
- `--privileged`: necessário para o Docker-in-Docker operar corretamente.
- `--network-alias docker`: expõe o daemon pelo hostname `docker` dentro da rede `jenkins`.
- `--env DOCKER_TLS_CERTDIR=/certs`: habilita TLS no daemon Docker e grava os certificados no volume compartilhado.
- `--volume jenkins-docker-certs:/certs/client`: compartilha os certificados TLS de cliente com o container do Jenkins.
- `--volume jenkins-data:/var/jenkins_home`: compartilha `/var/jenkins_home` para que containers criados pelo `dind` acessem os arquivos do workspace.

---

## Passo 3 — Criar o `Dockerfile` customizado do Jenkins

Crie um arquivo chamado `Dockerfile` em uma pasta local de trabalho:

```dockerfile
FROM jenkins/jenkins:lts-jdk21
USER root
RUN apt-get update && apt-get install -y lsb-release ca-certificates curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
    https://download.docker.com/linux/debian $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && apt-get install -y docker-ce-cli && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER jenkins
RUN jenkins-plugin-cli --plugins "docker-workflow json-path-api"
```

> **Dica:** para máxima reprodutibilidade fora de laboratório, você pode fixar uma versão específica da imagem (ex.: `jenkins/jenkins:2.580.2-jdk21`) em vez de `lts-jdk21`.

---

## Passo 4 — Construir a imagem e iniciar o Jenkins

Na mesma pasta do `Dockerfile`, execute o build:

```bash
docker build -t myjenkins:lts-1 .
```

Em seguida, execute o container do Jenkins:

### Linux / macOS (Bash/Zsh)

```bash
docker run --name jenkins --restart=on-failure --detach \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins:lts-1
```

### Windows (PowerShell)

```powershell
docker run --name jenkins --restart=on-failure --detach `
  --network jenkins `
  --env DOCKER_HOST=tcp://docker:2376 `
  --env DOCKER_CERT_PATH=/certs/client `
  --env DOCKER_TLS_VERIFY=1 `
  --publish 8080:8080 --publish 50000:50000 `
  --volume jenkins-data:/var/jenkins_home `
  --volume jenkins-docker-certs:/certs/client:ro `
  myjenkins:lts-1
```

### Windows (CMD)

```bat
docker run --name jenkins --restart=on-failure --detach ^
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 ^
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 ^
  --volume jenkins-data:/var/jenkins_home ^
  --volume jenkins-docker-certs:/certs/client:ro ^
  --publish 8080:8080 --publish 50000:50000 myjenkins:lts-1
```

**Portas e volumes:**
- `8080:8080`: interface web e API do Jenkins (`http://localhost:8080`).
- `50000:50000`: comunicação de agentes TCP inbound (não é necessária se você usar apenas agentes WebSocket).
- `jenkins-data:/var/jenkins_home`: persiste configurações, jobs, plugins e builds.
- `jenkins-docker-certs:/certs/client:ro`: certificados TLS montados como **somente leitura** (`:ro`).

---

## Passo 5 — Desbloqueio Inicial (*Post-installation Setup Wizard*)

1. Veja os logs do container para copiar a senha inicial de administrador (entre as duas linhas de asteriscos):
   ```bash
   docker logs jenkins
   ```
   *(Ou imprima o arquivo diretamente sem abrir shell interativo, conforme a doc oficial:)*
   ```bash
   docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
   ```
2. Abra `http://localhost:8080` no navegador.
3. Na tela **Unlock Jenkins**, cole a senha no campo **Administrator password** e clique em **Continue**.
4. Em **Customize Jenkins**, escolha **Install suggested plugins** (ou **Select plugins to install** para escolher manualmente).
5. Em **Create First Admin User**, preencha os dados do administrador e clique em **Save and Finish**.
6. Na tela **Jenkins is ready**, clique em **Start using Jenkins** *(se aparecer **Jenkins is almost ready!**, clique em **Restart**)*.

---

## Comandos Úteis do Dia a Dia

- **Abrir um shell dentro do container do Jenkins:**
  ```bash
  docker exec -it jenkins bash
  ```
- **Acompanhar logs em tempo real:**
  ```bash
  docker logs -f jenkins
  ```
- **Parar o ambiente mantendo os dados (volume `jenkins-data`):**
  ```bash
  docker stop jenkins jenkins-docker
  ```
  *(Nota: `jenkins-docker` usa `--rm`, então será removido ao parar; basta rodar o Passo 2 e `docker start jenkins` na próxima sessão.)*
- **Limpar completamente o laboratório (remove containers, rede e volumes):**
  ```bash
  docker rm -f jenkins jenkins-docker
  docker network rm jenkins
  docker volume rm jenkins-data jenkins-docker-certs
  ```

---

## Cenário Corporativo: Certificados CA Customizados (Opcional)

Se sua rede usar proxy corporativo ou certificados autoassinados, **não modifique** diretamente o truststore do sistema (`$JAVA_HOME/lib/security/cacerts`) dentro do container principal como usuário comum. A documentação oficial recomenda usar **Docker Compose com um Init Container** (`cert-init` rodando como `root` para copiar e importar os arquivos `.crt`/`.pem` de `./custom-certs` via `keytool` em um volume compartilhado, montado depois como `:ro` no Jenkins com `JAVA_OPTS="-Djavax.net.ssl.trustStore=/cacerts/cacerts -Djavax.net.ssl.trustStorePassword=changeit"`).
