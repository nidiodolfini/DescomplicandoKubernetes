# Guia rápido: Probes (Day 4) – Descomplicando o Kubernetes

Este guia acompanha o [Day 4](https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day-4/README.md) e cobre:
- **Liveness Probe** – verifica se o container está vivo; falha → reinicia o container
- **Readiness Probe** – verifica se está pronto para receber tráfego; falha → sai do Service/Endpoint
- **Startup Probe** – verifica se o container iniciou; usada antes de liveness/readiness

Se estiver em outro namespace, use `-n <namespace>` nos comandos.

---

## 1) Aplicar deployment com Liveness Probe

Usando o manifesto com `livenessProbe` (tcpSocket na porta 80):

```bash
kubectl apply -f k8s/deployment-probes-liveness.yaml
kubectl get pods -l app=nginx-deployment
kubectl rollout status deployment/nginx-deployment
```

---

## 2) Ver as probes no Pod (describe)

Confirme que a liveness está configurada:

```bash
POD=$(kubectl get pod -l app=nginx-deployment -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD"
```

Na seção do container, procure por algo como:

```text
Liveness:   tcp-socket :80 delay=10s timeout=5s period=10s #success=1 #failure=3
```

Ou com httpGet:

```text
Liveness:   http-get http://:80/ delay=10s timeout=5s period=10s #success=1 #failure=3
```

---

## 3) Testar Liveness falhando (reinício do container)

Se a probe falhar (ex.: `path: /giropops` que retorna 404), o Kubernetes reinicia o container após `failureThreshold` falhas consecutivas.

1. Edite o deployment e altere a liveness para um path que não existe (ex.: `path: /giropops`).
2. Aplique e acompanhe:

```bash
kubectl apply -f k8s/deployment-probes-liveness.yaml
kubectl get pods -l app=nginx-deployment -w
```

3. Em outro terminal, veja os eventos do pod (reinícios e motivo):

```bash
POD=$(kubectl get pod -l app=nginx-deployment -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD"
```

Nos Events, deve aparecer algo como:

```text
Liveness probe failed: HTTP probe failed with statuscode: 404
Container nginx failed liveness probe, will be restarted
```

E na saída do pod, `Restart Count` aumenta. Depois do teste, volte o `path` para `/` e aplique de novo.

---

## 4) Readiness Probe – aplicar e ver Pod ficar Ready

Com readiness, o Pod só entra no Endpoint do Service depois da probe passar. No início pode aparecer `0/1` em READY:

```bash
kubectl apply -f k8s/deployment-probes-readiness.yaml
kubectl get pods -l app=nginx-deployment
```

Aguarde alguns segundos (initialDelaySeconds + successThreshold × periodSeconds) e liste de novo:

```bash
kubectl get pods -l app=nginx-deployment
```

Quando a readiness passar, os Pods ficam `1/1` Ready. Confira no describe:

```bash
POD=$(kubectl get pod -l app=nginx-deployment -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD"
```

Procure por:

```text
Readiness:  http-get http://:80/ delay=10s timeout=5s period=10s #success=2 #failure=3
```

---

## 5) Testar Readiness falhando (rollout não conclui)

Se a readiness falhar (ex.: `path: /giropops`), o Pod fica Running mas não fica Ready. Em um rollout, o deployment pode ficar esperando “novas réplicas” sem terminar.

1. Altere no manifesto da readiness o `path` para `/giropops` e aplique.
2. Acompanhe o rollout e os pods:

```bash
kubectl apply -f k8s/deployment-probes-readiness.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx-deployment
```

3. Veja o evento do pod que não fica Ready:

```bash
kubectl describe pod -l app=nginx-deployment | tail -20
```

Esperado: `Readiness probe failed: HTTP probe failed with statuscode: 404`. Depois do teste, volte o `path` para `/`.

---

## 6) Startup Probe

A startup é executada no início da vida do container; depois que passar, liveness e readiness passam a valer. Use quando o processo demorar para subir.

```bash
kubectl apply -f k8s/deployment-probes-startup.yaml
kubectl get pods -l app=nginx-deployment
```

No describe do pod:

```bash
POD=$(kubectl get pod -l app=nginx-deployment -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD"
```

Procure por:

```text
Startup:    http-get http://:80/ delay=10s timeout=5s period=10s #success=1 #failure=3
```

Na startup, `successThreshold` deve ser 1 (só precisa passar uma vez para considerar “iniciado”).

---

## 7) Todas as probes juntas

Exemplo do Day 4 com liveness (exec), readiness (httpGet) e startup (tcpSocket):

```bash
kubectl apply -f k8s/deployment-probes.yaml
kubectl get pods -l app=nginx-deployment
kubectl rollout status deployment/nginx-deployment
```

Ver as três no describe:

```bash
POD=$(kubectl get pod -l app=nginx-deployment -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD"
```

Exemplo de saída:

```text
Liveness:   exec [curl -f http://localhost:80/] delay=10s timeout=5s period=10s #success=1 #failure=3
Readiness:  http-get http://:80/ delay=10s timeout=5s period=10s #success=1 #failure=3
Startup:    tcp-socket :80 delay=10s timeout=5s period=10s #success=1 #failure=3
```

---

## 8) Parâmetros comuns das probes

| Parâmetro            | Significado |
|----------------------|-------------|
| `initialDelaySeconds`| Esperar antes da primeira verificação |
| `periodSeconds`      | Intervalo entre verificações |
| `timeoutSeconds`     | Tempo máximo para cada verificação |
| `successThreshold`   | Quantas sucessos para considerar “ok” (readiness/startup) |
| `failureThreshold`   | Quantas falhas para considerar “falhou” (reinício ou not ready) |

Tipos de teste: `httpGet` (path + port), `tcpSocket` (port), `exec` (command).

---

## 9) Resumo rápido de comandos

| Objetivo              | Comando |
|-----------------------|--------|
| Aplicar com liveness  | `kubectl apply -f k8s/deployment-probes-liveness.yaml` |
| Aplicar com readiness | `kubectl apply -f k8s/deployment-probes-readiness.yaml` |
| Aplicar com startup   | `kubectl apply -f k8s/deployment-probes-startup.yaml` |
| Todas as probes       | `kubectl apply -f k8s/deployment-probes.yaml` |
| Ver probes no pod     | `kubectl describe pod <nome>` |
| Status do rollout     | `kubectl rollout status deployment/nginx-deployment` |

---

## Lição de casa (Day 4)

- Em tudo que criar, usar **probes** (liveness, readiness e startup quando fizer sentido) e **limites de recursos**.
- É inadmissível colocar Pods em produção sem probes e limites configurados.

Referência: [Day 4 – As Probes do Kubernetes](https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day-4/README.md#as-probes-do-kubernetes) e [A sua lição de casa](https://github.com/badtuxx/DescomplicandoKubernetes/blob/main/pt/day-4/README.md#a-sua-licao-de-casa).
