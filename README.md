# 🚀 CI/CD com FastAPI + Docker Hub + GitHub Actions + Argo CD (Rancher Desktop)

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
  - Repositórios: `projeto-uol-api` e `projeto-kubernetes-deployments`
- 🐋 **Conta no Docker Hub**
  - Repositório público: `otaviolxna/hello-app`
- 🔧 **Argo CD** instalado no cluster

---

## 📁 Estrutura dos repositórios

| Repositório | Função |
|--------------|--------|
| **projeto-uol-api** | Contém a API FastAPI, Dockerfile e workflow do GitHub Actions |
| **projeto-kubernetes-deployments** | Contém os manifests Kubernetes (`deployment.yaml`, `service.yaml`) observados pelo Argo CD |

---

## 🧩 1. Código da API

**`main.py`**
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello, Otávio! 🚀"}
```

**`requirements.txt`**
```
fastapi
uvicorn
```

**`Dockerfile`**
```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

> Teste local:
> ```bash
> docker build -t hello-app .
> docker run -p 8082:8080 hello-app
> # → http://localhost:8082
> ```

---

## ⚙️ 2. Manifests do Kubernetes

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
    - port: 8080
      targetPort: 8080
      protocol: TCP
  type: ClusterIP
```

---

## 🐋 3. Publicar no Docker Hub

Crie um repositório público no Docker Hub:  
➡️ `otaviolxna/hello-app`

As tags geradas:
- `latest`
- `<sha-curto>` (ex: `a12bc34d5f6`)

---

## ⚙️ 4. GitHub Actions (CI/CD)

### 🔐 Secrets necessários (no `projeto-uol-api`)
```
DOCKER_USERNAME = otaviolxna
DOCKER_PASSWORD = <token do Docker Hub>
GH_PAT          = <Personal Access Token com repo:contents write>
```

---

## 🧭 5. Instalar o Argo CD no Rancher Desktop

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

### Acessar o painel
```bash
kubectl port-forward -n argocd svc/argocd-server 9090:443
```
Acesse em: [http://localhost:9090](http://localhost:9090)

Login: Admin

Senha: (Rodar no PowerShell)
```powershell
kubectl -n argocd get secret argocd-initial-admin-secret ` -o jsonpath="{.data.password}" | %{ [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

---

## 🌐 6. Criar o App no Argo CD

- **Application Name:** `hello-app`
- **Project:** `default`
- **Repository URL:** `https://github.com/otaviolxna/projeto-kubernetes-deployments`
- **Revision:** `main`
- **Path:** `/`
- **Cluster:** `in-cluster`
- **Namespace:** `default`
- **Sync Policy:** ✅ Enable Auto-Sync

> 💡 Em ambiente local, o auto-sync depende de polling.
> 
---

## 🌍 7. Acesso à aplicação

**Via Port-Forward:**
```bash
kubectl port-forward -n default svc/hello-app 8081:8080
# Acesse em http://localhost:8081
```

---

## 🔄 8. Fluxo de Atualização

1. Edite a mensagem no `projeto-uol-api`  
2. GitHub Actions builda e publica a imagem  
3. Faz commit no `projeto-kubernetes-deployments`  
4. Argo CD detecta → sincroniza → cria novo pod  
5. Atualização visível na URL da API 🎉

---

## 🏁 Conclusão

Com esse projeto, você aprendeu a:
- Criar pipelines CI/CD reais com **GitHub Actions**
- Publicar imagens Docker automaticamente
- Aplicar o modelo **GitOps** com **Argo CD**
- Entregar uma API **FastAPI** totalmente automatizada 🎯

---
