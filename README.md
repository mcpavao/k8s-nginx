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

**Stack de observabilidade via Helm.** Prometheus, Grafana e Alertmanager instalados pelo chart `kube-prometheus-stack`, em namespace próprio. O chart traz dezenas de regras de alerta prontas, incluindo cobertura para deployments com réplicas indisponíveis — em produção vale verificar o que já está coberto antes de escrever regra própria.

**Exporter como sidecar, não como pod separado.** O `stub_status` do nginx só é acessível localmente, então o exporter precisa compartilhar a rede do pod para alcançá-lo via `localhost`. Como sidecar, cada réplica expõe as próprias métricas e o Prometheus identifica a origem por label. Um coletor externo teria que consumir de um Service, sem saber de qual pod veio o dado, e viraria ponto único de falha.

**Porta separada para o status.** O `stub_status` escuta em 8080, fora da porta 80 que serve o conteúdo público, para não expor informação interna junto com a aplicação.

**ServiceMonitor em vez de configuração estática.** O alvo é descoberto por label, não declarado por endereço: o ServiceMonitor encontra o Service, que encontra os pods. Nada no Deployment referencia o monitoramento, o que permite adicionar ou remover observabilidade sem tocar na aplicação nem reiniciar pods.

**Alertas sobre infraestrutura, não sobre serviço.** As regras atuais observam contagem de réplicas. Com `nginx_http_requests_total` disponível, seria possível alertar sobre throughput ou taxa de erro, que é o que afeta o usuário. Um alerta de ausência de tráfego não foi incluído porque este ambiente não tem carga constante e ele geraria falso positivo diariamente.