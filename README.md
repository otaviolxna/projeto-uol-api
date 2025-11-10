# 🚀 CI/CD com API + Docker + GitHub Actions + Argo CD

> **Objetivo:**  
> Automatizar **build**, **push** e **deploy** de uma API FastAPI em **Kubernetes local (Rancher Desktop)** usando **Docker Hub** como registry, **GitHub Actions** para CI/CD e **Argo CD** para entrega contínua (GitOps).

## ⚙️ Tecnologias

<div align="center">

  <!-- Linguagem & Framework -->
  <img alt="Python" src="https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white&style=for-the-badge" />
  
  <!-- Container & Registry -->
  <img alt="Docker" src="https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white&style=for-the-badge" />
  <img alt="Docker Hub" src="https://img.shields.io/badge/Docker%20Hub-2496ED?logo=docker&logoColor=white&style=for-the-badge" />

  <!-- CI/CD -->
  <img alt="GitHub Actions" src="https://img.shields.io/badge/GitHub%20Actions-2088FF?logo=githubactions&logoColor=white&style=for-the-badge" />

  <!-- Orquestração & GitOps -->
  <img alt="Kubernetes" src="https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white&style=for-the-badge" />
  <img alt="Argo CD" src="https://img.shields.io/badge/Argo%20CD-FB6A00?logo=argo&logoColor=white&style=for-the-badge" />
  <img alt="Rancher Desktop" src="https://img.shields.io/badge/Rancher%20Desktop-0075A8?logo=rancher&logoColor=white&style=for-the-badge" />

</div>

---

## ✅ Pré-requisitos

- 🧠 **Windows 10/11** com **PowerShell**
- 🐳 **Rancher Desktop** com Kubernetes habilitado (`v1.28+`)
  ```bash
  kubectl config use-context rancher-desktop
  kubectl get nodes
  ```
- ☁️ **Conta no GitHub**
  - Repositórios: `repositorio da api` e `repositorio dos manifests`
- 🐋 **Conta no Docker Hub**
  - Repositório público: `seunome/hello-app`
- 🔧 **Argo CD** instalado no cluster
- 📂 **Rancher Desktop** para utilização dos Kubernetes 

---

## 📁 Estrutura dos repositórios

| Repositório | Função |
|--------------|--------|
| **projeto-uol-api** | Contém a API FastAPI, Dockerfile e workflow do GitHub Actions |
| **projeto-kubernetes-deployments** | Contém os manifests Kubernetes (`deployment.yaml`, `service.yaml`) observados pelo Argo CD |

---

## 🧩 1. Repósitorio API e Workflows

1. Crie um repositório público

![Criação do primeiro repositório](images/repositorio1.png)

2. Adicione os seguintes arquivos:

**`main.py`**
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello World!"}
```

![Criar arquivo da API](images/api.png)

**`Dockerfile`**
```dockerfile
FROM python:3.10-slim

WORKDIR /app
COPY . /app

RUN pip install fastapi uvicorn

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

![Criar arquivo Dockerfile](images/dockerfile.png)

No repositório da **`api`**, crie o caminho:

.github/workflows/ci-cd.yml

