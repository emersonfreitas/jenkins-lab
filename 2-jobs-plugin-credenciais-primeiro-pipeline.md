# Jenkins — Aula 2: Jobs, Plugins, Credenciais e Primeiro Pipeline

Guia prático baseado estritamente na documentação oficial do Jenkins:
- [Best Practices](https://www.jenkins.io/doc/book/using/best-practices/)
- [Managing Plugins](https://www.jenkins.io/doc/book/managing/plugins/)
- [Managing Jenkins](https://www.jenkins.io/doc/book/managing/)
- [Using credentials](https://www.jenkins.io/doc/book/using/using-credentials/)
- [Getting started with Pipeline](https://www.jenkins.io/doc/book/pipeline/getting-started/)
- [Using a Jenkinsfile](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/)

---

## 1. Tipos de Projetos (Jobs) — Ordem de Preferência Oficial

O Jenkins moderno prioriza **Pipelines** versionados em **SCM** (*Source Control Management*, como Git) em vez de jobs manuais (*Freestyle*). Conforme a página *Best Practices*, a ordem recomendada é:

1. **Organization Folders** *(preferido em equipes)*: monitora uma organização inteira (GitHub, GitLab, Bitbucket ou Gitea) e cria/remove pipelines automaticamente conforme repositórios e branches são criados ou excluídos.
2. **Multibranch Pipelines**: monitora **um** repositório específico e cria/remove builds automaticamente para cada branch que possua um `Jenkinsfile`.
3. **Pipeline**: job único onde o script é definido diretamente na interface web ou lido de um `Jenkinsfile` no SCM.

> **Boa prática oficial:** evite espaços no campo **Enter an item name**, pois o Jenkins usa o nome do item para criar diretórios em disco e scripts podem falhar caso não tratem espaços corretamente.

---

## 2. Plugins — Extensões do Jenkins

Plugins são empacotados em arquivos `.hpi` autocontidos e distribuídos pelo **Update Center** (*repositório oficial de plugins*).

- **Via interface web:** **Manage Jenkins** (ícone de engrenagem no canto superior direito) → **Plugins** → aba **Available** → selecione o plugin e clique em **Install without restart** *(se a lista aparecer vazia, clique em **Check now** para buscar metadados do Update Center)*.
- **Via Jenkins CLI (automação):**
  ```bash
  java -jar jenkins-cli.jar -s http://localhost:8080/ install-plugin SOURCE ... [-deploy] [-restart]
  ```
- **Via imagem Docker (build imutável):** usando `jenkins-plugin-cli --plugins "..."` no `Dockerfile`.

---

## 3. Credenciais — Passo a Passo Exato na Interface

O gerenciamento de credenciais é fornecido pelo **Credentials Binding plugin**. Os segredos ficam criptografados no **Controller** (usando o *Controller ID*) e nos pipelines são referenciados apenas pelo **Credential ID**.

### Tipos suportados
- **Secret text:** tokens de API ou Personal Access Tokens (GitHub/GitLab).
- **Username and password:** usuário e senha (separados ou no formato `usuario:senha`).
- **Secret file:** arquivo com conteúdo confidencial.
- **SSH Username with private key:** par de chaves SSH (pública/privada).
- **Certificate:** arquivo de certificado **PKCS#12** e senha opcional.
- **Docker Host Certificate Authentication:** certificados para autenticação em daemons Docker.

### Como cadastrar uma credencial global (Exercício)
1. No **Dashboard**, clique no **ícone de engrenagem** no canto superior direito para abrir **Manage Jenkins**.
2. No grupo **Security**, clique em **Manage Credentials**.
3. Em **Stores scoped to Jenkins**, clique em **System**.
4. Em **System**, clique em **Global credentials (unrestricted)**.
5. No menu esquerdo, clique em **Add Credentials** *(ou no link **add some credentials**, se ainda não houver nenhuma)*.
6. Preencha os campos:
   - **Kind:** `Secret text`
   - **Scope:** `Global` *(disponibiliza a credencial para projetos/itens Pipeline)*
   - **Secret:** `segredo-teste-123` *(valor de exemplo para o laboratório)*
   - **ID:** `demo-api-token` *(opcional; atenção: depois de salvo, o ID não pode ser alterado)*
   - **Description:** descrição opcional (ex.: `Token de teste da Aula 2`)
7. Clique em **OK** *(ou no botão de criação/salvamento exibido)* para gravar.

---

## 4. Criando e Executando o Pipeline (`aula-02-fundamentos`)

### Passo A — Criar o item
1. No **Dashboard**, clique em **+ New Item** (menu lateral esquerdo) ou **Create a job** (quando não há jobs).
2. Em **Enter an item name**, digite **`aula-02-fundamentos`** (sem espaços).
3. Role a lista, selecione **Pipeline** e clique em **OK**.

### Passo B — Configurar o Pipeline pela UI (*Through the classic UI*)
1. Na página de configuração que abrir, clique na aba **Pipeline** no painel lateral esquerdo (ela leva diretamente à seção **Pipeline**).
2. Em **Definition**, mantenha selecionado **Pipeline script**.
3. Cole o script abaixo na área **Script**:

```groovy
pipeline {
    agent any

    environment {
        // Injeta a credencial Secret text na variável de ambiente API_TOKEN
        API_TOKEN = credentials('demo-api-token')
    }

    stages {
        stage('Inspecionar Ambiente') {
            steps {
                sh 'echo "Build #${BUILD_NUMBER} rodando no workspace ${WORKSPACE}"'
            }
        }
        stage('Validar Credencial Segura') {
            steps {
                // Use aspas simples ('') no sh para que o shell expanda $API_TOKEN
                // e o Jenkins mascare o valor nos logs (****). Evite aspas duplas ("") do Groovy.
                sh '''
                  if [ -n "$API_TOKEN" ]; then
                    echo "Credencial carregada com sucesso: $API_TOKEN"
                  fi
                '''
            }
        }
    }
}
```

4. Clique em **Save**.

> **Nota sobre uso real (In SCM):** para projetos reais, a prática recomendada é versionar esse conteúdo em um arquivo `Jenkinsfile` na raiz do repositório Git e selecionar **Pipeline script from SCM** no campo **Definition**.

### Passo C — Executar e inspecionar os logs
1. Na página do projeto `aula-02-fundamentos`, clique em **Build Now** no menu esquerdo *(ou no ícone verde `▷` na linha do projeto no Dashboard)*.
2. Em **Build History** à esquerda, clique em **`#1`**.
3. No menu esquerdo do build `#1`, clique em **Console Output** *(atalho alternativo: no Dashboard, clique no globo colorido à esquerda de `#1`)*.
4. Confirme que a impressão de `$API_TOKEN` aparece mascarada como **`****`** no log.

---

## 5. Recursos Nativos que Ajudam no Dia a Dia

A documentação integrada do Pipeline fica acessível pelo link **Pipeline Syntax** no menu lateral de qualquer projeto Pipeline já configurado, ou globalmente em `http://localhost:8080/pipeline-syntax`:

- **Snippet Generator:** selecione o step em **Sample Step**, preencha os campos e clique em **Generate Pipeline Script** para gerar o código Groovy pronto (inclusive `withCredentials` para chaves SSH ou certificados).
- **Declarative Directive Generator:** gera diretivas estruturais do *Declarative Pipeline* (`agent`, `options`, `triggers`, `when`).
- **Global Variable Reference** (`/pipeline-syntax/globals`): lista variáveis globais dinâmicas disponíveis em execução (`env`, `params`, `currentBuild`).

---

## 6. Referência Rápida: Outros Tipos de Credencial em `Jenkinsfile`

- **`Username and password` via `credentials('id')`:** cria automaticamente 3 variáveis de ambiente:
  - `MINHA_CRED` (`usuario:senha`)
  - `MINHA_CRED_USR` (somente usuário)
  - `MINHA_CRED_PSW` (somente senha)
- **`SSH Username with private key` e `Certificate`:** não usam o helper `credentials()` em `environment`; usam o step `withCredentials(...)` dentro de `steps` (gere o bloco exato pelo **Snippet Generator** usando *withCredentials: Bind credentials to variables*).
