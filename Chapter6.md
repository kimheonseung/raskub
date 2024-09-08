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


**[목차](./README.md#목차)**  
