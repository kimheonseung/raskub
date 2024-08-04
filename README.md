# RASKUB

라즈베리 파이를 활용하여 쿠버네티스 클러스터 구축해보기  
**RAS**pberrypi **KUB**ernetes &rarr; RASKUB  

# 목차
[0. 배경](#0-배경)  
[1. 하드웨어 구성](#1-하드웨어-구성)  
[2. k3s 설치 및 클러스터 세팅](#2-k3s-설치-및-클러스터-세팅)  
[3. Master Node에 helm 설치](#3-master-node에-helm-설치)  
[4. kubernetes dashboard 구성](#4-kubernetes-dashboard-구성)  
[5. plex media server 구성](#5-plex-media-server-구성)  
[6. ArgoCD 구성](#6-argocd-구성)  
[7. hello-world 어플리케이션 배포해보기](#7-hello-world-어플리케이션-배포해보기)  


# 0. 배경
* 3대의 라즈베리 파이를 다음 용도로 활용 중이었다.
  * Plex Media Server
  * NFS
  * 개발 공부에 필요한 인프라 등
* 점점 여러 대의 디바이스를 관리하기도 귀찮고, 문득 쿠버네티스 환경으로 이전을 해보고 싶다는 생각이 들었다.
* 기기가 넉넉하진 않지만 3대면 클러스터링을 해볼 수 있지 않을까?  
  
  
# 1. 하드웨어 구성
1대의 마스터노드와 2대의 워커노드로 구성한다.
|  |Master Node|Worker Node|  
|--|-----------|-----------|  
|모델|Raspberry Pi 4|Raspberry Pi 5|  
|RAM|4G|8G|
|수량|1ea|2ea|

### 노드별 호스트 네임과 네트워크 설정
* 아이피는 유/무선 아이피를 고정 할당 한다.
* 호스트 네임은 ```rpi```를 prefix, ```.local```을 suffix로 한다.
  * 라즈베리파이 세대와 해당 노드의 순서를 조합하여 생성한다.
  * ex. Worker Node 1은 라즈베리파이 5세대이고 첫번째 노드이므로 ```rpi51.local```

|  |호스트 네임|유선 (eth0)|무선 (wlan0)|
|--|-------|---|---|
|Master Node|rpi41.local|172.30.1.30|172.30.1.20|
|Worker Node 1|rpi51.local|172.30.1.31|172.30.1.21|
|Worker Node 2|rpi52.local|172.30.1.32|172.30.1.22|

```mermaid
---
title: "구성도"
---
flowchart LR
  MODEM("모뎀")
  ROUTER("라우터 (공유기)")
  SWITCH("스위치 허브")
  RPI41("Master Node\nrpi41.local")
  RPI51("Worker Node 1\nrpi51.local")
  RPI52("Worker Node 2\nrpi52.local")
  ME("내 장비")

  subgraph OUTER["외부 세상"]
    direction LR
    GOOGLE
    NAVER
    ...
  end

  subgraph HOMENET["홈 네트워크"]
    MODEM
    ROUTER
    SWITCH
    subgraph K8S["Kubernetes Cluster"]
      RPI41
      RPI51
      RPI52
    end
    ME
  end

  OUTER <-.-> MODEM
  MODEM <--> ROUTER
  ROUTER -.- SWITCH
  SWITCH -.- |"172.30.1.30\n172.30.1.20"| RPI41
  SWITCH -.- |"172.30.1.31\n172.30.1.21"| RPI51
  SWITCH -.- |"172.30.1.32\n172.30.1.22"| RPI52
  ME <-.-> ROUTER
```
  
### 결과물
<img src="./images/hardware/cluster.png" style="display: block; margin: 0 auto" width="60%" title="결과물">  
  
     
# 2. k3s 설치 및 클러스터 세팅
### 공통 세팅
* cgroup 설정
  * 컨테이너별 리소스 사용 배분을 위해 설정한다.
    ```bash
    $ sudo vi /boot/firmware/cmdline.txt
    ... cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
    ```
* swap off
  * 디스크 I/O 성능 저하 발생
  * cgroup 외에도 메모리 관여 포인트가 늘어남
    ```bash
    # 임시 off
    $ sudo swapoff -a
    # 영원히 off
    $ sudo vi /etc/dphys-swapfile
    CONF_SWAPSIZE=0
    ```
* 일단 무선랜 사용 off
  ```bash
  # 리부팅마다 해줘야 함
  $ sudo ifconfig wlan0 down
  $ sudo rfkill block $WLAN_ID
  ```
* apt 업데이트 / 업그레이드
  ```bash
  $ sudo apt update
  $ sudo apt upgrade
  ```
* hostname 등록
  ```bash
  # /etc/hosts
  172.30.1.20     rpi41.local
  172.30.1.30     rpi41.local

  172.30.1.21     rpi51.local
  172.30.1.31     rpi51.local

  172.30.1.22     rpi52.local
  172.30.1.32     rpi52.local
  ```

---
### k3s 설치
* 여러 설치 옵션이 있지만, 일단 기본값으로 설치한다.
  * ingress controller나 flannel backend 등의 [옵션](https://docs.k3s.io/kr/installation/configuration)을 커스터마이징 할 수 있음

* Master Node (rpi41.local)
  ```bash
  $ curl -sfL https://get.k3s.io | sh -

  # 설치 완료 후 노드 토큰 조회
  $ sudo cat /var/lib/rancher/k3s/server/node-token
  K10251e60ba702f29456c2c2d49031a752f03eb6014b134dddc2c368d1b4bcb1c36::server:278c5377975fed99a9c7e142c75602b3
  ```

* Worker Node (rpi51.local, rpi52.local)
  ```bash
  $ sudo vi /etc/profile
  # k3s
  K3S_URL="https://rpi41.local:6443"
  K3S_TOKEN="K10251e60ba702f29456c2c2d49031a752f03eb6014b134dddc2c368d1b4bcb1c36::server:278c5377975fed99a9c7e142c75602b3"
  export K3S_URL
  export K3S_TOKEN

  $ curl -sfL https://get.k3s.io | K3S_URL=$K3S_URL K3S_TOKEN=$K3S_TOKEN sh -
  ```

# 3. Master Node에 helm 설치
* helm 설치
  * https://helm.sh/docs/intro/install/ 참고
    ```bash
    $ curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
    $ sudo apt-get install apt-transport-https --yes
    $ echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    $ sudo apt-get update
    $ sudo apt-get install helm
    ```
* 이슈
  * k3s와 helm이 사용하는 설정파일 문제
    |  |k3s|helm|
    |--|---|----|
    |참조 설정|/etc/rancher/k3s/k3s.yaml|$HOME/.kube/config|
    * 사용자 권한으로 실행하는 경우
      * k3s이 참조하는 설정 파일은 읽기 권한이 없음
      * group-readable, world-readable 설정하면 보안 취약
      * helm이 참조하는 설정 파일에 k3s가 참조하는 설정파일을 복사하여 사용한다.
      * 다음 명령어로 readable 파일을 지정하여 실행이 가능
        ```bash
        $ helm --kubeconfig $KUBE_CONFIG_PATH [COMMAND]...
        ```
      * 또는 helm은 ```KUBECONFIG``` 환경변수를 인식하므로 ```bashrc``` 등에 넣어둔다.
        ```bash
        $ export KUBECONFIG=[CONFIG_PATH]
        ```
  * 내가 적용한 방법
    * 사용자의 경우 (설정이 변경될 때 마다 동기화가 필요함..)
      ```bash
      $ cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
      $ echo 'export KUBECONFIG="$HOME/.kube/config"' >> ~/.bashrc 
      $ source ~/.bashrc
      ```
    * 관리자의 경우
      ```bash
      $ echo 'export KUBECONFIG="/etc/rancher/k3s/k3s.yaml"' >> ~/.bashrc 
      $ source ~/.bashrc
      ```

# 4. kubernetes dashboard 구성
### Kubernetes Dashboard 설치
* [Kubernetes Dashboard](https://github.com/kubernetes/dashboard)
  ```bash
  # Add kubernetes-dashboard repository
  $ helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
  # Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart
  $ helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
  Release "kubernetes-dashboard" does not exist. Installing it now.
  NAME: kubernetes-dashboard
  LAST DEPLOYED: Sat Jul  6 14:05:51 2024
  NAMESPACE: kubernetes-dashboard
  STATUS: deployed
  REVISION: 1
  TEST SUITE: None
  NOTES:
  *************************************************************************************************
  *** PLEASE BE PATIENT: Kubernetes Dashboard may need a few minutes to get up and become ready ***
  *************************************************************************************************
  
  Congratulations! You have just installed Kubernetes Dashboard in your cluster.
  
  To access Dashboard run:
    kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
  
  NOTE: In case port-forward command does not work, make sure that kong service name is correct.
        Check the services in Kubernetes Dashboard namespace using:
          kubectl -n kubernetes-dashboard get svc
  
  Dashboard will be available at:
    https://localhost:8443
  ```
* 배포 확인
  ```bash
  $ kubectl -n kubernetes-dashboard get svc
  NAME                                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
  kubernetes-dashboard-api               ClusterIP   10.43.96.66     <none>        8000/TCP                        127m
  kubernetes-dashboard-auth              ClusterIP   10.43.155.232   <none>        8000/TCP                        127m
  kubernetes-dashboard-kong-manager      NodePort    10.43.93.22     <none>        8002:32080/TCP,8445:32408/TCP   127m
  kubernetes-dashboard-kong-proxy        ClusterIP   10.43.188.188   <none>        443/TCP                         127m
  kubernetes-dashboard-metrics-scraper   ClusterIP   10.43.40.83     <none>        8000/TCP                        127m
  kubernetes-dashboard-web               ClusterIP   10.43.65.37     <none>        8000/TCP                        127m
  ```

### Kubernetes Dashboard NodePort로 변경
* Kubernetes Dashboard는 기본적으로 ClusterIP로 배포된다.
* 외부에서 접근할 수 있도록 NodePort로 변경하고 HTTP 연결을 허용해본다.

* Kubernetes Dashboard는 자체적으로 kong-proxy를 사용하고 있어서, 이쪽 설정을 변경해줘야 한다.
  ```yaml
  # kong-values.yaml
  kong:
    proxy:
      type: NodePort
    http:
      enabled: true
  ```
  ```bash
  # 위 설정값으로 helm 갱신
  $ helm upgrade kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -f kong-values.yaml -n kubernetes-dashboard
  Release "kubernetes-dashboard" has been upgraded. Happy Helming!
  NAME: kubernetes-dashboard
  LAST DEPLOYED: Sun Jul  7 10:51:34 2024
  NAMESPACE: kubernetes-dashboard
  STATUS: deployed
  REVISION: 2
  TEST SUITE: None
  NOTES:
  *************************************************************************************************
  *** PLEASE BE PATIENT: Kubernetes Dashboard may need a few minutes to get up and become ready ***
  *************************************************************************************************
  
  Congratulations! You have just installed Kubernetes Dashboard in your cluster.
  
  To access Dashboard run:
    kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
  
  NOTE: In case port-forward command does not work, make sure that kong service name is correct.
        Check the services in Kubernetes Dashboard namespace using:
          kubectl -n kubernetes-dashboard get svc
  
  Dashboard will be available at:
    https://localhost:8443
  
  $ k get svc -n kubernetes-dashboard
  NAME                                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
  kubernetes-dashboard-api               ClusterIP   10.43.96.66     <none>        8000/TCP                        20h
  kubernetes-dashboard-auth              ClusterIP   10.43.155.232   <none>        8000/TCP                        20h
  kubernetes-dashboard-kong-manager      NodePort    10.43.93.22     <none>        8002:32080/TCP,8445:32408/TCP   20h
  kubernetes-dashboard-kong-proxy        NodePort    10.43.188.188   <none>        443:32414/TCP                   20h
  kubernetes-dashboard-metrics-scraper   ClusterIP   10.43.40.83     <none>        8000/TCP                        20h
  kubernetes-dashboard-web               ClusterIP   10.43.65.37     <none>        8000/TCP                        20h  
  ```
* helm 갱신 후 revision이 2가 되고, proxy 서비스가 NodePortfh 변경된 것을 확인할 수 있다.
* 32414 포트로 https 접속하면 대시보드 접근이 가능하다.
* 로그인 페이지가 나오는데, 관련 sa, crb 등을 생성해야 한다.
  <img src="./images/kubernetes-dashboard/dashboard-login.png" style="display: block; margin: 0 auto" width="60%" title="로그인">
  ```yaml
  # users.yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: admin-user
    namespace: kubernetes-dashboard
  
  ---
  
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: admin-user
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-admin
  subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
  
  ---
  
  apiVersion: v1
  kind: Secret
  metadata:
    name: admin-user
    namespace: kubernetes-dashboard
    annotations:
      kubernetes.io/service-account.name: "admin-user"
  type: kubernetes.io/service-account-token
  ```
  ```bash
  $ kubectl apply -f users.yaml
  $ kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
  eyJhbGciOiJSUzI1NiIsImtpZCI6ImVBQk9LV2lFSEdpWi1aVE1RRjd5ZHpzTTdOcGxnY3ZLbUUyR1F0ejcxOE0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJhNDY0MmZhYy1jMTM2LTRlN2UtYmFiZi04OGMxMzI1M2E3ZTMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.gYHrFYJKrjBvlnIKZ3v2wfmsBBy0yxY5lrvR4zspiaj2pNsYxW2DdcXBWM5SC8Q_mkuWqPuOSg4lJstiMV09PNf5ukv9seGV_cnEUsTfijjsnZPXd7ubMCZtk5mx-bZ9jofxbgqc0iZnSqz6iYo3G2zrnliDi0BAlN_dSRZq435J1Lw7QOMDcovWxvLLODy1mdUC4bWVaAg_HtfaX81jqEyEcvVoIfBN5DyHGMDCKQseG_Tn3ebZ2GLh0U4hOG5fdplgaVoGRPine5cGtfLjnZuM0DBjyyfsAt_aH1X2lOmA_ydhQRVoWAL9PRATjeCnRBCP0vG-nmQeM4iY7_H5JA
  
  # 위 토큰으로 대시보드 접근하기
  ```
* 다음 방법으로 서비스를 하나 더 두어 대시보드를 NodePort로 노출이 가능하지만.. 
    * 불필요한 서비스가 하나 더 생성되고
    * proxy 서비스를 이용하지 않아 안티패턴 같다.
  ```yaml
  # kubernetes-dashboard-kong.yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: kubernetes-dashboard-kong-nodeport
    namespace: kubernetes-dashboard
  spec:
    ports:
    - name: kong-proxy-tls
      nodePort: 32001 # Your desired port
      port: 443
      protocol: TCP
      targetPort: 8443
    selector:
      app.kubernetes.io/component: app
      app.kubernetes.io/instance: kubernetes-dashboard
      app.kubernetes.io/name: kong
    type: NodePort
  ```

# 5. plex media server 구성
### PVC로 활용할 NFS 서버 구축
* Plex에서 사용할 PVC를 위해 워커노드 중 한대를 NFS Server로 사용한다.
* ```rpi51.local``` 서버를 NFS 서버로 설정해본다.
  ```bash
  # nfs-kernel-server 설치
  $ sudo apt install nfs-kernel-server -y
  
  # 공유 디렉토리 생성
  $ sudo mkdir -p /mnt/nfsshare
  
  # 권한 설정
  # 공유 디렉토리의 소유자/그룹 설정
  $ sudo chown -R rpi51:rpi51 /mnt/nfsshare
  # 공유 디렉토리 하위의 모든 디렉토리에 755 권한 부여
  $ sudo find /mnt/nfsshare/ -type d -exec chmod 755 {} \;
  # 공유 디렉토리 하위의 모든 파일에 644 권한 부여
  $ sudo find /mnt/nfsshare/ -type f -exec chmod 644 {} \;
  
  # 현 사용자의 uid, gid 조회
  $ id rpi51
  uid=1000(rpi51) gid=1000(rpi51) ...
  
  # NFS 접근 관련 파일, 디렉토리 설정
  $ sudo vi /etc/exports
  # 아래 설정을 입력 - 
  /mnt/nfsshare *(rw,all_squash,insecure,async,no_subtree_check,anonuid=1000,anongid=1000)
  # 저장~
  
  # 새로운 폴더를 공유로 추가하였으므로 exportfs 명령어를 통해 갱신한다.
  $ sudo exportfs -ra
  # nfs-kernel-server 리스타트
  $ sudo systemctl restart nfs-kernel-server
  ```
### Plex Media Server arm64용 이미지 빌드
* 공식 이미지 레지스트리에서는 linux/amd64 이미지만 제공한다.
* 라즈베리파이는 arm64 아키텍처용 이미지가 필요하다.
* 개인 이미지 레지스트리에 일단 하나 생성해둔다.
  ```bash
  $ docker pull gpoleze/pms-docker:arm64v8-2022-09-25
  $ docker login
  $ docker tag gpoleze/pms-docker:arm64v8-2022-09-25 khs920210/pms-docker:arm64v8-2022-09-25
  $ docker push khs920210/pms-docker:arm64v8-2022-09-25
  ```

### Plex Media Server 배포
**PV 및 PVC 생성**
* 하나의 파드만 사용할 예정이라 ```storage = capacity```로 설정한다.
  ```yaml
  # nfs-pv.yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: nfs-pv
    labels:
      type: nfs
  spec:
    capacity:
      storage: 100Gi
    accessModes:
      - ReadWriteMany
    nfs:
      server: rpi51.local
      path: /mnt/nfsshare
  
  # nfs-pvc.yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: nfs-pvc
  spec:
    storageClassName: ""
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 100Gi
    selector:
      matchLabels:
        type: nfs
  ```
  ```bash
  $ kubectl apply nfs-pv.yaml
  $ kubectl apply nfs-pvc.yaml
  ```
  ```bash
  # 예시 설정값들 확인
  $ helm show values plex/plex-media-server > values.yaml
  ```
  ```yaml
  # values.yaml
  # 이미지 지정
  image:
    registry: index.docker.io
    repository: khs920210/pms-docker
    # If unset use "latest"
    tag: "arm64v8-2022-09-25"
    sha: ""
    pullPolicy: IfNotPresent
  
  # pvc를 연동
  extraVolumeMounts:
    - name: nfs-volume
      mountPath: /data/nfs
      readOnly: true
  extraVolumes:
    - name: nfs-volume
      persistentVolumeClaim:
        claimName: nfs-pvc
        
  # NodePort로 오픈
  service:
    type: NodePort
    port: 32400
  
    # Port to use when type of service is "NodePort" (32400 by default)
    # nodePort: 32400
  
    # optional extra annotations to add to the service resource
    annotations: {}
  ```
  ```bash
  # 위 설정값으로 배포
  $ helm repo add plex https://raw.githubusercontent.com/plexinc/pms-docker/gh-pages
  $ helm upgrade --install plex plex/plex-media-server --values values.yaml
  ```

### Plex 초기 설정 진행
* 반드시 배포된 호스트의 IP로 들어가야 초기 설정을 진행할 수 있음
  * 호스트 네임으로 접근시 설정이 불가능
  * 파드가 배포된 워커노드의 호스트 IP로 접근하기
  * 배포된 파드의 호스트 워커노드 조회해보기
```bash
$ kubectl get pods -o wide
NAME                       READY   STATUS    RESTARTS   AGE     IP           NODE    NOMINATED NODE   READINESS GATES
plex-plex-media-server-0   1/1     Running   0          9m56s   10.42.2.58   rpi52   <none>           <none>

# rpi52의 호스트IP로 접근하기
```

# 6. ArgoCD 구성
Wrting...

# 7. hello-world 어플리케이션 배포해보기
Wrting...
