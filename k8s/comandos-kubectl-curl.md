# Guia rapido de exec, curl, IP e logs (sem Service) - WSL/Ubuntu

Este guia usa os deployments:
- alvo-demo
- cliente-curl-demo

Se voce estiver em outro namespace, adicione `-n <namespace>` em todos os comandos.

## 1) Conferir pods e status

```bash
kubectl get pods -l app=alvo-demo -o wide
kubectl get pods -l app=cliente-curl-demo -o wide
```

## 2) Descobrir nome e IP dos pods

```bash
ALVO_POD=$(kubectl get pod -l app=alvo-demo -o jsonpath='{.items[0].metadata.name}')
CLIENTE_POD=$(kubectl get pod -l app=cliente-curl-demo -o jsonpath='{.items[0].metadata.name}')

ALVO_IP=$(kubectl get pod "$ALVO_POD" -o jsonpath='{.status.podIP}')
CLIENTE_IP=$(kubectl get pod "$CLIENTE_POD" -o jsonpath='{.status.podIP}')

echo "ALVO_POD:    $ALVO_POD"
echo "ALVO_IP:     $ALVO_IP"
echo "CLIENTE_POD: $CLIENTE_POD"
echo "CLIENTE_IP:  $CLIENTE_IP"
```

## 3) Entrar no pod cliente com kubectl exec e fazer curl

```bash
kubectl exec -it deploy/cliente-curl-demo -- sh
```

Dentro do container:

```sh
curl -sS -i http://<ALVO_IP>
exit
```

## 4) Fazer curl sem entrar no pod (usando so kubectl)

```bash
ALVO_IP=$(kubectl get pod -l app=alvo-demo -o jsonpath='{.items[0].status.podIP}')
kubectl exec deploy/cliente-curl-demo -- curl -sS -i "http://$ALVO_IP"
```

## 5) Mostrar IP do cliente direto no terminal (via kubectl exec)

```bash
kubectl exec deploy/cliente-curl-demo -- hostname -i
```

Opcional: mostrar IP do pod alvo via exec no container web:

```bash
kubectl exec deploy/alvo-demo -c web -- hostname -i
```

## 6) Ver logs com kubectl logs

Logs do deployment cliente:

```bash
kubectl logs deploy/cliente-curl-demo
```

Logs em tempo real do cliente:

```bash
kubectl logs -f deploy/cliente-curl-demo
```

Logs do deployment alvo (container web):

```bash
kubectl logs deploy/alvo-demo -c web
```

Logs do sidecar no alvo:

```bash
kubectl logs deploy/alvo-demo -c sidecar
```

## 7) Teste rapido fim a fim

```bash
ALVO_IP=$(kubectl get pod -l app=alvo-demo -o jsonpath='{.items[0].status.podIP}')
kubectl exec deploy/cliente-curl-demo -- curl -sS "http://$ALVO_IP" | head -n 5
```

Se estiver OK, voce deve ver o HTML padrao do Nginx.
