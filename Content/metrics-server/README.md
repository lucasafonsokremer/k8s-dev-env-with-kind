## Instalação e configuração do Metrics Server

O [**Metrics Server**](https://github.com/kubernetes-sigs/metrics-server) é um recurso do Kubernetes na qual coleta métricas de recursos do [**kubelet**](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) e as expõem  no [**Kubernetes API server**](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) por meio da API Metrics.

Para instalar o Metrics Server podemos simplesmente aplicar o arquivo de manifesto ```components.yaml```, mas aqui no meu caso que estou utilizando o Kind terei problemas relacionados ao certificado CA que não é válido.

Abaixo um exemplo do erro que ocorre:
```
E0919 18:23:43.157897       1 manager.go:111] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:estudo-control-plane: unable to fetch metrics from Kubelet estudo-control-plane (estudo-control-plane): Get https://estudo-control-plane:10250/stats/summary?only_cpu_and_memory=true: x509: certificate signed by unknown authority, unable to fully scrape metrics from source kubelet_summary:estudo-worker: unable to fetch metrics from Kubelet estudo-worker (estudo-worker): Get https://estudo-worker:10250/stats/summary?only_cpu_and_memory=true: x509: certificate signed by unknown authority, unable to fully scrape metrics from source kubelet_summary:estudo-worker2: unable to fetch metrics from Kubelet estudo-worker2 (estudo-worker2): Get https://estudo-worker2:10250/stats/summary?only_cpu_and_memory=true: x509: certificate signed by unknown authority]
```

Diante disso, baixe o arquivo ```components.yml```.
```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml
```

Procure o trecho do ```Deployment``` do ```metrics-server``` e passe para o container o argumento ```--kubelet-insecure-tls```. Dessa forma não será consultado a CA dos certificados gerados para os serviços do Kubernetes.

O ```Deployment``` ficará da seguinte forma:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=443
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        image: k8s.gcr.io/metrics-server/metrics-server:v0.5.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /livez
            port: https
            scheme: HTTPS
          periodSeconds: 10
        name: metrics-server
        ports:
        - containerPort: 443
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /readyz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
```

Feito isso, só criar o recurso.
```bash
kubectl create -f components.yaml
```

Após alguns minutos será possível consultar as métrics de recursos do seu cluster. Abaixo dois exemplos simples disso.

```
➜ kubectl top nodes                                                                       
NAME                        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
kindcluster-control-plane   131m         3%     860Mi           22%       
kindcluster-worker          43m          1%     378Mi           9%        
kindcluster-worker2         38m          0%     338Mi           8%
```

```
➜ kubectl top pods --all-namespaces            
NAMESPACE            NAME                                                CPU(cores)   MEMORY(bytes)   
kube-system          coredns-74ff55c5b-4hf8r                             3m           10Mi            
kube-system          coredns-74ff55c5b-95kpl                             4m           10Mi            
kube-system          etcd-kindcluster-control-plane                      11m          40Mi            
kube-system          kube-apiserver-kindcluster-control-plane            53m          339Mi           
kube-system          kube-controller-manager-kindcluster-control-plane   16m          50Mi            
kube-system          kube-proxy-5j22w                                    1m           14Mi            
kube-system          kube-proxy-h5v5r                                    1m           16Mi            
kube-system          kube-proxy-q4wmg                                    1m           20Mi            
kube-system          kube-scheduler-kindcluster-control-plane            3m           20Mi            
kube-system          metrics-server-c5f5f4c85-wwm4s                      4m           17Mi            
kube-system          weave-net-7wv8k                                     3m           43Mi            
kube-system          weave-net-gtxnq                                     1m           43Mi            
kube-system          weave-net-hmp6s                                     1m           43Mi            
local-path-storage   local-path-provisioner-547f784dff-b7czt             1m           7Mi             
metallb-system       controller-6b78bff7d9-f65xj                         1m           6Mi             
metallb-system       speaker-d5djc                                       6m           11Mi            
metallb-system       speaker-rtl25                                       5m           10Mi            
metallb-system       speaker-v9r7j                                       7m           10Mi
```