**`ci-cd.yml`**
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    env:
      IMAGE_NAME: ${{ secrets.DOCKER_USERNAME }}/hello-app
      MANIFESTS_REPO_SSH: git@github.com:otaviolxna/projeto-kubernetes-deployments.git
      MANIFESTS_BRANCH: main
      # se seus YAMLs estiverem em subpasta (ex.: k8s), mude para "k8s"
      MANIFESTS_PATH: .

    steps:
      - name: Checkout app repo (projeto-uol-api)
        uses: actions/checkout@v4

      - name: Set image tag (short SHA)
        id: vars
        run: echo "TAG=$(echo $GITHUB_SHA | cut -c1-12)" >> $GITHUB_OUTPUT

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build & Push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ steps.vars.outputs.TAG }}

      # --- SSH para acessar o repo de manifests ---
      - name: Configure SSH for git
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan github.com >> ~/.ssh/known_hosts

      - name: Clone manifests repo (projeto-kubernetes-deployments)
        run: |
          set -euxo pipefail
          git clone --depth 1 -b "${{ env.MANIFESTS_BRANCH }}" "${{ env.MANIFESTS_REPO_SSH }}" manifests
          cd manifests
          git config user.name "github-actions"
          git config user.email "ci@github.com"

          # entra na pasta onde estão os YAMLs
          cd "${{ env.MANIFESTS_PATH }}"

          # Atualiza a imagem no deployment.yaml (ajuste o nome do arquivo se precisar)
          if [ -f deployment.yaml ]; then
            sed -i -E "s|image: .*hello-app.*|image: ${{ env.IMAGE_NAME }}:${{ steps.vars.outputs.TAG }}|" deployment.yaml
          else
            # fallback: tenta achar qualquer deployment*.yaml contendo 'hello-app'
            files=$(grep -ril --include="*deployment*.yaml" -e 'hello-app' . || true)
            for f in $files; do
              sed -i -E "s|image: .*hello-app.*|image: ${{ env.IMAGE_NAME }}:${{ steps.vars.outputs.TAG }}|" "$f"
            done
          fi

          # Só commita se houve alteração
          if git status --porcelain | grep .; then
            git add -A
            git commit -m "chore(ci): bump image to ${{ env.IMAGE_NAME }}:${{ steps.vars.outputs.TAG }}"
            git push origin "${{ env.MANIFESTS_BRANCH }}"
          else
            echo "Nenhuma mudança nos manifests (imagem já estava nessa tag)."
          fi
```

![Criar arquivo Workflow](images/workflow.png)

---

## ⚙️ 2. Repositório Manifests do Kubernetes

1. Crie um novo repositório público para os manifests do Kubernetes:

![Criação do segundo repositório](images/repositorio2.png)

2. 
**`deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  annotations:
    ci.lastUpdate: "manual"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
      annotations:
        ci.lastUpdate: "manual"
    spec:
      containers:
        - name: hello-app
          image: otaviolxna/hello-app:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
```

![Criação do arquivo deployment](images/deployment.png)

**`service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-app
spec:
  selector:
    app: hello-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

![Criação do arquivo service](images/service.png)

---

## 🐋 3. Publicar no Docker Hub

Crie um repositório público no Docker Hub: 

Exemplo:
➡️ `otaviolxna/hello-app`

As tags geradas:
- `latest`
- `<sha-curto>` (ex: `a12bc34d5f6`)

![Criação do repositório no Docker Hub](images/dockerhub.png)

---

## ⚙️ 4. GitHub Actions (CI/CD)

## 🔐 4. Configurando os Secrets do GitHub Actions

Para que o pipeline consiga:
- Fazer **login no Docker Hub** (para publicar as imagens);
- Ter permissão de **editar o repositório de manifests** (para atualizar o `deployment.yaml`);
- E autenticar com segurança, **sem expor senhas no código**;

...é necessário configurar **3 secrets** no repositório da **api`**.

---

### 🧭 Passo a passo

1. **Acesse o repositório da API no GitHub**  

2. Vá até:  
   **Settings → Secrets and variables → Actions → New repository secret**

4. Agora você criará **três secrets** (um de cada vez):

---

### 🧩 Secret 1 — `DOCKER_USERNAME`

```
- **Valor:** seu nome de usuário do Docker Hub  
  Exemplo:
  otaviolxna
