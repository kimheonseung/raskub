# RASKUB

라즈베리 파이를 활용하여 쿠버네티스 클러스터 구축해보기  
**RAS**pberrypi **KUB**ernetes &rarr; RASKUB  

# 목차
[0. 배경](./Chapter0.md)  
[1. 하드웨어 구성](./Chapter1.md)  
[2. k3s 설치 및 클러스터 세팅](./Chapter2.md)  
[3. Master Node에 helm 설치](./Chapter3.md)  
[4. kubernetes dashboard 구성](./Chapter4.md)  
[5. plex media server 구성](./Chapter5.md)  
[6. ArgoCD 구성](./Chapter6.md)  
[7. hello-world 어플리케이션 배포해보기](./Chapter7.md)  
[8. PostgreSQL 배포해보기 ](./Chapter8.md)  
[9. ConfigMap, Secret으로 Spring 어플리케이션 설정 주입](./Chapter9.md)


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
  RPI41("Master Node
  rpi41.local")
  RPI51("Worker Node 1
  rpi51.local")
  RPI52("Worker Node 2
  rpi52.local")
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
  SWITCH -.- |"172.30.1.30
  172.30.1.20"| RPI41
  SWITCH -.- |"172.30.1.31
  172.30.1.21"| RPI51
  SWITCH -.- |"172.30.1.32
  172.30.1.22"| RPI52
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
### 설치
공식 홈페이지에서 제공하는 방법으로 설치한다.
```bash
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 설치 확인
$ kubectl get po -n argocd
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          77s
argocd-applicationset-controller-65bb5ff89-bdzjs   1/1     Running   0          78s
argocd-dex-server-69b469f8fb-24rfd                 1/1     Running   0          78s
argocd-notifications-controller-64bc7c9f7-h2l86    1/1     Running   0          78s
argocd-redis-867d4785f-l5cs2                       1/1     Running   0          78s
argocd-repo-server-5744559fff-jq2zf                1/1     Running   0          77s
argocd-server-697df9f478-55t9b                     1/1     Running   0          77s

# 서비스 확인
$ kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.43.24.7      <none>        7000/TCP,8080/TCP            4m31s
argocd-dex-server                         ClusterIP   10.43.229.25    <none>        5556/TCP,5557/TCP,5558/TCP   4m31s
argocd-metrics                            ClusterIP   10.43.57.140    <none>        8082/TCP                     4m31s
argocd-notifications-controller-metrics   ClusterIP   10.43.144.66    <none>        9001/TCP                     4m31s
argocd-redis                              ClusterIP   10.43.48.223    <none>        6379/TCP                     4m31s
argocd-repo-server                        ClusterIP   10.43.200.201   <none>        8081/TCP,8084/TCP            4m31s
argocd-server                             ClusterIP   10.43.171.78    <none>        80/TCP,443/TCP               4m31s
argocd-server-metrics                     ClusterIP   10.43.87.97     <none>        8083/TCP                     4m31s

# 초기 비밀번호 확인
$ kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
VbyJsVX034MMMil1
```

### Ingress 설정
* argocd-server 서비스가 ClusterIP 형태이므로 해당 서비스에 접근할 수 있도록 별도 Ingress를 설정한다.
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
    - host: rpi.k8s.argocd
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
```
* 이후 계정 초기화는 UI로 접근하여 진행


# 7. hello-world 어플리케이션 배포해보기
### hello-world 어플리케이션
https://github.com/kimheonseung/hello-world
```mermaid
---
title: Hello World
---
flowchart TB

    subgraph IMAGE_REGISTRY["Image Registry"];
        subgraph DOCKERHUB["Docker Hub"]
            IMAGE("khs920210/hello-world")
        end
    end
    
    subgraph GITHUB["Github"]
        direction TB
        subgraph APP["hello-world"]
            subgraph DEPLOY["Deployments"]
            end
            subgraph WORKFLOW["CI/CD Workflow"]
            end
        end
    end
    
    subgraph K8S["Kubernetes Cluster"]
        subgraph GIT_OPS["Git Ops"]
            ARGOCD("Argo CD")
        end
        subgraph APPS["Applications"]
            HELLO_WORLD("hello-world")
        end
    end

    IMAGE -.-> |"Image pull"| ARGOCD
    DEPLOY -.-> |"deploy yaml"| ARGOCD
    ARGOCD -.-> |"Sync"| HELLO_WORLD
    WORKFLOW -.-> |"Build and push"| IMAGE
    WORKFLOW -.-> |"Update new image tag"| DEPLOY
    
    click IMAGE href "https://hub.docker.com/r/khs920120/hello-world/tags" "dockerhub image tags"
    click DEPLOY href "https://github.com/kimheonseung/hello-world/blob/main/deploy/manifest/deployment.yaml#L26" "deployments yaml"
    click WORKFLOW href "https://github.com/kimheonseung/hello-world/blob/main/.github/workflows/ci-cd-hello-world.yaml" "ci/cd yaml"
```

### ArgoCD에 레파지토리 등록
<img src="./images/argocd/new-app.png" style="display: block; margin: 0 auto" width="60%" title="new app">

### ArgoCD 배포 확인
<img src="./images/argocd/deployed.png" style="display: block; margin: 0 auto" width="60%" title="배포 확인">


# 8. PostgreSQL 배포해보기 
### postgres-operator
- 고가용성과 자동 페일오버 등 고급 기능을 제공하는 오퍼레이터를 사용해본다.
- 오버스펙이지만.. 다큐먼트가 잘 되어있고, 커스텀한 설정을 많이 만져볼 수 있는 오퍼레이터로 골라보려 한다.
- 선택지는 크게 두 가지가 있다.
  1. [bitnami](https://github.com/bitnami/charts/tree/main/bitnami/postgresql)
      - 설정이 간단하고 개발 환경에 적합한 경량 오퍼레이터
      - 마스터-리플리카 구조 설정 가능
      - 고가용성 요구에는 적합하지 않음
        - 자동 페일오버 기능이 없어 마스터 다운시 수동 대응해야함
  2. [zalando](https://github.com/zalando/postgres-operator)
      - 설정이 복잡하지만 고가용성, 자동 페일오버 등 지원
      - 프로덕션 환경에 적합
      - [Patroni](https://github.com/patroni/patroni?tab=readme-ov-file#patroni-a-template-for-postgresql-ha-with-zookeeper-etcd-or-consul), etcd를 활용한 자동 페일오버, 마스터 선출 등 지원   
- zalando로 결정

### 알게된 사실
- zalando operator는 내부적으로 patroni 라는 모듈을 사용하여 failover 상황에서 동적으로 Master 인스턴스를 선정한다.
- 다수의 인스턴스를 클러스터로 생성할 수 있다.
  - LoadBalancer 타입의 서비스 포트를 static 하게 설정할 수 없다.
  - 볼륨은 기존에 존재하는 PVC에 연결하려 했지만 StorageClass만 지정 가능했다.
    - 아마 DB 인스턴스가 다수로 구성될 때 각자가 볼륨을 가지려면... static 한 PVC 이용을 제한하고 StorageClass를 통한 동적 할당을 수행하는 것이 추후 유연한 관리가 가능해서 아닐까..
    - 기본으로 local-path StorageClass에 할당되도록 했다.


### 구성
- Master / Replica 구조로 구성한다.
- 
  |  |갯수|용도|
  |--|---|---|
  |Master|1|Read / Write|
  |Replica|1|Read|
- 클러스터 이름은 ```raskub-pg-cluster```
- ```raskub-rw``` 유저는 Master Read/Write 수행
- ```raskub-ro``` 유저는 Replica Read 수행
- Master, Replica 모두 LoadBalancer 타입으로 배포

### 배포
1. helm 레포지토리 추가
    - postgres-operator 배포
      ```bash
      # helm repo 추가
      $ helm repo add postgres-operator https://opensource.zalando.com/postgres-operator/charts/postgres-operator

      # helm repo 갱신
      $ helm repo update
      ```
2. postgres-operator 배포
    ```bash
    $ helm install postgres-operator postgres-operator/postgres-operator --namespace postgres-operator --create-namespace

    # operator pod 확인
    $ kubectl --namespace=postgres-operator get pods -l "app.kubernetes.io/name=postgres-operator"
    ```
3. postgres 클러스터 구성
    ```yaml
    # postgres-cluster.yaml
    apiVersion: "acid.zalan.do/v1"
    kind: postgresql
    metadata:
      name: raskub-pg-cluster
      namespace: default
    spec:
      teamId: "raskub"
      numberOfInstances: 2
      users:
        raskub-rw:
          - superuser
          - createdb
        raskub-ro: []
      databases:
        raskub: raskub-rw
      postgresql:
        version: "15"
        parameters: 
          shared_buffers: "32MB"
          max_conections: "10"
          log_statement: "all"
            #      password_encryption: scram-sha-256
      volume:
        size: 10Gi
      enableMasterLoadBalancer: true
      enableReplicaLoadBalancer: true
      preparedDatabases:
        raskub:
          defaultUsers: false
      allowedSourceRanges:
        - 0.0.0.0/0  # Allows external connections from any IP (can be limited)
      resources:
        requests:
          cpu: 500m
          memory: 500Mi
        limits:
          cpu: 500m
          memory: 500Mi
    ```
4. 클러스터 배포
    ```bash
    $ kubectl apply -f postgres-cluster.yaml
    ```
5. 확인
    ```bash
    # statefulset 확인
    $ kubectl get statefulset raskub-pg-cluster
    NAME                READY   AGE
    raskub-pg-cluster   2/2     21h

    # statefulset에 해당하는 라벨 조회
    $ kubectl get statefulset raskub-pg-cluster -o jsonpath='{.spec.selector.matchLabels}'
    {"application":"spilo","cluster-name":"raskub-pg-cluster"}

    # service 확인
    $ kubectl get svc -l cluster-name=raskub-pg-cluster
    NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP                                       PORT(S)          AGE
    raskub-pg-cluster          LoadBalancer   10.43.202.93   192.168.219.120,192.168.219.121,192.168.219.122   5432:30614/TCP   22h
    raskub-pg-cluster-config   ClusterIP      None           <none>                                            <none>           22h
    raskub-pg-cluster-repl     LoadBalancer   10.43.91.10    <pending>                                         5432:31779/TCP   22h

    # secret 확인
    $ kubectl get secret -l cluster-name=raskub-pg-cluster
    postgres.raskub-pg-cluster.credentials.postgresql.acid.zalan.do    Opaque   2      23h
    raskub-ro.raskub-pg-cluster.credentials.postgresql.acid.zalan.do   Opaque   2      23h
    raskub-rw.raskub-pg-cluster.credentials.postgresql.acid.zalan.do   Opaque   2      23h
    standby.raskub-pg-cluster.credentials.postgresql.acid.zalan.do     Opaque   2      23h

    # 유저 secret 조회 
    $ kubectl get secret raskub-rw.raskub-pg-cluster.credentials.postgresql.acid.zalan.do -o 'jsonpath={.data.password}' | base64 -d
    $ kubectl get secret raskub-ro.raskub-pg-cluster.credentials.postgresql.acid.zalan.do -o 'jsonpath={.data.password}' | base64 -d
    ```


# 9. ConfigMap, Secret으로 Spring 어플리케이션 설정 주입
### [Spring Cloud Kubernetes](https://spring.io/projects/spring-cloud-kubernetes)
- K8S 환경에서 Spring 어플리케이션을 통합할 수 있게 해준다.
  - 서비스 레지스트리
    - 어플리케이션 설정에 namespace, service 명을 그대로 활용 가능
  - ConfigMap, Secret 통합
    - 어플리케이션 설정 및 환경변수에 ConfigMap, Secret 등의 값 주입할 수 있음
  - Health Probe 연동
    - livenessProbe, readinessProbe와 Spring Actuator 연동 가능
    - 연동을 하지 않으면 어플리케이션 컨테이너에 직접 접근이 어려워 프록시를 거쳐야함 (Ingress 등)
  - 그 외에도 로드밸런싱 등을 지원한다.

### To Do
  - ConfigMap, Secret 통합
  - Health Probe 연동
  - namespace 연동

### 라이브러리 추가
  ```gradle
  ...
  extra["springCloudVersion"] = "2023.0.3"
  ...
  dependencies {
      implementation("org.springframework.cloud:spring-cloud-starter-kubernetes-client")
      // health probe와 연동 확인
      implementation("org.springframework.boot:spring-boot-starter-actuator")
  }
  ...
  dependencyManagement {
      imports {
          mavenBom("org.springframework.cloud:spring-cloud-dependencies:${property("springCloudVersion")}")
      }
  }
  ```

### 주입받을 값을 출력하는 컨트롤러 정의
- ```cm.custom.message```
  - ConfigMap에서 가져올 설정값
- ```sec.custom.message```
  - Secret에서 가져올 설정값
```kotlin
import org.springframework.beans.factory.annotation.Value
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController
import java.time.LocalDateTime

@RestController
@RequestMapping("/hello-world")
class HelloWorldController(
    @Value("\${cm.custom.message}")
    val customMessage: String,
    @Value("\${sec.custom.message}")
    val customSecMessage: String,
) {
    @GetMapping
    fun getHelloWorld() = "hello! ${LocalDateTime.now()}. cmCustomMessage: $customMessage, customSecMessage: $customSecMessage"
}
```

### yaml 설정
  - Spring Cloud Kubernetes 라이브러리를 추가한 뒤 어플리케이션을 구동하면 kuberentes 프로파일로 구동한다.
    ```bash
    2024-09-23T01:50:01.199Z  INFO 1 --- [hello-world] [  main] c.k.h.HelloWorldApplicationKt   : The following 1 profile is active: "kubernetes"
    ```
  - 기본 프로파일용 yaml과 kubernetes 프로파일용 yaml 두가지를 설정해본다.
  - application.yaml  
    ```yaml
    spring:
      application:
        name: hello-world
    management:
      endpoints:
        web:
          exposure:
            include: health # health probe가 요청할 actuator path 설정
    ```
  - application-kubernetes.yaml  
    ```yaml
    spring:
      cloud:
        kubernetes:
          discovery:
            all-namespaces: false
            namespaces: raskub-app # 어플리케이션의 네임스페이스를 raskub-app으로 제한한다.  
    sec:
      custom:
        message: ${SECURITY_MESSAGE} # Secret에서 가져올 값을 명시한다.
    ```

### Manifest 작성
- Spring 어플리케이션 배포 상태를 정의한다.  
  1. ConfigMap, Secret을 정의한다.
  2. 위와 관련된 Role을 정의한다.
      - 특정 namespace 기준이라 Cluster Role 대신 Role 정의
  3. Service Account를 정의한다.
  3. 위 Role과 Service Account와 관련된 Role Binding을 정의한다.
  4. Service, Ingress를 정의한다.
  5. Deployment를 정의한다.

#### ConfigMap
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: hello-world-config
    namespace: raskub-app
  data:
    application.yaml: |-
      cm:
        custom:
          message: hello world from cm config
  ```
#### Secret
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: hello-world-secret
    namespace: raskub-app
  stringData:
    sec_message: hello world from secret config
  ```
#### Role
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: raskub-app
    name: hello-world-reader
  rules:
    - apiGroups: [""]
      resources: ["pods", "configmaps", "secrets", "services", "endpoints"]
      verbs: ["get", "watch", "list"]
  ```
#### Service Account
  ```yaml
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: hello-world-sa
    namespace: raskub-app
  ```
#### Service
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: hello-world
    namespace: raskub-app
    labels:
      app: hello-world
  spec:
    ports:
      - port: 8080
        protocol: TCP
        targetPort: 8080
    selector:
      app:  hello-world
    type: ClusterIP
  ```
#### Ingress
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: hello-world
    namespace: raskub-app
    annotations:
      kubernetes.io/ingress.class: "traefik"
  spec:
    rules:
      - host: rpi.k8s.hello-world
        http:
          paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: hello-world
                  port:
                    number: 8080
  ```
#### Deplyment
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: hello-world
    name: hello-world
    namespace: raskub-app
  spec:
    revisionHistoryLimit: 5
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 50%
        maxUnavailable: 0%
    selector:
      matchLabels:
        app: hello-world
    template:
      metadata:
        labels:
          app: hello-world
      spec:
        serviceAccount: hello-world-sa
        securityContext:
          fsGroup: 65534
        containers:
          - name: hello-world
            image: khs920210/hello-world:6
            imagePullPolicy: IfNotPresent
            env:
              - name: SECURITY_MESSAGE # Secret 값 주입
                valueFrom:
                  secretKeyRef:
                    name: hello-world-secret
                    key: sec_message
            ports:
              - name: http
                protocol: TCP
                containerPort: 8080
            resources:
              limits:
                memory: 500Mi
              requests:
                cpu: 300m
                memory: 500Mi
            readinessProbe: # Health Probe와 Actuator 연동
              httpGet:
                path: /actuator/health/readiness
                port: 8080
              initialDelaySeconds: 10
              periodSeconds: 10
            livenessProbe: # Health Probe와 Actuator 연동
              httpGet:
                path: /actuator/health/liveness
                port: 8080
              initialDelaySeconds: 120
              periodSeconds: 10
            volumeMounts:
              - name: config-volume
                mountPath: /opt/hello-world/config # ConfigMap 주입. volume은 따로 정의한다.
        volumes:
          - name: config-volume # ConfigMap Volume
            configMap:
              name: hello-world-config
  ```

### 확인
<img src="./images/spring-cloud-kubernetes/message.png" style="display: block; margin: 0 auto" width="100%" title="메시지 확인">

### Git Repo
https://github.com/kimheonseung/hello-world

  
