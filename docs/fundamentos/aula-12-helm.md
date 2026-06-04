---
title: "Aula 12 — Helm"
parent: Fundamentos
nav_order: 13
---

# Aula 12 — Helm

## O que é o Helm

Helm é o gerenciador de pacotes do Kubernetes — equivalente ao apt, npm ou brew, mas para recursos K8s.

Em vez de escrever na mão múltiplos YAMLs para subir uma aplicação, você usa um **chart** pronto:

```bash
helm install meu-postgres bitnami/postgresql
```

---

## O que é um chart

Um chart é uma pasta com templates de YAML + um arquivo de valores. Nenhuma mágica:

```
meu-chart/
├── Chart.yaml          # metadados: nome, versão, descrição
├── values.yaml         # valores padrão — é aqui que você customiza
└── templates/          # YAMLs com variáveis no lugar de valores fixos
    ├── deployment.yaml
    ├── service.yaml
    └── secret.yaml
```

Quando você roda `helm install`, o Helm:
1. Pega os templates
2. Substitui as variáveis pelos valores do `values.yaml`
3. Aplica os YAMLs resultantes no cluster

---

## Comandos essenciais

### Repositórios

```bash
# Adicionar repositório
helm repo add bitnami https://charts.bitnami.com/bitnami

# Atualizar índice de charts disponíveis
helm repo update
```

### Inspecionar antes de instalar

```bash
# Ver todos os valores configuráveis do chart
helm show values bitnami/nginx

# Gerar os YAMLs sem aplicar nada — desmistifica a "mágica"
helm template meu-nginx bitnami/nginx
```

O `helm template` é o comando mais importante para entender o que um chart faz: prova que no final é só YAML sendo gerado.

### Instalar e gerenciar

```bash
# Instalar
helm install meu-nginx bitnami/nginx --namespace laboratorio

# Atualizar configuração
helm upgrade meu-nginx bitnami/nginx --namespace laboratorio --set replicaCount=2

# Ver releases instaladas
helm list -n laboratorio

# Histórico de revisões
helm history meu-nginx -n laboratorio

# Rollback para uma revisão anterior
helm rollback meu-nginx 1 -n laboratorio

# Remover tudo que a release criou
helm uninstall meu-nginx -n laboratorio
```

---

## Revisões e rollback

Cada `install` ou `upgrade` cria uma nova revisão. O Helm preserva o histórico:

```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         deployed    Rollback to 1
```

O rollback cria uma **nova revisão** que representa o estado anterior — o histórico nunca é apagado.

---

## Como o Helm rastreia o que criou

Todos os recursos criados pelo Helm recebem labels automáticos:

```yaml
app.kubernetes.io/managed-by: Helm
helm.sh/chart: nginx-25.0.0
```

O `helm uninstall` usa esses labels para localizar e deletar todos os recursos da release de uma vez.

---

## Customizando com values.yaml

Em vez de passar tudo via `--set`, o jeito correto em projetos reais:

```bash
# Salva todos os valores padrão do chart
helm show values bitnami/nginx > values-nginx.yaml

# Edita só o que precisa e instala apontando para o arquivo
helm install meu-nginx bitnami/nginx -f values-nginx.yaml -n laboratorio
```

### Múltiplos ambientes

O mesmo chart com valores diferentes por ambiente:

```bash
helm install meu-app ./meu-app -f values-dev.yaml
helm install meu-app ./meu-app -f values-prod.yaml
```

---

## Empacotando sua própria infra

Você pode criar um chart com tudo que sua aplicação precisa e replicar em qualquer cluster com um único comando:

```
meu-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    └── secret.yaml
```

```bash
# Em qualquer cluster
helm install meu-app ./meu-app
```

---

## Referência rápida

| Comando | O que faz |
|---|---|
| `helm repo add` | Adiciona repositório de charts |
| `helm repo update` | Atualiza índice de charts |
| `helm show values` | Mostra valores configuráveis |
| `helm template` | Gera YAMLs sem aplicar |
| `helm install` | Instala uma release |
| `helm upgrade` | Atualiza uma release |
| `helm rollback` | Volta para revisão anterior |
| `helm history` | Histórico de revisões |
| `helm list` | Lista releases instaladas |
| `helm uninstall` | Remove tudo da release |
