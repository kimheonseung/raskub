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


## **[목차](./README.md#목차)**  
