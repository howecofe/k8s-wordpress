# kubeadem을 이용한 k8s 배포 (Ubuntu)

## 1. kubeadm Installation
모든 노드에서 아래 커맨드를 실행한다.
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
- **kubeadm** : Kubernetes에서 제공하는 기본적인 도구로서, Kubernetes Cluster를 가장 빨리 구축하기 위한 다양한 기능을 제공한다.
- **kubectl** : Kubernetes Cluster를 제어하기 위한 커맨드 라인 도구
- **kubelet** : Worker Nodes에 존재하여 Cluster의 각 Node에서 실행되는 에이전트로, 쉽게 말하면 Node의 선장 역할을 한다. Pod에서 Container가 확실하게 동작하도록 관리한다. 일반적으로 kube-apiserver와의 통신을 담당한다

---
## 2. Cluster Deployment
### [Control-plane]
```bash
kubeadm init —control-plane-endpoint <IP 주소> --pod-network-cidr <IP 대역> --apiserver-advertise-address <IP 주소>
```
+ `--apiserver-advertise-address`: 다른 노드에게 api 주소를 알려주기 위한 옵션

- 자격 증명 파일
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```
- 컨테이너 네트워크 인터페이스(CNI) 설치
```bash
kubectl apply –f <add-on.yaml>
```
- `<add-on.yaml>` : calico network(https://docs.projectcalico.org/manifests/calico.yaml)
- 또는 `curl https://docs.projectcalico.org/manifests/calico.yaml -O`
#### 확인
STATUS가 READY 상태가 되어야 한다.
```
kubectl get nodes
```

### [Worker Nodes]
- 각 Worker Node를 클러스터에 추가
```bash
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
  + `<control-plane-host>:<control-plane-port>` : IP 주소:포트 번호
  + `<token>` : **control-plane**에서 `kubeadm token list` 명령어로 확인
  + `<hash>` : **control-plane**에서 아래 명령어로 확인
  ```bash
  openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null |\
  openssl dgst -sha256 -hex | sed 's/^.* //'
  ```

---
## 3. Upgrade Version
각 노드에서 업그레이드하는 순서가 매우 중요하다.
<mark>kubeadm --> kubectl / kubelet</mark>

- 업그레이드할 버전 결정
```
sudo apt update && sudo apt-cache madison kubeadm
```

### [Control-plane]
컨트롤 플레인 노드의 업그레이드 절차는 한 번에 한 노드씩 실행해야 한다. 먼저 업그레이드할 컨트롤 플레인 노드를 선택한다. `/etc/kubernetes/admin.conf` 파일이 있어야 한다.

#### kubeadm upgrade
- 1.21.x-00에서 x를 최신 패치 버전으로 바꾼다.
```
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.21.x-00 && \
apt-mark hold kubeadm
```

- apt-get 버전 1.1부터 다음 방법을 사용할 수도 있다.
```
sudo apt-get update && \
sudo apt-get install -y --allow-change-held-packages kubeadm=1.21.x-00
```

```
sudo kubeadm upgrade apply v1.21.x
```

#### kubectl / kubelet upgrade
- 1.21.x-00에서 x를 최신 패치 버전으로 바꾼다.
```
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && apt-get install -y kubelet=1.21.x-00 kubectl=1.21.x-00 && \
sudo apt-mark hold kubelet kubectl
```

- apt-get 버전 1.1부터 다음 방법을 사용할 수도 있다.
```
sudo apt-get update && \
sudo apt-get install -y --allow-change-held-packages kubelet=1.21.x-00 kubectl=1.21.x-00
```

- kubelet 다시 시작
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### [Worker Nodes]
#### kubeadm upgrade

- 1.21.x-00의 x를 최신 패치 버전으로 바꾼다.
```
sudo apt-mark unhold kubeadm && \
sudo apt-get update && apt-get install -y kubeadm=1.21.x-00 && \
sudo apt-mark hold kubeadm
```

- apt-get 버전 1.1부터 다음 방법을 사용할 수도 있다.
```
sudo apt-get update && \
sudo apt-get install -y --allow-change-held-packages kubeadm=1.21.x-00
```

- 워커 노드의 경우 로컬 kubelet 구성을 업그레이드한다.
```
sudo kubeadm upgrade node
```

#### kubectl / kubelet upgrade
- 1.21.x-00의 x를 최신 패치 버전으로 바꾼다.
```
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && apt-get install -y kubelet=1.21.x-00 kubectl=1.21.x-00 && \
sudo apt-mark hold kubelet kubectl
```

- apt-get 버전 1.1부터 다음 방법을 사용할 수도 있다.
```
sudo apt-get update && \
sudo apt-get install -y --allow-change-held-packages kubelet=1.21.x-00 kubectl=1.21.x-00
```

- kubelet 다시 시작
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
- Downgrade
```
sudo apt install kubeadm=1.19.12-00 kubectl=1.19.12-00 kubelet=1.19.12-00
```

#### Cluster 상태 확인
```
kubectl get nodes
```

---
## 4. Addon
- Kubernetes Cluster는 Control-plane Node, Worker Node 외에 추가로 Addon도 존재한다.
- Addon은 Kubernetes Cluster에서 기능을 구현하거나 확장하는 역할을 한다.
- Kubernetes Cluster가 필요한 기능을 실행하기 위해 pod와 service 형태로 존재한다.
- Addon이 사용하는 Namespace는 kube-system이다.
- Addon에 사용되는 Pod는 Deployment, Replication controlller 등에 의해 관리된다.
- 외부 연동 라이브러리
- [참고](https://ikcoo.tistory.com/3)

---
### 4-1. MetalLB
- pod 서비스를 외부로 노출시키기 위한 원시적인 방법은 NodePort를 이용하는 것이지만, 이는 인스턴스의 IP가 변경되면 해당 서비스에서 이를 반영해야 한다는 불편함이 있다. 따라서 GCP나 Azure와 같은 클라우드 벤더에서는 외부 사용자가 특정 서비스에 접근할 수 있도록 LoadBalancer나 Ingres 서비스 타입을 통해 서비스를 외부로 노출할 수 있도록 지원한다. 그러나 Kubernetes 자체에서는 LoadBalancer 타입의 서비스가 제공되지 않는다.
- MetalLB는 On-Premise 환경에서도 로드밸런서 타입의 서비스를 배포할 수 있도록 LoadBalancing 기능을 제공한다.
- [참고1](https://cla9.tistory.com/94)
- [참고2](https://boying-blog.tistory.com/16)


```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

