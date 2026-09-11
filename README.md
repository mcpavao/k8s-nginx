# k8s-nginx

Deployment, Service, ConfigMap e Ingress do nginx em Kubernetes, executado localmente em cluster kind.

## Como rodar

```bash
kind create cluster --name k8s-nginx --config kind-config.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=120s

kubectl apply -f configmap.yaml -f deployment.yaml -f service.yaml -f ingress.yaml
```

Acesse `http://localhost`. Para remover: `kind delete cluster --name k8s-nginx`.

## Decisões

**Deployment em vez de pods soltos.** Um pod não é recriado se morrer. O Deployment mantém o número declarado de réplicas, permite rolling update sem downtime e guarda os ReplicaSets anteriores, o que torna `kubectl rollout undo` uma reversão de segundos.

**Duas réplicas.** Permite atualização sem indisponibilidade: os pods novos sobem e só então os antigos são encerrados.

**Imagem com tag fixa (`nginx:1.27`).** Com `latest`, réplicas diferentes podem subir com versões diferentes sem aviso.

**Service para endereço estável.** Pods são efêmeros e o IP muda a cada recriação. O Service localiza os pods por label, não por lista de IPs, e por isso continua funcionando enquanto eles vão e vêm.

**Readiness e liveness.** Readiness controla se o pod entra nos endpoints do Service; liveness

**Regras de alerta com severidades separadas.** Duas regras em vez de uma: `NginxReplicasDegraded` (warning, `for: 2m`) para capacidade reduzida, e `NginxDown` (critical, `for: 1m`) para serviço indisponível. A cláusula `for` evita que um rolling update normal dispare alerta, já que a contagem de réplicas cai por alguns segundos durante a substituição dos pods. O `for` menor no alerta crítico reflete que a espera custa mais caro quando o serviço está fora do ar.