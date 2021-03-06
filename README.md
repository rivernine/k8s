# k8s

## 1. Requirement
  - CPU: 2core
  - RAM: 2GB
  - Storage: 30GB
  - Disabled swap memory
    ```sh
    sudo swapoff -a
    # /etc/fstab에서 swap memory 관련 줄 주석처리
    sudo vim /etc/fstab
    ```

## 2. Port
  - Master
    - 6443: Kubernetes API Server / Used By All
    - 2379~2380: etcd server client API / Used By kube-apiserver, etcd
    - 10250: Kubelet API / Used By Self, Control plane
    - 10251: kube-scheduler / Used By Self
    - 10252: kube-controller-manager / Used By Self
  - Worker
    - 10250: Kubelet API / Used By Self, Control plane
    - 30000~32767: NodePort Services / Used By All

## 3. Docker setup
  - Install docker
    ```sh
    # 패키지 관리 도구 업데이트
    sudo apt update
    sudo apt-get update
    # 기존 docker 설치된 리소스 확인 후 발견되면 삭제
    sudo apt-get remove docker docker-engine docker.io
    # docker를 설치하기 위한 각종 라이브러리 설치
    sudo apt-get install apt-transport-https ca-certificates curl software-properties-common -y
    # curl 명령어를 통해 gpg key 내려받기
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    # 키를 잘 내려받았는지 확인
    sudo apt-key fingerprint 0EBFCD88
    # 패키지 관리 도구에 도커 다운로드 링크 추가
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    # 패키지 관리 도구 업데이트
    sudo apt-get update
    # docker-ce의 버젼을 최신으로 사용하는게 아니라 18.06.2~3의 버젼을 사용하는 이유는 
    # kubernetes에서 권장하는 버젼의 범위가 최대 v18.09 이기 때문이다.
    sudo apt-get install docker-ce=18.06.2~ce~3-0~ubuntu -y
    # Docker 설치 완료 후 테스트로 hello-world 컨테이너 구동
    sudo docker run hello-world
    ```
  - Use systemd instead of cgroupfs
    ```sh
    sudo cat > /etc/docker/daemon.json <<EOF
    {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m"
      },
      "storage-driver": "overlay2"
    }
    EOF

    sudo mkdir -p /etc/systemd/system/docker.service.d
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```

## 4. Kubernetes setup
  - Install k8s
    ```sh
    sudo apt-get update
    sudo apt-get upgrade

    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF 

    sudo apt-get update

    sudo apt-get install -y kubelet kubeadm kubectl

    # 패키지가 자동으로 설치, 업그레이드, 제거되지 않도록 hold함.
    sudo apt-mark hold kubelet kubeadm kubectl

    # 설치 완료 확인
    kubeadm version
    kubelet --version
    kubectl version
    ```
## 5. Create & Run Master node
  - Configurate Pod network 
    - Flannel이라는 애드온을 설치할 예정이므로 --pod-network-cidr=10.244.0.0/16사용
  - Configurate API server address
    - enp0s8 이더넷 주소 사용
  - Create & Run Master node
    ```sh
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.99.102
    ``` 
  - 정상 완료 시 다음과 같은 출력을 볼 수 있음
    ```sh
    [init] Using Kubernetes version: v1.16.3
    [preflight] Running pre-flight checks
      [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at <https://kubernetes.io/docs/setup/cri/>
      [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 19.03.4. Latest validated version: 18.09
    [preflight] Pulling images required for setting up a Kubernetes cluster
    [preflight] This might take a minute or two, depending on the speed of your internet connection
    ...
    ...
    [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy
    Your Kubernetes control-plane has initialized successfully!
    To start using your cluster, you need to run the following as a regular user:
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      <https://kubernetes.io/docs/concepts/cluster-administration/addons/>
    Then you can join any number of worker nodes by running the following on each as root:
    kubeadm join 192.168.99.102:6443 --token fnbiji.5wob1hu12wdtnmyr \
        --discovery-token-ca-cert-hash sha256:701d4da5cbf67347595e0653b31a7f6625a130de72ad8881a108093afd06188b
    ```
  - Use cluster
    ```sh
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -i):$(id -g) $HOME/.kube/config
    ```
  - Check
    ```sh
    kubectl get nodes
    ```
  - 다른 컴퓨터에서 클러스터와 통신할 시
    ```sh
    scp ./admin.conf $USER@$IP:/etc/kubernetes/admin.conf
    kubectl --kubeconfig ./admin.conf get nodes
    ```
  - 클러스터 외부에서 마스터 노드의 API서버 연결 시
    ```sh
    scp ./admin.conf $USER@$IP:/etc/kubernetes/admin.conf
    kubectl --kubeconfig ./admin.conf proxy
    ```
  - Deploy Pod network (Flannel)
    ```sh
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    ```

## 6. Worker node setting
  - Join
    ```sh
    # 마스터 노드 생성 후 맨 아래에 다음과 같은 메시지가 출력된다.
    # 다음 명령어를 워커노드에서 실행
    kubeadm join 192.168.99.102:6443 --token fnbiji.5wob1hu12wdtnmyr \
        --discovery-token-ca-cert-hash sha256:701d4da5cbf67347595e0653b31a7f6625a130de72ad8881a108093afd06188b
    ```
  - token 분실 시
    ```sh
    kubeadm token list
    ```
  - hash 분실 시
    ```sh
    openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
    ```
  - token 만료 시
    ```sh
    kubeadm token create
    ```
  - 완료 후 Master node에서 확인
    ```sh
    kubectl get nodes
    ```

## 7. Hello world 배포
  - Hello world 샘플 프로젝트는 deployment.yaml을 따로 작성하지 않는다.
    ```sh
    kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
    ```
  - Deployment 확인
    ```sh
    kubectl get deployments
    ```
  - Pod 확인
    ```sh
    # Pod의 IP는 worker의 IP이다.
    kubectl get pods -o wide
    ```
  - 통신
    ```sh
    curl http://10.244.1.2:8080
    ```