```

- **Função:** permite que o GitHub Actions saiba **qual conta Docker** usar para enviar imagens (`docker push`).

---

### 🧩 Secret 2 — `DOCKER_PASSWORD`
```
- **Valor:** um **Access Token** do Docker Hub (não a senha da conta).  
Para gerar:
1. Acesse [hub.docker.com](https://hub.docker.com)
2. Vá em **Account Settings → Security → New Access Token**
3. Dê um nome (ex: `projeto uol`)
```

![Criação do Token](images/token.png)

```
4. Copie o token gerado
5. Cole no campo de valor do secret no GitHub
```

⚠️ **Importante:** esse token é exibido apenas uma vez — se perder, gere outro.

---

### 🧩 Secret 3 — `SSH_PRIVATE_KEY`

#### 1) Gerar um par de chaves SSH (ed25519)

```bash
ssh-keygen -t ed25519 -C "ci@github-actions" -f ./id_ed25519
```

![Criação das Chaves SSH](images/chaves.png)

Isso vai gerar:

Chave privada: id_ed25519

Chave pública: id_ed25519.pub

⚠️ Nunca faça commit da chave privada em repositórios.

---

### 2) Adicionar a chave pública como Deploy Key (com write)

No GitHub, abra o repositório dos manifests do Kubernetes →
Settings → Deploy keys → Add deploy key

Title: GitHub Actions

Key: (cole o conteúdo completo de id_ed25519.pub)

✅ Allow write access (marque essa opção)

Add key

![Deploy Key](images/deploykey.png)

---

1. Agora adicione a chave privada como secret do repositório da API
2. 
**Valor:** SSH_PRIVATE_KEY

Conteúdo:

```
Conteúdo da chave privada
```

---

Ao realizar todos os passos, sua aba Actions Secrets ficará assim:

![Actions Secrets](images/secrets.png)

---

## 🧭 5. Instalar o Argo CD no Rancher Desktop

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

![ArgoCD](images/argo.png)

### Acessar o painel
```bash
kubectl port-forward -n argocd svc/argocd-server 9090:443
```

![Iniciar ArgoCD](images/interface.png)

Acesse em: [http://localhost:9090](http://localhost:9090)

Login: Admin

![Login](images/login.png)

Senha: (Rodar no PowerShell)
```powershell
kubectl -n argocd get secret argocd-initial-admin-secret ` -o jsonpath="{.data.password}" | %{ [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

![Senha](images/senha.png)

---

## 🌐 6. Criar o App no Argo CD

- **Application Name:** `GitHub Actions`
- **Project:** `default`
- **Sync Policy:** ✅ Enable Auto-Sync

![Criação do APP](images/app1.png)

- **Repository URL:** `https://github.com/otaviolxna/projeto-kubernetes-deployments`
- **Revision:** `main`
- **Path:** `.`

![Criação do APP](images/app2.png)

- **Cluster:** `in-cluster`
- **Namespace:** `default`

![Criação do APP](images/app3.png)

> 💡 Em ambiente local, o auto-sync depende de polling.

---

## 🌍 7. Acesso à aplicação

**Via Port-Forward:**
```bash
kubectl port-forward -n default svc/hello-app 8080:8080
# Acesse em http://localhost:8080
```

![API no Ar](images/site.png)

---

## 🔄 8. Fluxo de Atualização

1. Edite a mensagem no `projeto-uol-api`  
2. GitHub Actions builda e publica a imagem  
3. Faz commit no `projeto-kubernetes-deployments`  
4. Argo CD detecta → sincroniza → cria novo pod  
5. Atualização visível na URL da API 🎉

![Resultado](images/resultado.png)

---

## 🏁 Conclusão

Com esse projeto, você aprendeu a:
- Criar pipelines CI/CD reais com **GitHub Actions**
- Publicar imagens Docker automaticamente
- Aplicar o modelo **GitOps** com **Argo CD**
- Entregar uma API **FastAPI** totalmente automatizada 🎯

---

## ✨ Sobre mim

Olá! 👋 Sou **Otávio Lana**, estudante de **Segurança da Informação** e entusiasta de **DevSecOps** e **Cloud Security**.  
Atualmente sou estagiário na **UOL Compass**, com foco em **AWS, automação e práticas de FinOps**.  
Meu objetivo é crescer na área de segurança em nuvem e levar meus projetos para o nível internacional 🌎.

Gosto de transformar aprendizado em prática — seja construindo pipelines, automatizando deploys ou criando conteúdos sobre cibersegurança e tecnologia.  
Se você curtiu este projeto, sinta-se à vontade para **⭐ dar uma estrela** e contribuir! 😄

💼 [LinkedIn](https://www.linkedin.com/in/otaviolxna) 

</div>
