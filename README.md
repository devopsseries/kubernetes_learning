#maintenance *Jarosław Jeznach, Mirosław Kudła*

# kubernetes_learning

1. Utworzenie nowego projektu na Google Cloud Platform.
2. Utworzenie klastra Kubernetes na Google Cloud Platform.
3. Instalacja i konfiguracja GCloud SDK w celu połączenia się ze swoim klastrem.
4. Instalacja i konfiguracja aplikacji Księga Gości.

## Klonowanie repozytorium kodu.

git clone https://github.com/devopsseries/kubernetes_learning.git

## Utworzenie nowego namespace'u o nazwie "warsztaty"

## Tworzenie i zmiana namespace

Utworzenie nowego namespace'u o nazwie "warsztaty"
```
kubectl create ns warsztaty
```

Wylistowanie wszystkich namespaces
```
kubectl get ns
```

Przejście do nowo utworzonego namespace'u
```
kubectl config set-context $(kubectl config current-context) --namespace=warsztaty
```

## Uruchomienie aplikacji Guestbook z Redis

Aplikacja Guesbook używa Redis do przechowywania danych. Zapisuje swoje dane do głównej instancji Redis i odczytuje je z wielu instancji podrzędnych.

### Deployment Redis Master
W pierwszej kolejności uruchomimy Redis Master. Musimy utworzyć plik konfiguracyjny deploymentu wykonując komendę:

```
vi redis-master-deployment.yaml
```
Zawartość pliku będzie wyglądała następująco:

```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: redis-master
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
      role: master
      tier: backend
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: k8s.gcr.io/redis:e2e  # or just image: redis
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
```

Po utworzeniu pliku należy wykonać polecenie polecenie tworzące  obiekt deployment:

```
kubectl apply -f redis-master-deployment.yaml
```
Sprawdź czy pod został poprawnie utworzony:
```
kubectl get pod
```
Output powinien wyglądać następująco:
```
NAME                            READY     STATUS    RESTARTS   AGE
redis-master-1068406935-3lswp   1/1       Running   0          28s
```
Uruchom poniższe polecenie aby wyświetlić logi z poda Redis Master:
```
kubectl logs -f <nazwa_poda>
```

### Usługa Redis Master

Następnie należy utworzyć obiekt typu service Redis Master.
```
vi redis-master-service.yaml
```
Zawartość pliku będzie wyglądała następująco::
```
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    app: redis
    role: master
    tier: backend
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: master
    tier: backend
```
Tak jak poprzednio należy wykonać polecenie które utworzy obiekt:
```
kubectl apply -f redis-master-service.yaml
```
Sprawdź czy usługa została utworzona poprawnie:
```
kubectl get service
```
albo krócej:
```
kubectl get svc
```
Output powinien wyglądać podobnie do tego:
```
NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
redis-master   ClusterIP   10.0.0.151   <none>        6379/TCP   8s
```
### Uruchomienie redis slave. 

W celu utworzenia deploymentu dla redis slave Musimy utworzyć plik manifestu wykonując komendę: 
```
vi redis-slave-deployment.yaml
```
Zawartość pliku będzie wyglądała następująco:
```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: redis-slave
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
      role: slave
      tier: backend
  replicas: 2
  template:
    metadata:
      labels:
        app: redis
        role: slave
        tier: backend
    spec:
      containers:
      - name: slave
        image: gcr.io/google_samples/gb-redisslave:v1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # Using `GET_HOSTS_FROM=dns` requires your cluster to
          # provide a dns service. As of Kubernetes 1.3, DNS is a built-in
          # service launched automatically. However, if the cluster you are using
          # does not have a built-in DNS service, you can instead
          # access an environment variable to find the master
          # service's host. To do so, comment out the 'value: dns' line above, and
          # uncomment the line below:
          # value: env
        ports:
        - containerPort: 6379
```
Utworzenie obiektu API deployment:
```
kubectl apply -f redis-slave-deployment.yaml
```
Sprawdź nowo utworzone pody:
```
kubectl get pod
```
Powinieneś zobaczyć:
```
NAME                            READY     STATUS              RESTARTS   AGE
redis-master-1068406935-3lswp   1/1       Running             0          1m
redis-slave-2005841000-fpvqc    0/1       ContainerCreating   0          6s
redis-slave-2005841000-phfv9    0/1       ContainerCreating   0          6s
```

