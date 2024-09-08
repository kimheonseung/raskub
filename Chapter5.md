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


**[목차](./README.md#목차)**  
