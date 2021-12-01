# A ideia deste lab de testes é instalar um Rancher em HA, cluster de 3 nodes, usando instâncias AWS EC2.

## Para este cenário é necessário ter uma conta na AWS e uma máquina para gerenciar o cluster com os seguintes requisitos:
  - Ter o Kubectl instalado.
  - Ter o RKE instalado.
  - Ter o Helm instalado.
  - Ter acesso ssh às instancias EC2.
  - 
 
 ##  Próximos passos - instancias EC2
  - crie 3 instâncias EC2 em sua conta AWS em zonas de disponibilidade diferentes, exemplo (us-east-1a, us-east-1b e us-east-1c), neste lab estou utilizando com Ubuntu 20.04.
  - Crie 2 target groups rancher-80 e rancher-443 apontando para as máquinas do cluster.
  - Crie o load balancer NLB com dois listeners apontando para os target groups rancher-80 e rancher-443.
  - Crie uma entrada de dns no Route53 rancher.seudominio.tld apontando para o CNAME do Load Balancer.
  - Crie um security group para liberar acesso a porta 80 e 443 as máquinas do cluster.
  - Crie um security group para que possa instalar o cluster a partir do seu IP via rke (all ports) caso esteja utilizando uma máquina local(fora da AWS).


## Preparando as instancias EC2
  - Instalar o docker nas 3 instancias.
  - Neste lab estou executando a instalação diretamente do user_data na criação da instancia e só alterando o nome do host no comando do script abaixo na linnha `hostnamectl set-hostname` e os nomes `k8s-1`, `k8s-2` e `k8s-3` para uma melhor identificação.



```bash
#!/bin/bash
# Update system
apt update && apt upgrade -y

# Definindo um nome para o host
hostnamectl set-hostname k8s-1

# install Docker
curl https://releases.rancher.com/install-docker/20.10.sh | sh
usermod -aG docker ubuntu
clear
```


## Com tudo instalado vamos configurar o RKE
 #### Preparando e instalando o cluster (da sua máquina) - criando configuração. Execute o comando abaixo para criar o manifesto do cluster.
```
rke config
```

#### Ele vai te fazer umas perguntinhas, cadastre os nodes, aponte o local da sua chave ssh, a mesma que usou nas EC2, forneça todos os dados necessários como ip da instancia, usuário, as roles(que neste lab estou usando os 3 nodes como controlplane, worker e etcd).
#### Abaixo segue um exemplo:

```
[+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]:
[+] Number of Hosts [1]:
[+] SSH Address of host (1) [none]: 54.31.31.27
[+] SSH Port of host (1) [22]:
[+] SSH Private Key Path of host (54.31.31.27) [none]:
[-] You have entered empty SSH key path, trying fetch from SSH key parameter
[+] SSH Private Key of host (54.31.31.27) [none]:
[-] You have entered empty SSH key, defaulting to cluster level SSH key: ~/.ssh/id_rsa
[+] SSH User of host (54.31.31.27) [ubuntu]:
[+] Is host (54.31.31.27) a Control Plane host (y/n)? [y]: y
[+] Is host (54.31.31.27) a Worker host (y/n)? [n]: y
[+] Is host (54.31.31.27) an etcd host (y/n)? [n]: y
[+] Override Hostname of host (54.31.31.27) [none]:
[+] Internal IP of host (54.31.31.27) [none]:
[+] Docker socket path on host (54.31.31.27) [/var/run/docker.sock]:
[+] Network Plugin Type (flannel, calico, weave, canal, aci) [canal]:
[+] Authentication Strategy [x509]:
[+] Authorization Mode (rbac, none) [rbac]:
[+] Kubernetes Docker image [rancher/hyperkube:v1.21.5-rancher1]:
[+] Cluster domain [cluster.local]:
[+] Service Cluster IP Range [10.43.0.0/16]:
[+] Enable PodSecurityPolicy [n]:
[+] Cluster Network CIDR [10.42.0.0/16]:
[+] Cluster DNS Service IP [10.43.0.10]:
[+] Add addon manifest URLs or YAML files [no]:

```

#### O comando `rke config` cria o arquivo `cluster.yml` que contém todas as configurações do cluster. Este arquivo pode ser modificado de acordo com a necessidade, novos nodes podem ser adicionados, modificados e até removidos.
#### Agora que os nodes já estão cadastrados, vamos começar o provisionamento executando o comando abaixo:
```
rke up —config cluster.yml
```

#### Após instalar verá a seguinte mensagem (se tudo der certo).
```
INFO[0255] Finished building Kubernetes cluster successfully
```

#### Configure o kubectl - Se preferir adicione também a linha abaixo no `~/.bashrc` do usuário.
```
export KUBECONFIG=kube_config_cluster.yml
```

#### Verifique se o cluster está funcionando
```
kubctl get nodes                
kubectl get pods --all-namespaces     
```


#### Preparando e instalando o cert-manager neste cluster (maquina que irá gerenciar).
#### Instale os crds do cert-manager
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.crds.yaml
```


#### Adicione o repo e atualize o índices.
```
helm repo add jetstack https://charts.jetstack.io              
helm repo update
```

#### Criar o namespace cert-manager.
```
kubectl create namespace cert-manager
```

#### Instalar o cert-mamanger
```
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.6.1
```

#### Crie o cluster issuer, pois sem isso ele não gera os certs via Let's Encrypt. No exemplo abaixo é necessário definir seu e-mail, coloquei o nome do arquivo de issuer.yml.

```yml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: marcio.mmachado1976@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx

```



#### Após criar o arquivo execute o comando:
```
kubectl apply -f issuer.yml
```



#### saída esperada
```
clusterissuer.cert-manager.io/letsencrypt-prod created
```



#### Verifique se está ok.
```
kubectl get clusterissuer 
```

#### Saída esperada
```
NAME               READY   AGE
letsencrypt-prod   True    48s
```
#### se estiver mostrando “True” deu certo!



#### Preparando e instalando o rancher neste cluster (maquina que irá gerenciar), adicione o repo e atualize os índices.
```
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest              
helm repo update              
```


#### Criar o namespace cattle-system.
```
kubectl create namespace cattle-system
```

#### instalar o rancher.
```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.cloud-na-veia.cf \
  --set replicas=3 \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=marcio.mmachado1976@gmail.com
```

#### Verifique os pods do rancher.
```
kubectl get pods -n cattle-system
```

#### Após o Helm instalar o rancher você verá a saída do comando para pegar a senha gerada para o primeiro acesso, depois disso, acesse o rancher via web através da URL definida e siga os procedimentos para trocar a senha e iniciar o uso do seu rancher.



#### Ref: 
- RKE e Rancher
  - <https://rancher.com/docs/rancher/v2.6/en/installation/resources/k8s-tutorials/infrastructure-tutorials/infra-for-ha/>

  - <https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/>

- Helm - rancher via helm
  - <https://artifacthub.io/packages/helm/rancher-stable/rancher>


- Helm cert-manager
  - <https://artifacthub.io/packages/helm/cert-manager/cert-manager>






































