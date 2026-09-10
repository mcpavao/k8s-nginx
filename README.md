# k8s-nginx

Deployment e Service do nginx em Kubernetes, executado localmente em cluster kind.

## Como rodar

```bash
kind create cluster --name k8s-nginx
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl port-forward service/nginx 8080:80
```

Acesse `http://localhost:8080`. Para remover: `kind delete cluster --name k8s-nginx`.

## Decisões

**Deployment em vez de pods soltos.** Um pod não é recriado se morrer. O Deployment mantém o número declarado de réplicas, permite rolling update sem downtime e guarda os ReplicaSets anteriores, o que torna `kubectl rollout undo` uma reversão de segundos.

**Duas réplicas.** Permite atualização sem indisponibilidade: os pods novos sobem e só então os antigos são encerrados.

**Imagem com tag fixa (`nginx:1.27`).** Com `latest`, réplicas diferentes podem subir com versões diferentes sem aviso.

**Service para endereço estável.** Pods são efêmeros e o IP muda a cada recriação. O Service localiza os pods por label, não por lista de IPs, e por isso continua funcionando enquanto eles vão e vêm.

**Readiness e liveness.** Readiness controla se o pod entra nos endpoints do Service; liveness reinicia o container travado. Os períodos são assimétricos de propósito: readiness verifica com mais frequência porque o custo de errar é baixo, liveness é mais conservadora porque uma probe agressiva pode reiniciar pods saudáveis sob carga.

**Limitação conhecida.** As duas probes apontam para o mesmo endpoint. Em produção isso é questionável: a liveness deveria verificar algo mais profundo, senão só adiciona risco de reinício sem detectar nada que a readiness já não detecte.