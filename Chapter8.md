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
