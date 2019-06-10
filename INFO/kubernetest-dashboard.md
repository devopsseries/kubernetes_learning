# Konfiguracja dostępu do dashboardu

### Dostęp do dashboard (GUI)

- zalogowanie się do klastra
- zmiana namespaces na kube-system
```
kubectl config set-context $(kubectl config current-context) --namespace=kube-system
```
- zmiana typu servisu kubernetes-dashboard
```
kubectl edit svc kubernetes-dashboard

spec:
  clusterIP: 10.7.249.103
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 31541
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: LoadBalancer
```

### Utworzenie użytkownika do dashboard

```
pliku dash-user.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
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
  namespace: kube-system
```

Uruchamiamy komendę
```
kubectl create -f dash-user.yaml
```
### Wyświetlenie tokena:

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
