# Guia de Deploy com ArgoCD - Conversão de Distância

## Visão Geral

Esta documentação descreve como configurar e fazer deploy da aplicação **Conversão de Distância** usando ArgoCD seguindo as práticas de GitOps.

## Análise da Aplicação

### Tecnologia
- **Framework**: Flask (Python)
- **Servidor**: Gunicorn em produção
- **Container**: Docker com imagem baseada em Python 3.11-slim
- **Repositório**: jhowssilva/conversao-distancia no Docker Hub

### Funcionalidades
- Conversão entre diferentes unidades de distância
- Interface web responsiva
- API REST para conversões
- Health checks configurados

## Estrutura do Projeto

```
conversao-distancia/
├── argocd/                     # Configurações ArgoCD
│   ├── project.yaml           # AppProject
│   ├── application.yaml       # Application simples
│   ├── application-production.yaml
│   └── application-staging.yaml
├── k8s/                       # Manifests Kubernetes
│   ├── base/                  # Recursos base (Kustomize)
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml
│   │   └── kustomization.yaml
│   ├── overlays/              # Overlays por ambiente
│   │   ├── production/
│   │   │   ├── kustomization.yaml
│   │   │   └── resource-patch.yaml
│   │   └── staging/
│   │       ├── kustomization.yaml
│   │       └── ingress-patch.yaml
│   └── deployment.yaml        # Manifest original (legacy)
├── src/                       # Código da aplicação
└── README.md
```

## Pré-requisitos

1. **Cluster Kubernetes** funcionando
2. **ArgoCD** instalado no cluster
3. **Nginx Ingress Controller** (opcional, para acesso externo)
4. **Cert-Manager** (opcional, para HTTPS automático)
5. Acesso ao repositório Git

## Instalação do ArgoCD

Se ainda não tiver o ArgoCD instalado:

```bash
# Criar namespace
kubectl create namespace argocd

# Instalar ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Aguardar pods ficarem prontos
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Obter senha inicial do admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward para acessar UI (opcional)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Configuração do ArgoCD

### 1. Criar o AppProject

```bash
kubectl apply -f argocd/project.yaml
```

### 2. Deploy da Aplicação

Você pode escolher entre três abordagens:

#### Opção A: Application Simples
```bash
kubectl apply -f argocd/application.yaml
```

#### Opção B: Ambientes Separados (Recomendado)
```bash
# Production
kubectl apply -f argocd/application-production.yaml

# Staging
kubectl apply -f argocd/application-staging.yaml
```

#### Opção C: Via ArgoCD CLI
```bash
# Login no ArgoCD
argocd login <ARGOCD_SERVER>

# Criar application
argocd app create conversao-distancia \
  --repo https://github.com/JhowSSilva/conversao-distancia.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

## Configurações de Ambiente

### Staging
- **Namespace**: staging
- **Replicas**: 1
- **Image Tag**: latest
- **Branch**: develop
- **Host**: staging.conversao-distancia.example.com

### Production
- **Namespace**: production
- **Replicas**: 3
- **Image Tag**: v1.0.0
- **Branch**: main
- **Host**: conversao-distancia.example.com
- **Recursos**: 256Mi/200m CPU (requests), 512Mi/1000m CPU (limits)

## Monitoramento

### Verificar Status da Aplicação
```bash
# Via kubectl
kubectl get applications -n argocd

# Via ArgoCD CLI
argocd app list
argocd app get conversao-distancia-production
```

### Verificar Recursos no Cluster
```bash
# Production
kubectl get all -n production -l app=conversao-distancia

# Staging
kubectl get all -n staging -l app=conversao-distancia
```

### Logs da Aplicação
```bash
kubectl logs -n production deployment/conversao-distancia -f
```

## Operações Comuns

### Forçar Sincronização
```bash
argocd app sync conversao-distancia-production
```

### Atualizar Imagem
1. Atualizar tag no arquivo `k8s/base/kustomization.yaml`
2. Fazer commit e push
3. ArgoCD sincronizará automaticamente

### Rollback
```bash
argocd app rollback conversao-distancia-production <REVISION>
```

### Pausar Sincronização Automática
```bash
argocd app set conversao-distancia-production --sync-policy none
```

## Solução de Problemas

### Application Não Sincroniza
1. Verificar se o repositório está acessível
2. Verificar logs do ArgoCD:
   ```bash
   kubectl logs -n argocd deployment/argocd-application-controller
   ```

### Pods Não Startam
1. Verificar se a imagem existe no registry
2. Verificar recursos disponíveis no cluster
3. Verificar logs do pod:
   ```bash
   kubectl logs -n production <pod-name>
   ```

### Ingress Não Funciona
1. Verificar se o Nginx Ingress Controller está instalado
2. Verificar DNS do domínio
3. Verificar certificados TLS

## Melhorias Futuras

1. **Implementar Health Checks** mais específicos
2. **Configurar Prometheus/Grafana** para monitoramento
3. **Implementar Secrets Management** com Sealed Secrets ou External Secrets
4. **Configurar Image Updater** para atualizações automáticas de imagem
5. **Implementar Multi-cluster** deployment

## Segurança

- Imagem roda com usuário não-root (UID 1000)
- ReadOnlyRootFilesystem ativado
- Capabilities desnecessários removidos
- Recursos limitados
- TLS configurado para Ingress

## Links Úteis

- [Documentação ArgoCD](https://argo-cd.readthedocs.io/)
- [Kustomize](https://kustomize.io/)
- [Repositório da Aplicação](https://github.com/JhowSSilva/conversao-distancia)