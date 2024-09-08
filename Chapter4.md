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


**[목차](./README.md#목차)**  
