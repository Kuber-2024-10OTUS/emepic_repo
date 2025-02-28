# Создим файл mysql-crd.yaml с манифестом для CRD:
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: mysqls.otus.homework
spec:
  group: otus.homework
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                image:
                  type: string
                database:
                  type: string
                password:
                  type: string
                storage_size:
                  type: string
              required:
                - image
                - database
                - password
                - storage_size
  scope: Namespaced
  names:
    plural: mysqls
    singular: mysql
    kind: MySQL
    shortNames:
      - mysql
# Создим файл rbac.yaml для определения ServiceAccount, ClusterRole и ClusterRoleBinding:

apiVersion: v1
kind: ServiceAccount
metadata:
  name: mysql-operator
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: mysql-operator
rules:
- apiGroups: ["", "apps", "batch", "extensions"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["otus.homework"]
  resources: ["mysqls"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: mysql-operator
subjects:
- kind: ServiceAccount
  name: mysql-operator
  namespace: default
roleRef:
  kind: ClusterRole
  name: mysql-operator
  apiGroup: rbac.authorization.k8s.io

# Создим файл operator-deployment.yaml для развертывания оператора:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-operator
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-operator
  template:
    metadata:
      labels:
        app: mysql-operator
    spec:
      serviceAccountName: mysql-operator
      containers:
      - name: mysql-operator
        image: roflmaoinmysoul/mysql-operator:1.0.0
        env:
        - name: WATCH_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: OPERATOR_NAME
          value: "mysql-operator"

# Создим файл mysql-instance.yaml с примером кастомного ресурса MySQL:
apiVersion: otus.homework/v1
kind: MySQL
metadata:
  name: mysql-instance
spec:
  image: mysql:5.7
  database: mydatabase
  password: mypassword
  storage_size: 1Gi

# Применим все созданные манифесты
kubectl apply -f mysql-crd.yaml
kubectl apply -f rbac.yaml
kubectl apply -f operator-deployment.yaml
kubectl apply -f mysql-instance.yaml

# Убедимся, что CRD создан
nfilippov@MacBook-Pro-Nikolay kubernetes-operators % kubectl get crd mysqls.otus.homework
NAME                   CREATED AT
mysqls.otus.homework   2025-02-24T14:41:56Z

# Проверим, что оператор запущен
nfilippov@MacBook-Pro-Nikolay kubernetes-operators % kubectl get pods -l app=mysql-operator
NAME                              READY   STATUS    RESTARTS   AGE
mysql-operator-5b6c945f5c-z98zw   1/1     Running   0          118s

# Проверим, что созданы ресурсы (Deployment, Service, PV, PVC):
nfilippov@MacBook-Pro-Nikolay kubernetes-operators % kubectl get deployment,service,pv,pvc
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql-instance   0/1     1            0           80s
deployment.apps/mysql-operator   1/1     1            1           3m14s

NAME                     TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/kubernetes       ClusterIP   10.96.0.1    <none>        443/TCP    6m53s
service/mysql-instance   ClusterIP   None         <none>        3306/TCP   80s

NAME                                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/mysql-instance-pv   1Gi        RWO            Retain           Bound    default/mysql-instance-pvc   standard       <unset>                          80s

NAME                                       STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/mysql-instance-pvc   Bound    mysql-instance-pv   1Gi        RWO            standard       <unset>                 80s


Задание c *

# Обновим файл rbac.yaml, чтобы предоставить только необходимые права для работы с CRD и связанными ресурсами:

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: mysql-operator
rules:
  # Права для работы с CRD (MySQL)
- apiGroups: ["otus.homework"]
  resources: ["mysqls"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Права для работы с Pods, Deployments, Services, PVC, PV
- apiGroups: [""]
  resources: ["pods", "services", "persistentvolumeclaims", "persistentvolumes"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Применим обновленный манифест rbac.yaml:

nfilippov@MacBook-Pro-Nikolay kubernetes-operators % kubectl apply -f rbac.yaml
clusterrole.rbac.authorization.k8s.io/mysql-operator configured

# Убедимся, что оператор продолжает работать:
nfilippov@MacBook-Pro-Nikolay kubernetes-operators % kubectl get pods -l app=mysql-operator
NAME                              READY   STATUS    RESTARTS   AGE
mysql-operator-5b6c945f5c-bwbsn   1/1     Running   0          2m27s

# Проверим, что оператор создает ресурсы (Deployment, Service, PVC, PV) при создании кастомного ресурса MySQL
nfilippov@MacBook-Pro-Nikolay kubernetes-operators % kubectl delete -f mysql-instance.yaml
mysql.otus.homework "mysql-instance" deleted

nfilippov@MacBook-Pro-Nikolay kubernetes-operators % kubectl apply -f mysql-instance.yaml
mysql.otus.homework/mysql-instance created

nfilippov@MacBook-Pro-Nikolay kubernetes-operators % kubectl get deployment,service,pvc,pv
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql-instance   0/1     1            0           17s
deployment.apps/mysql-operator   1/1     1            1           4m28s

NAME                     TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/kubernetes       ClusterIP   10.96.0.1    <none>        443/TCP    3d20h
service/mysql-instance   ClusterIP   None         <none>        3306/TCP   17s

NAME                                       STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/mysql-instance-pvc   Bound    mysql-instance-pv   1Gi        RWO            standard       <unset>                 17s

NAME                                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/mysql-instance-pv   1Gi        RWO            Retain           Bound    default/mysql-instance-pvc   standard       <unset>                          17s

# Убедимся, что оператор корректно удаляет ресурсы при удалении кастомного ресурса MySQL:

nfilippov@MacBook-Pro-Nikolay kubernetes-operators % kubectl delete -f mysql-instance.yaml
mysql.otus.homework "mysql-instance" deleted

nfilippov@MacBook-Pro-Nikolay kubernetes-operators % kubectl get deployment,service,pvc,pv
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql-operator   1/1     1            1           6m12s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3d20h

