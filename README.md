# final_rebrain

## 0. Prepare

### 0.1 Install helm

wget https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz

tar -xvzf helm-v3.16.2-linux-amd64.tar.gz

sudo mv linux-amd64/helm /usr/local/bin/helm

### 0.2 Install nfs-subdir-external-provisioner

sudo apt install nfs-common -y <!-- на всех нодах>

<!-- $ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=x.x.x.x \
    --set nfs.path=/exported/path -->

git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git

cd nfs-subdir-external-provisioner/

sed -i'' "s/namespace:.*/namespace: nfs-provisioner/g" ./deploy/rbac.yaml ./deploy/deployment.yaml

kubectl create ns nfs-provisioner

kubectl apply -f deploy/rbac.yaml

nano deploy/deployment.yaml

``` 
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 10.129.0.12
            - name: NFS_PATH
              value: /opt/nfs/
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.129.0.12
            path: /opt/nfs/
```

kubectl apply -f deploy/deployment.yaml

kubectl apply -f deploy/class.yaml

kubectl get deploy -n=nfs-provisioner

## 1. MYSQL

<!-- Пример https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/ -->

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm pull bitnami/mysql
tar xvf mysql-11.1.19.tgz
cd mysql
nano values.yaml
<!-- в values отключить pvc enabled: false-->

kubectl create ns mysql-ns
export MYSQL_ROOT_PASSWORD=strong-password
helm install --set mysqlRootPassword=$MYSQL_ROOT_PASSWORD --set volumePermissions.enabled=false -n mysql-ns mysql-release01 ./
<!-- helm repo add mysql-operator https://mysql.github.io/mysql-operator/
helm repo update -->

## 2. Cert-manager & nginx-ingress-controller

### 2.1 
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: ov@fevlake.com
    privateKeySecretRef:
      name: letsencrypt-private-key
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
       ingress:
         class: nginx
```

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: back
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/auth-type: "basic"
    nginx.ingress.kubernetes.io/auth-secret: "basic-auth"
    nginx.ingress.kubernetes.io/auth-secret-type: "auth-file"
    # Указываем, каким образом выписывать сертификат
    cert-manager.io/cluster-issuer: "letsencrypt"
spec:
  ingressClassName: nginx
  rules:
  - host: my-sandbox.ru
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: 80
  # Указываем настройки для tls — для какого хоста нужен tls и куда сохранить полученный сертификат
  tls:
  - hosts:
    - my-sandbox.ru
    secretName: back-dev-kis-im-cert
```

### 2.2 

<!-- кажется уже установлен -->
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/cloud/deploy.yaml

kubectl -n ingress-nginx get deploy 