```
vi metallb-config.yaml

#
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.200.xxx-192.168.200.xxx
#
```

```
kubectl create -f metallb-config.yaml
```

#### 확인
nginx의 EXTERNAL-IP가 <pending>이 아니라 metallb-config.yaml 파일에서 설정한 주소 범위 안의 ip 주소로 할당된다.
  
```
kubectl create deploy nginx --image=nginx
kubectl expose deploy nginx --port 80 --type LoadBalancer
kubectl get svc
```

---
### 4-2. Ingress
- Ingress는 외부에서 쿠버네티스 클러스터 내부로 들어오는 네트워크 요청 즉, Ingress 트래픽을 어떻게 처리할지 정의한다.
- 외부에서 쿠버네티스에서 실행 중인 Deployment와 Service에 접근하기 위한, 일종의 관문 (Gateway) 같은 역할을 담당한다.
- [참고](https://blog.naver.com/alice_k106/221502890249)

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/aws/deploy.yaml
```

```
kubectl get ns
kubectl get pods -n ingress-nginx
kubectl edit svc -n ingress-nginx ingress-nginx-controller
# --> spec.externalIPs: 에 각 node IP 주소를 적어준다. (워커노드의 목록을 나열)
kubectl get svc -n ingress-nginx
```

```
vi myapp-ing.yaml
#
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: myapp-ing
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: myapp-svc-np
          servicePort: 80
#
```

```
kubectl create -f myapp-ing.yaml
```
---
### 4-3. Rook
#### Vagantfile 생성
각 노드의 VM에 10G 크기의 비어 있는 디스크를 할당한다.
```
vi Vagantfile
#
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # k-control VM
  config.vm.define "k-control" do |config|
    config.vm.box = "ubuntu/focal64"
    config.vm.provider "virtualbox" do |vb|
       vb.name = "k-control"
       vb.cpus = 2
       vb.memory = 3000
    end
    config.vm.hostname = "k-control"
    config.vm.network "private_network", ip: "192.168.200.50"
  end

  # k-node1 VM
  config.vm.define "k-node1" do |config|
    config.vm.box = "ubuntu/focal64"
    config.vm.provider "virtualbox" do |vb|
      vb.name = "k-node1"
      vb.cpus = 2
      vb.memory = 3000
      unless File.exist?('./.disk/ceph1.vdi')
       vb.customize ['createmedium', 'disk', '--filename', './.disk/ceph1.vdi', '--size', 10240]
      end
      vb.customize ['storageattach', :id, '--storagectl', 'SCSI', '--port', 2, '--device', 0, '--type', 'hdd', '--medium',
'./.disk/ceph1.vdi']
    end
    config.vm.hostname = "k-node1"
    config.vm.network "private_network", ip: "192.168.200.51"
  end

  # k-node2 VM
  config.vm.define "k-node2" do |config|
    config.vm.box = "ubuntu/focal64"
    config.vm.provider "virtualbox" do |vb|
      vb.name = "k-node2"
      vb.cpus = 2
      vb.memory = 3000
      unless File.exist?('./.disk/ceph2.vdi')
        vb.customize ['createmedium', 'disk', '--filename', './.disk/ceph2.vdi', '--size', 10240]
      end
      vb.customize ['storageattach', :id, '--storagectl', 'SCSI', '--port', 2, '--device', 0, '--type', 'hdd', '--medium',
'./.disk/ceph2.vdi']
    end
    config.vm.hostname = "k-node2"
    config.vm.network "private_network", ip: "192.168.200.52"
  end

  # k-node3 VM
  config.vm.define "k-node3" do |config|
    config.vm.box = "ubuntu/focal64"
    config.vm.provider "virtualbox" do |vb|
      vb.name = "k-node3"
      vb.cpus = 2
      vb.memory = 3000
      unless File.exist?('./.disk/ceph3.vdi')
        vb.customize ['createmedium', 'disk', '--filename', './.disk/ceph3.vdi', '--size', 10240]
      end
      vb.customize ['storageattach', :id, '--storagectl', 'SCSI', '--port', 2, '--device', 0, '--type', 'hdd', '--medium',
'./.disk/ceph3.vdi']
    end
    config.vm.hostname = "k-node3"
    config.vm.network "private_network", ip: "192.168.200.53"
  end
end
#
```

- vagrant halt 후 reload
```
vagrant reload
```

- 빈 디스크가 생성되었는지 확인
```
vagrant ssh k-node`x` -- lsblk -f
```

#### ceph cluster 생성

- Ceph 소스 다운로드
```
git clone --single-branch --branch v1.6.7 https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
```

- crds, common, operator 설치
```
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

- cluster 생성
```
kubectl create -f cluster.yaml (3 worker)
or
kubectl create -f cluster-test.yaml (1 worker)
```

#### Block 장치용 스토리지 클래스 생성
```
kubectl create -f csi/rbd/storageclass.yaml
```

#### File 스토리지 생성
- 파일 시스템 생성
```
kubectl create -f filesystem.yaml (3 worker)
or
kubectl create -f filesystem-test.yaml (1 worker)
```

- 파일 스토리지 생성
```
kubectl create -f csi/cephfs/storageclass.yaml
```

#### rook-ceph status 확인
- toolbox 생성
```
kubectl create -f toolbox.yaml
```

- health: HEALTH_WARN / HEALTH_OK 둘중 하나의 상태여야 한다.
```
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph -s
```
---
### 4-4. Metrics-server
- 쿠버네티스 v1.11부터 heapster가 더 이상 사용되지 않게 되었으므로(deprecated), HPA(horizontal pod autoscaler)나 `kubectl top` 명령어를 사용하려면 metrics-server를 사용해야 한다.
- Metrics Server는 heapster를 간소화한 버전이다. 
- Metrics Server는 클러스터 전체의 리소스 사용량 데이터를 집계한다. 각 노드에 설치된 kublet을 통해 노드나 컨테이너의 CPU나 메모리 사용량과 같은 메트릭을 수집한다. 즉, kubelet에서 메트릭데이터를 수집해서 메모리에 저장한다.
- 쿠버네티스에서 필요한 핵심 데이터들은 대부분 etcd에 저장된다. 그러나 메트릭 데이터들을 etcd에 저장하면 etcd의 부하가 너무 커지기 때문에 Metrics-server는 그렇게 하지 않고 메모리에 저장하도록 되어 있다.
- +) 데이터를 메모리에 저장하기 때문에 메트릭 서버용으로 실행한 포드가 재시작되면 수집되었던 데이터가 사라진다. 따라서 데이터 보관주기를 길게 하려면 별도의 외부 스토리지를 사용하도록 설정해야 한다.
- [참고1](https://kangwoo.github.io/devops/kubernetes/kubernetes-metrics-server-install/)
- [참고2](https://arisu1000.tistory.com/27856)

#### Metrics Server git 저장소 복제(Clone)
```
git clone https://github.com/kubernetes-incubator/metrics-server.git
```

#### kubectl을 이용해서 적용
  - v1beta1.metrics.k8s.io 라는 apiservce가 생성
  - metrics-server라는 디플로이먼트와 서비스 생성
```
cd metrics-server
kubectl apply -f deploy/1.8+/
```

#### 확인
- apiservice 확인
```
kubectl get apiservices | grep metrics
```

- deployment & service 확인
```
kubectl -n kube-system get deploy,svc | grep metrics-server
```

- 노드의 CPU & Memory 사용량 확인
```
kubectl top node
```
