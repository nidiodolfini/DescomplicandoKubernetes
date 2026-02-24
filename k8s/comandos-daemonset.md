# Guia rápido: DaemonSet (Day 4) – Descomplicando o Kubernetes

Este guia acompanha o [Day 4](https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day-4/README.md) e cobre:
- **DaemonSet** – garante **uma réplica do Pod em cada nó** do cluster (agentes de monitoramento, node-exporter, proxy de rede, segurança)
- criar, listar e inspecionar DaemonSet
- ver Pods por nó; adicionar/remover nós e o que acontece
- remover DaemonSet

Se estiver em outro namespace, use `-n <namespace>` nos comandos.

---

## 1) Aplicar o DaemonSet (node-exporter)

Um DaemonSet cria um Pod por nó. Exemplo com Node Exporter (métricas por nó):

```bash
kubectl apply -f k8s/deployment-daemonset.yml
kubectl get daemonset
kubectl get pods -l app=node-exporter
```

A saída do `get daemonset` mostra DESIRED/CURRENT/READY/UP-TO-DATE/AVAILABLE (um por nó).

---

## 2) Ver em qual nó cada Pod está rodando

```bash
kubectl get pods -l app=node-exporter -o wide
```

A coluna `NODE` mostra o nó de cada Pod. Deve haver **um Pod por nó** (exceto master/control-plane se tiver taints que o DaemonSet não tolera).

---

## 3) Detalhes do DaemonSet (describe)

```bash
kubectl describe daemonset node-exporter
```

Na saída aparecem, entre outros:
- **Desired Number of Nodes Scheduled** – quantos nós devem ter o Pod
- **Current Number of Nodes Scheduled** – em quantos já foi agendado
- **Number of Nodes Misscheduled** – quantos Pods estão em nós que não deveriam
- **Pod Template** – imagem, portas, volumes (ex.: hostPath /proc, /sys)

---

## 4) Aumentar nós no cluster (novo nó → novo Pod)

Ao adicionar um nó ao cluster, o DaemonSet **cria automaticamente** um Pod nesse nó.

1. Aumente o número de nós (ex.: com `kind`, `kubeadm`, `eksctl`, etc.). Exemplo genérico:

```bash
# Exemplo EKS (ajuste cluster e nodegroup):
# eksctl scale nodegroup --cluster=eks-cluster --nodes 3 --name eks-cluster-nodegroup
kubectl get nodes
```

2. Depois que o novo nó estiver Ready, verifique o DaemonSet e os Pods:

```bash
kubectl get daemonset node-exporter
kubectl get pods -l app=node-exporter -o wide
```

Deve aparecer **um Pod a mais**, rodando no novo nó.

---

## 5) Diminuir nós no cluster (nó removido → Pod removido)

Ao remover um nó, o DaemonSet **remove** o Pod que estava naquele nó (não mantém “réplicas extras” em outros nós).

```bash
# Remover o nó pelo seu gerenciador (eksctl, kind, etc.), depois:
kubectl get nodes
kubectl get daemonset node-exporter
kubectl get pods -l app=node-exporter -o wide
```

O número de Pods deve bater com o número de nós agendáveis.

---

## 6) Remover o DaemonSet (e todos os Pods por nó)

**Por nome:**

```bash
kubectl delete daemonset node-exporter
```

**Por manifesto:**

```bash
kubectl delete -f k8s/deployment-daemonset.yml
```

Todos os Pods gerenciados pelo DaemonSet são removidos.

---

## 7) Criar manifesto com kubectl create (opcional)

Para gerar um YAML de exemplo (sem aplicar):

```bash
kubectl create daemonset node-exporter \
  --image=prom/node-exporter:latest \
  --port=9100 \
  --host-port=9100 \
  -o yaml --dry-run=client > node-exporter-daemonset.yaml
```

Ajuste o arquivo gerado (labels, volumes, etc.) e use `kubectl apply -f node-exporter-daemonset.yaml` se quiser.

---

## 8) Resumo rápido

| Objetivo              | Comando |
|-----------------------|--------|
| Listar DaemonSets      | `kubectl get daemonset` |
| Detalhes do DaemonSet  | `kubectl describe daemonset node-exporter` |
| Pods do DaemonSet      | `kubectl get pods -l app=node-exporter` |
| Pods por nó (-o wide)  | `kubectl get pods -l app=node-exporter -o wide` |
| Remover por nome       | `kubectl delete daemonset node-exporter` |
| Remover por manifesto  | `kubectl delete -f k8s/deployment-daemonset.yml` |

---

## Casos de uso (Day 4)

- Agentes de monitoramento: **Prometheus Node Exporter**, **Fluentd**
- Proxy/rede em todo nó: **kube-proxy**, **Weave Net**, **Calico**, **Flannel**
- Segurança por nó: **Falco**, **Sysdig**

---

## Lição de casa (Day 4)

- Replicar os exemplos do Day 4: criar DaemonSet, listar, adicionar/remover nó e ver o comportamento.
- DaemonSet não tem “réplicas” configuráveis: sempre **um Pod por nó** (respeitando selector e tolerations).

Referência: [Day 4 – O DaemonSet](https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day-4/README.md#o-daemonset) e [A sua lição de casa](https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day-4/README.md#a-sua-licao-de-casa).