### Tworzenie servisu dla redis slave.

Aplikacja Guestbook musi komunikować się z podami Redis Slave w celu odczytania danych. 
Musimy utworzyć plik manifestu wykonując komendę:
```
vi  redis-slave-service.yaml
```
Zawartość pliku będzie wyglądała następująco:
```
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: redis
    role: slave
    tier: backend
spec:
  ports:
  - port: 6379
  selector:
    app: redis
    role: slave
    tier: backend
```
Polecenie tworzące  obiekt API typy service:
```
kubectl apply -f redis-slave-service.yaml
```
Sprawdźmy, czy usługa działa poprawnie:
```
kubectl get svc
```
Przykładowy output:
```
NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
redis-master   ClusterIP   10.0.0.151   <none>        6379/TCP   1m
redis-slave    ClusterIP   10.0.0.223   <none>        6379/TCP   6s
```
Następnie instalujemy aplikację Guestbook.
Tworzenie pliku konfiguracyjnego:
```
vi frontend-deployment.yaml
```
Zawartość pliku będzie wyglądała następująco:
```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: frontend
  labels:
    app: guestbook
spec:
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
  replicas: 3
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google-samples/gb-frontend:v4
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # Using `GET_HOSTS_FROM=dns` requires your cluster to
          # provide a dns service. As of Kubernetes 1.3, DNS is a built-in
          # service launched automatically. However, if the cluster you are using
          # does not have a built-in DNS service, you can instead
          # access an environment variable to find the master
          # service's host. To do so, comment out the 'value: dns' line above, and
          # uncomment the line below:
          # value: env
        ports:
        - containerPort: 80
```
Zastosuj deployment:
```
kubectl apply -f frontend-deployment.yaml
```
Sprawdź listę podów w celu weryfikacji czy faktycznie utworzyły się 3 repliki, zgodnie z tym co podaliśmy w pliku deploymentu.
```
kubectl get pods -l app=guestbook -l tier=frontend
```
Wynik powinien wyglądać podobnie do tego:
```
NAME                        READY     STATUS    RESTARTS   AGE
frontend-3823415956-dsvc5   1/1       Running   0          54s
frontend-3823415956-k22zn   1/1       Running   0          54s
frontend-3823415956-w9gbt   1/1       Running   0          54s
```

Teraz musimy utworzyć service dla frontend.

Serwisy redis-slave i redis-master są dostępne tylko w klastrze kontenera, ponieważ domyślnym typem usługi jest ClusterIP.

ClusterIP zapewnia pojedynczy adres IP dla zestawu slave-wów, na które wskazuje Usługa. Ten adres IP jest dostępny tylko w klastrze.
Aby umożliwić gościom dostęp do księgi gości, należy skonfigurować usługę Frontend Service tak, aby była widoczna zewnętrznie, aby klient mógł zażądać usługi poza klastrem kontenerów.

Tworzymy plik konfiguracyjny wykonując komendę:
```
vi frontend-service.yaml
```
Zawartość pliku będzie wyglądała następująco:

```
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # comment or delete the following line if you want to use a LoadBalancer
  #type: NodePort 
  # if your cluster supports it, uncomment the following to automatically create
  # an external load-balanced IP for the frontend service.
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: guestbook
    tier: frontend
```
Wykonaj polecenie:
```
kubectl apply -f frontend-service.yaml
```
Sprawdź czy obiekty zostały utworzone:
```
  kubectl get services 
```
Output powinien wyglądać podobnie do tego:

```
 NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
frontend       ClusterIP   10.0.0.112   <pending>       80:31323/TCP   6s
redis-master   ClusterIP   10.0.0.151   <none>        6379/TCP       2m
redis-slave    ClusterIP   10.0.0.223   <none>        6379/TCP       1m
```

Po utworzeniu usługi w statusie External-IP pojawi nam się zewnętrzny adres IP, dzięki któremu będziemy mogli zobaczyć jak wygląda nasza Księga Gości.
