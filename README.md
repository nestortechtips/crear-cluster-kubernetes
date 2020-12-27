![Nestor Tech Tips](https://storage.googleapis.com/nestortechtips.online/cover.png)

# Preparar clúster de Kubernetes en Ubuntu 18.04 con kubeadm
En este documento aprendereás cómo crear un clúster de Kubernetes

# Requisitos
* 3 servidores con el Sistema Operativo Ubuntu 18.04(1 actuará como *controlador* y 2 serán los *trabajadores*)
* Acceso a Internet en los 3 servidores
* Acceso como súper usuario o *root*

# Procedimiento

## Configuración de repositorios (3 servidores)
Necesitaremos agregar 2 repositorios a los servidores, los cuales nos permitirán instalar **Docker** como motor de contenedores y **Kubernetes** como orquestrador.

### Docker

El primer comando descargará la llave pública del repositorio *Docker*, la cuál nos ayudará a verificar que los paquetes sean legítimos
```bash
user@controlplane:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg
OK
```
A continuación agregamos el repositorio *Docker*
```bash
user@controlplane:~$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
Hit:1 http://us-west-1.ec2.archive.ubuntu.com/ubuntu bionic InRelease
Get:2 http://us-west-1.ec2.archive.ubuntu.com/ubuntu bionic-updates InRelease [88.7 kB]
Get:3 http://us-west-1.ec2.archive.ubuntu.com/ubuntu bionic-backports InRelease [74.6 kB]
Hit:4 http://security.ubuntu.com/ubuntu bionic-security InRelease
Get:5 https://download.docker.com/linux/ubuntu bionic InRelease [64.4 kB]
Get:6 http://us-west-1.ec2.archive.ubuntu.com/ubuntu bionic-updates/universe amd64 Packages [1680 kB]
Get:7 https://download.docker.com/linux/ubuntu bionic/stable amd64 Packages [13.0 kB]
Fetched 1921 kB in 1s (2389 kB/s)
Reading package lists... Done
```
### Kubernetes
Agregamos la llave pública de *Kubernetes*
```bash
user@controlplane:~$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
OK
```
Ahora agregamos el repositio
```bash
user@controlplane:~$ cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
deb https://apt.kubernetes.io/ kubernetes-xenial main
```

Para confirmar que ambos repositorioas se hayan agregado correctamente vamos a actualizar la lista de paquetes

```bash
user@controlplane:~$ sudo apt-get update
Hit:1 http://us-west-1.ec2.archive.ubuntu.com/ubuntu bionic InRelease
Hit:2 http://us-west-1.ec2.archive.ubuntu.com/ubuntu bionic-updates InRelease
Get:3 http://us-west-1.ec2.archive.ubuntu.com/ubuntu bionic-backports InRelease [74.6 kB]
Hit:4 https://download.docker.com/linux/ubuntu bionic InRelease
Hit:6 http://security.ubuntu.com/ubuntu bionic-security InRelease
Get:5 https://packages.cloud.google.com/apt kubernetes-xenial InRelease [8993 B]
Get:7 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 Packages [41.3 kB]
Fetched 125 kB in 1s (167 kB/s)
Reading package lists... Done
```
La salida del comando nos muestra que *https://download.docker.com/linux/ubuntu bionic InRelease* y *https://packages.cloud.google.com/apt kubernetes-xenial InRelease* se encuentran disponibles

## Instalación de Docker (3 servidores)

En esta ocasión utilizaremos *Docker* como el motor de contenedores en nuestro clúster debido a la amplia adopción y basta documentación, cabe mencionar que existen otras [opciones](https://kubernetes.io/docs/concepts/containers/#container-runtimes)

Antes de instalar *Docker* nos aseguraremos de deshabilitar la memoria swap en el servidor
```bash
user@controlplane:~$ sudo swapoff -a
```

Una vez que la memoria swap esté deshabilitada continuaremos con la instalación de *Docker*
```bash
user@controlplane:~$ sudo apt-get install -y docker-ce
Reading package lists... Done
Building dependency tree
Reading state information... Done
....
Created symlink /etc/systemd/system/sockets.target.wants/docker.socket → /lib/systemd/system/docker.socket.
Processing triggers for libc-bin (2.27-3ubuntu1.2) ...
Processing triggers for systemd (237-3ubuntu10.42) ...
Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
Processing triggers for ureadahead (0.100.0-21) ...
```

Ahora agregaremos nuestro usuario al grupo Docker por si queremos analizar algún contenedor en el futuro. Los cambios tomarán efecto después de reiniciar nuestra sesión
```bash
sudo usermod -aG docker $USER
```

Podemos comprabar que *Docker* se ha instalado de manera correcta con el siguiente comando
```bash
user@controlplane:~$ docker version
Client: Docker Engine - Community
 Version:           19.03.13
 API version:       1.40
 Go version:        go1.13.15
 Git commit:        4484c46d9d
 Built:             Wed Sep 16 17:02:36 2020
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.13
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       4484c46d9d
  Built:            Wed Sep 16 17:01:06 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.3.7
  GitCommit:        8fba4e9a7d01810a393d5d25a3621dc101981175
 runc:
  Version:          1.0.0-rc10
  GitCommit:        dc9208a3303feef5b3839f4323d9beb36df0a9dd
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```

La salida del comando nos muestra la versión actual del cliente (19.03.13) y servidor/motor (19.03.13) de *Docker*

## Instalación de Kubernetes (3 servidores)
Una vez finalizada la instalación del motor de contenedores continuaremos continuaremos con la instalación de los componentes de nuestro orquestrador. kubelet se encarga de gestionar los *pod* asignados al nodo, kubeadm se encargará de iniciar y configurar los componentes de nuestro controlador y kubectl nos permitirá interactuar con el clúster a través del API
```bash
user@controlplane:~$ sudo apt-get install -y kubelet kubeadm kubectl
Reading package lists... Done
Building dependency tree
Reading state information... Done
...
Created symlink /etc/systemd/system/multi-user.target.wants/kubelet.service → /lib/systemd/system/kubelet.service.
Setting up kubectl (1.19.3-00) ...
Setting up kubeadm (1.19.3-00) ...
Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
```
Instalados los paquetes configurarémos las *IPTables* de los servidores para que la comunicación intra-cluster no tenga ningún problema
```bash
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
```

## Configuración del clúster
### Inicialización del clúster (servidor controlador/maestro)
Para inicializar los componentes basta con indicarle a kubeadm el rango de red con el que trabajarán los pods de nuestro clúster
```bash
user@controlplane:~$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
W1029 21:59:38.413226   32226 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.19.3
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
....
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.39.180:6443 --token 6u51cd.c9mqy58au927fdq2 \
    --discovery-token-ca-cert-hash sha256:964b795be9c5f49273608bc3620493bc91022b356cbc3ce63613719392938ed3
```

La salida del comando nos advierte que para comenzar a utilizar el clúster ejecutemos los comandos
```bash
  user@controlplane:~$ mkdir -p $HOME/.kube
  user@controlplane:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  user@controlplane:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### Unir nodos al clúster (2 servidores trabajadores/worker)
Para que los demás servidores se unan al clúster basta con ejecutar el comando mencionado en la inicialización del clúster
```bash
user@worker1:~$ sudo kubeadm join 172.31.39.180:6443 --token 6u51cd.c9mqy58au927fdq2 \
>     --discovery-token-ca-cert-hash sha256:964b795be9c5f49273608bc3620493bc91022b356cbc3ce63613719392938ed3
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
....
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

Podemos verificar que los 2 servidores worker/trabajadores se han unido al clúster ejecuntando el siguiente comando el nodo maestro/controlador
```bash
user@controlplane:~$ kubectl get nodes
NAME           STATUS     ROLES    AGE     VERSION
controlplane   NotReady   master   7m53s   v1.19.3
worker1        NotReady   <none>   113s    v1.19.3
worker2        NotReady   <none>   111s    v1.19.3
```

### Instalar plugin de red (servidor controlador/maestro)
Los tres nodos se han unido al clúster pero si ponemos atención a la columna de *Status* del comando anterior nos percataremos que no se encuentran listos. Esto es debido a que nos falta configurar un componente escencial en el clúster, el plugin de red. Sin este plugin no es posible tener comunicación entre los pods del clúster. Existe una gran variedad de plugins disponibles, en esta ocasión instalaremos *flannel* debido a la facilidad para configurarlo.

```bash 
user@controlplane:~$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```
De esta forma estamos instalando el plugin a través de objetos de *Kubernetes*. Después de unos segundos, dependiendo de la red en donde se encuentran los servidores, estará lista la comunicación en el clúster y podemos verificarlo checando los nodos del clúster de nueva cuenta
```bash
user@controlplane:~$ kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
controlplane   Ready    master   16m   v1.19.3
worker1        Ready    <none>   10m   v1.19.3
worker2        Ready    <none>   10m   v1.19.3
```

El *Status* de los nodos es *Ready*. Ahora es posible crear objetos de Kubernetes en nuestro ambiente.

Como si se tratara de una introducción a un lenguaje de programación, creaemos un ejemplo *Hola Mundo* al estilo de Kubernetes.
```bash
user@controlplane:~$ kubectl create deployment hola-mundo --image=nginx
deployment.apps/hola-mundo created
```
Después de unos momentos el *deployment* estará listo.
```bash
user@controlplane:~$ kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hola-mundo   1/1     1            1           81s
```

Para poder acceder al deployment basta con exponerlo de la siguiente forma, con lo cuál será posible consultar en el puerto *30080* del nodo al que se asigne

```bash
user@controlplane:~$ kubectl expose deployment hola-mundo --type=NodePort --port=30080
service/hola-mundo exposed
user@controlplane:~$ curl 10.97.228.29:30080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Contribuciones
Puedes contribuir utilizando **Pull Requests** en el repositorio correspondiente. De no existir crea un archivo *contrib.txt*, de existir modifica el archivo para agregar tu nombre de usuario o correo. Por ejemplo:
* `fulano.detal@correo.com`
* `perengano`

## Créditos
El contenido de este repositorio corresponde a la publicación homónima en [nestortechtips.online](https://nestortechtips.online/). 