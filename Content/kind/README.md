## Instalação e Configuração do Kind.

<p align="center">
  <img width="500" height="350" src="https://d33wubrfki0l68.cloudfront.net/d0c94836ab5b896f29728f3c4798054539303799/9f948/logo/logo.png">
</p>

O [**Kind**](https://kind.sigs.k8s.io/) (Kubernetes in Docker) é uma ferramenta para executar o Kubernetes local em containers [**Docker**](https://docs.docker.com/). O Kind foi inclusive projetado para testar o próprio Kubernetes.

Como pré-requisito, você precisa ter o Docker devidamente instalado e funcional. [Clicando aqui](https://docs.docker.com/get-docker/) você será direcionado para a documentação de instalação do Docker.

**01. Download e instalação do kind.**

Na página do [**Github**](https://github.com/kubernetes-sigs/kind/releases) do projeto você encontra os arquivos compilados para diversas distribuições.

Exemplo de instalação em um GNU/Linux x86_64.

```bash
wget https://github.com/kubernetes-sigs/kind/releases/download/v0.11.1/kind-linux-amd64
chmod +x kind-linux-amd64
sudo mv kind-linux-amd64 /usr/local/bin/kind
kind version
```

**02. Inicializando o cluster.**

É possível inicializar o kind com apenas um nó contendo todas as funções necessárias do Kubernetes.

Um exemplo prático disso é executando o comando abaixo que irá subir um único node.

```bash
kind create cluster
```
> Utilize o **help** do kind para entender todas as possibilidades de uso.

No meu caso, irei inicializar um cluster com 3 nodes, 1 **control-plane** e 2 **workers**, a partir de um arquivo de configuração YAML.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: docker.io/kindest/node:v1.20.7@sha256:cbeaf907fc78ac97ce7b625e4bf0de16e3ea725daf6b04f930bd14c67c671ff9
    kubeadmConfigPatches:
    - |
      kind: InitConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP

  - role: worker
    image: docker.io/kindest/node:v1.20.7@sha256:cbeaf907fc78ac97ce7b625e4bf0de16e3ea725daf6b04f930bd14c67c671ff9
    kubeadmConfigPatches:
    - |
      kind: JoinConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "workloads=true"

  - role: worker
    image: docker.io/kindest/node:v1.20.7@sha256:cbeaf907fc78ac97ce7b625e4bf0de16e3ea725daf6b04f930bd14c67c671ff9
    kubeadmConfigPatches:
    - |
      kind: JoinConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "workloads=true"

networking:
  disableDefaultCNI: true
```

Observe que no YAML informei a versão `1.20.7` do Kubernetes. O kind na versão que estou utilizando, `v0.11.1`, suporta as versões abaixo do Kubernetes:
```
1.21: kindest/node:v1.21.1@sha256:69860bda5563ac81e3c0057d654b5253219618a22ec3a346306239bba8cfa1a6
1.20: kindest/node:v1.20.7@sha256:cbeaf907fc78ac97ce7b625e4bf0de16e3ea725daf6b04f930bd14c67c671ff9
1.19: kindest/node:v1.19.11@sha256:07db187ae84b4b7de440a73886f008cf903fcf5764ba8106a9fd5243d6f32729
1.18: kindest/node:v1.18.19@sha256:7af1492e19b3192a79f606e43c35fb741e520d195f96399284515f077b3b622c
1.17: kindest/node:v1.17.17@sha256:66f1d0d91a88b8a001811e2f1054af60eef3b669a9a74f9b6db871f2f1eeed00
1.16: kindest/node:v1.16.15@sha256:83067ed51bf2a3395b24687094e283a7c7c865ccc12a8b1d7aa673ba0c5e8861
1.15: kindest/node:v1.15.12@sha256:b920920e1eda689d9936dfcf7332701e80be12566999152626b2c9d730397a95
1.14: kindest/node:v1.14.10@sha256:f8a66ef82822ab4f7569e91a5bccaf27bceee135c1457c512e54de8c6f7219f8
```

Agora utililize o comando abaixo para inicializar o cluster.

**OBSERVAÇÃO: Para versões do fedora 32 ou superiores, você precisa executar o seguinte passo antes:**

```bash
# Adiciona a interface do docker no env trusted para que o docker faça conexões remotas
firewall-cmd --permanent --zone=trusted --add-interface=docker0
# Retorna a sua zona
firewall-cmd --get-zone-of-interface=<your eth interface>
# Com o retorno da zona acima, libera o docker a criar conexões locais
firewall-cmd --zone=<zone from above> --add-masquerade --permanent
# restart das regras de firewall
firewall-cmd --reload
# Instalação do grubby
dnf install -y grubby
grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
reboot
```

* Link do problema:

```
https://kind.sigs.k8s.io/docs/user/known-issues/#fedora32-firewalld
```

Se você utiliza fedora e já executou os comandos acima, ou utiliza outra distro, execute o comando abaixo para que com base no yml acima, ele faça a criação do cluster e crie um kubeconfig:

```bash
kind create cluster --name kindcluster --config kind.yml --kubeconfig ~/.kube/kind.yml
```

Para instalar o kubectl basta:
```bash
curl -LO https://dl.k8s.io/release/v1.20.7/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
mkdir ~/.kube
```

Com o cluster inicializado, exporte a variável `KUBECONFIG`, pois no comando acima foi informado onde seria escrito o arquivo `kubeconfigfile`.
```bash
export KUBECONFIG="$HOME/.kube/kind.yml"
```

Executando o comando ```kubectl get nodes``` percebe-se que o status dos nodes será **NotReady**, pois o CNI padrão, o **kindnet**, está desabilitado. Dessa forma os pods não podem se comunicar.

Faço isso, porque prefiro estudar/trabalhar com o CNI [**Weave-net**](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/), que para instalar basta executar o comando abaixo.

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
Finalizado o deployment do `weave-net` você terá um cluster pronto para estudo.

```bash
NAME                        STATUS   ROLES                  AGE   VERSION
kindcluster-control-plane   Ready    control-plane,master   12h   v1.20.7
kindcluster-worker          Ready    <none>                 12h   v1.20.7
kindcluster-worker2         Ready    <none>                 12h   v1.20.7
```
