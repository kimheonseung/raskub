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


**[목차](./README.md#목차)**  
