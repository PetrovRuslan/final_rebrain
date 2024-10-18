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

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

<!-- helm repo add mysql-operator https://mysql.github.io/mysql-operator/
helm repo update -->
