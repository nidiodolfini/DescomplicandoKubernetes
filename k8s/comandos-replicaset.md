# Guia rápido: ReplicaSet (Day 4) – Descomplicando o Kubernetes

Este guia acompanha o [Day 4](https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day-4/README.md) e cobre:
- relação **Deployment → ReplicaSet → Pod**
- listar e inspecionar ReplicaSets
- escalar réplicas (scale) e rollout
- criar e remover ReplicaSet direto (sem Deployment)
- rollback

Se estiver em outro namespace, use `-n <namespace>` nos comandos.

---

## 1) Deployment cria ReplicaSet; ReplicaSet cria os Pods

Criar um Deployment (ex.: nginx) e ver que um ReplicaSet foi criado:

```bash
kubectl apply -f k8s/deployment-alvo.yaml
# ou um deployment nginx como no Day 4:
# kubectl apply -f nginx-deployment.yaml

kubectl get deployments
kubectl get replicasets
kubectl get pods
```

O ReplicaSet aparece com nome `<deployment>-<hash>` (ex.: `nginx-deployment-6dd8d7cfbd`).

---

## 2) Ver qual ReplicaSet o Deployment está usando

```bash
kubectl describe deployment alvo-demo
# ou: kubectl describe deployment nginx-deployment
```

Na saída, procure por `NewReplicaSet:` — é o ReplicaSet que está gerenciando as réplicas atuais.

---

## 3) Aumentar/diminuir réplicas (scale)

**Opção A – por comando (não fica no Git):**

```bash
kubectl scale deployment alvo-demo --replicas=3
kubectl get replicasets
kubectl get pods -o wide
```

**Opção B – editando o manifesto e aplicando (recomendado, versionado):**

Altere `spec.replicas` no YAML do Deployment e depois:

```bash
kubectl apply -f k8s/deployment-alvo.yaml
kubectl get replicasets
```

Ao mudar só o número de réplicas, o **mesmo** ReplicaSet continua; apenas DESIRED/CURRENT/READY mudam.

---

## 4) Atualizar imagem (novo ReplicaSet + rollout)

Ao mudar a imagem (ou o template do Pod) no Deployment e aplicar, o Kubernetes cria um **novo** ReplicaSet e faz o rollout. O ReplicaSet antigo fica com 0 réplicas (guardado para rollback).

```bash
kubectl apply -f k8s/deployment-alvo.yaml
kubectl get replicasets
kubectl rollout status deployment/alvo-demo
```

Você verá dois ReplicaSets: o novo com réplicas, o antigo com 0.

---

## 5) Rollback para a revisão anterior

```bash
kubectl rollout undo deployment/alvo-demo
kubectl get replicasets
kubectl describe deployment alvo-demo
```

O Deployment volta a usar o ReplicaSet anterior; o que estava “novo” passa a 0 réplicas.

---

## 6) Criar um ReplicaSet direto (sem Deployment)

Não é boa prática (sem rollout/rollback), mas o Day 4 mostra o exemplo. Ex.: `nginx-replicaset.yaml` com `kind: ReplicaSet` e `selector.matchLabels` + `template` iguais.

```bash
kubectl apply -f nginx-replicaset.yaml
kubectl get replicasets
kubectl get pods -l app=nginx-app
```

Se alterar a imagem no YAML e der `apply`, o ReplicaSet **não** recria os Pods automaticamente; só novos Pods (ou após deletar um Pod) virão com a nova imagem.

Deletar um Pod para o ReplicaSet criar um novo (ex.: com nova imagem):

```bash
kubectl delete pod <nome-do-pod>
kubectl get pods -l app=nginx-app
```

Listar Pods e imagens (JSONPath):

```bash
kubectl get pods -o=jsonpath='{range .items[*]}{"\n"}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{"\t"}{end}{end}'
```

---

## 7) Remover ReplicaSet (e seus Pods)

**Por nome:**

```bash
kubectl delete replicaset nginx-replicaset
```

**Por manifesto:**

```bash
kubectl delete -f nginx-replicaset.yaml
```

---

## 8) Resumo rápido

| Objetivo              | Comando |
|-----------------------|--------|
| Listar ReplicaSets    | `kubectl get replicasets` |
| Detalhes do RS        | `kubectl describe replicaset <nome>` |
| Scale (Deployment)    | `kubectl scale deployment <nome> --replicas=N` |
| Status do rollout     | `kubectl rollout status deployment/<nome>` |
| Rollback              | `kubectl rollout undo deployment/<nome>` |
| Deletar ReplicaSet    | `kubectl delete replicaset <nome>` |

---

## Lição de casa (Day 4)

- Replicar os exemplos do Day 4 (Deployment → ReplicaSet, scale, troca de imagem, rollback).
- Em tudo que criar daqui pra frente: usar **probes** (liveness, readiness, startup) e **limites de recursos** nos Pods.

Referência: [Day 4 – A sua lição de casa](https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day-4/README.md#a-sua-licao-de-casa)
