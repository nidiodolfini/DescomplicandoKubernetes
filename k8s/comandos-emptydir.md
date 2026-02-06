# Guia rapido: emptyDir (acesso, arquivos e persistencia) - WSL/Ubuntu

Este guia usa o deployment `emptydir-demo` e mostra:
- como entrar no pod e visualizar arquivos
- como fazer comandos sem shell interativo
- como testar persistencia do `emptyDir`

Se voce estiver em outro namespace, adicione `-n <namespace>` em todos os comandos.

## 1) Aplicar e validar o deployment

```bash
kubectl apply -f k8s/deployment-emptydir.yaml
kubectl get pods -l app=emptydir-demo -o wide
```

## 2) Descobrir nome e IP do pod

```bash
POD=$(kubectl get pod -l app=emptydir-demo -o jsonpath='{.items[0].metadata.name}')
POD_IP=$(kubectl get pod "$POD" -o jsonpath='{.status.podIP}')

echo "POD: $POD"
echo "POD_IP: $POD_IP"
```

## 3) Acessar via exec e visualizar arquivos

Entrar no pod:

```bash
kubectl exec -it deploy/emptydir-demo -- sh
```

Dentro do pod:

```sh
echo "arquivo criado no emptyDir" > /dados/arquivo.txt
ls -la /dados
cat /dados/arquivo.txt
exit
```

## 4) Fazer sem shell interativo (so com kubectl)

Criar e listar arquivo direto do terminal:

```bash
kubectl exec deploy/emptydir-demo -- sh -c 'echo "teste via kubectl exec" > /dados/prova.txt && ls -la /dados && cat /dados/prova.txt'
```

## 5) Testar persistencia: restart de container (deve manter)

1. Guardar nome atual do pod e criar arquivo de teste:

```bash
POD_ANTES=$(kubectl get pod -l app=emptydir-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$POD_ANTES" -- sh -c 'echo "sobrevive ao restart do container" > /dados/persistencia.txt && ls -la /dados'
```

2. Matar PID 1 do container para forcar restart do container no mesmo pod:

```bash
kubectl exec "$POD_ANTES" -- sh -c 'kill 1'
sleep 5
```

3. Validar que o pod continua com mesmo nome e o arquivo ainda existe:

```bash
POD_DEPOIS_RESTART=$(kubectl get pod -l app=emptydir-demo -o jsonpath='{.items[0].metadata.name}')
echo "Mesmo pod apos restart do container? $([ "$POD_ANTES" = "$POD_DEPOIS_RESTART" ] && echo sim || echo nao)"
kubectl exec "$POD_DEPOIS_RESTART" -- sh -c 'ls -la /dados && cat /dados/persistencia.txt'
```

Esperado: o arquivo ainda existe, porque o pod nao foi recriado.

## 6) Testar persistencia: recriacao do pod (deve perder dados)

1. Deletar o pod para o Deployment criar outro:

```bash
kubectl delete pod "$POD_DEPOIS_RESTART"
kubectl wait --for=condition=ready pod -l app=emptydir-demo --timeout=90s
```

2. Comparar nome e testar se arquivo antigo sumiu:

```bash
POD_NOVO=$(kubectl get pod -l app=emptydir-demo -o jsonpath='{.items[0].metadata.name}')
echo "Pod antigo: $POD_DEPOIS_RESTART"
echo "Pod novo:   $POD_NOVO"

kubectl exec "$POD_NOVO" -- sh -c 'ls -la /dados; if [ -f /dados/persistencia.txt ]; then echo "arquivo existe"; else echo "arquivo nao existe"; fi'
```

Esperado: novo pod e `arquivo nao existe`.

## 7) Logs do deployment/pod

```bash
kubectl logs deploy/emptydir-demo
kubectl logs -f deploy/emptydir-demo
```

Se quiser logs por pod:

```bash
POD=$(kubectl get pod -l app=emptydir-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs "$POD"
```
