# Задание 1

# Создадим структуру директорий для Helm-чарта:

otushw-6-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── namespace.yaml
│   ├── pvc.yaml
│   ├── service.yaml
│   ├── storageclass.yaml
│   └── NOTES.txt
└── charts/

# В файле Chart.yaml укажем метаданные чарта:

apiVersion: v2
name: otushw-6-chart
description: A Helm chart for deploying my application
version: 0.1.0
appVersion: "1.0"

# В файле values.yaml определим переменные, которые будут использоваться в шаблонах:

namespace: homework

configMap:
  name: cmhw4
  data:
    key1: value1
    key2: value2
    key3: value3

deployment:
  name: controllers-pod
  replicas: 3
  image:
    repository: nginx
    tag: stable
  initContainer:
    image: busybox
    tag: stable
  service:
    port: 8000
    targetPort: 80

ingress:
  name: webapp-ingress
  host: homework.otus
  paths:
    - path: /homepage
      pathType: Exact
    - path: /
      pathType: Prefix

persistentVolumeClaim:
  name: pvchw4
  storageClassName: otushw4
  storage: 1Gi

storageClass:
  name: otushw4
  provisioner: k8s.io/minikube-hostpath
  reclaimPolicy: Retain
  volumeBindingMode: Immediate

# Создадим файлы 

# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.configMap.name }}
  namespace: {{ .Values.namespace }}
data:
{{- toYaml .Values.configMap.data | nindent 2 }}

# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.deployment.name }}
  namespace: {{ .Values.namespace }}
  labels:
    app: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  replicas: {{ .Values.deployment.replicas }}
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: webapp
    spec:
      nodeSelector:
        homework: "true"
      initContainers:
      - name: init-container
        image: {{ .Values.deployment.initContainer.image }}:{{ .Values.deployment.initContainer.tag }}
        command:
        - wget
        - "-O"
        - "/init/index.html"
        - http://example.com
        volumeMounts:
        - name: intro-workdir
          mountPath: "/init"
      dnsPolicy: Default
      containers:
      - name: nginx-container 
        image: {{ .Values.deployment.image.repository }}:{{ .Values.deployment.image.tag }}
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: intro-workdir
          mountPath: /usr/share/nginx/html
        - name: conf-volume
          mountPath: /usr/share/nginx/html/conf
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh","-c","rm /homework/index.html"]
        readinessProbe:
          httpGet:
            path: /index.html
            port: 80
      volumes:
      - name: intro-workdir
        persistentVolumeClaim:
          claimName: {{ .Values.persistentVolumeClaim.name }}
      - name: conf-volume
        configMap:
          name: {{ .Values.configMap.name }}

# templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.ingress.name }}
  namespace: {{ .Values.namespace }}
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      {{- range .Values.ingress.paths }}
      - path: {{ .path }}
        pathType: {{ .pathType }}
        backend:
          service:
            name: {{ $.Values.deployment.name }}-service
            port:
              number: {{ $.Values.deployment.service.port }}
      {{- end }}

# templates/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace }}

# templates/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ .Values.persistentVolumeClaim.name }}
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.persistentVolumeClaim.name }}
spec:
  storageClassName: {{ .Values.persistentVolumeClaim.storageClassName }}
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.persistentVolumeClaim.storage }}

# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.deployment.name }}-service
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: {{ .Values.deployment.service.port }}
      targetPort: {{ .Values.deployment.service.targetPort }}
  type: ClusterIP

# templates/storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: {{ .Values.storageClass.name }}
provisioner: {{ .Values.storageClass.provisioner }}
reclaimPolicy: {{ .Values.storageClass.reclaimPolicy }}
volumeBindingMode: {{ .Values.storageClass.volumeBindingMode }}

# templates/NOTES.txt
Your application has been deployed!

You can access your application at:
http://{{ .Values.ingress.host }}

# Добавим зависимость добавить зависимость в файл requirements.yaml
dependencies:
  - name: mysql
    version: "12.2.2"
    repository: "https://charts.bitnami.com/bitnami"

# Выполним команду для обновления зависимостей:

filippov@MacBook-Pro-Nikolay kubernetes-templating % helm dependency update otushw-7-chart                   
load.go:120: Warning: Dependencies are handled in Chart.yaml since apiVersion "v2". We recommend migrating dependencies to Chart.yaml.
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Downloading mysql from repo https://charts.bitnami.com/bitnami
Pulled: registry-1.docker.io/bitnamicharts/mysql:12.2.2
Digest: sha256:0ea1c03508d478ac57c9a957e7a692e0d78519bf695affb97f3766736822268d
Deleting outdated charts

# Установим чарт с помощью Helm

nfilippov@MacBook-Pro-Nikolay kubernetes-templating % helm install otushw6 ./otushw-6-chart
load.go:120: Warning: Dependencies are handled in Chart.yaml since apiVersion "v2". We recommend migrating dependencies to Chart.yaml.
NAME: otushw6
LAST DEPLOYED: Wed Feb  5 16:34:39 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Your application has been deployed!

You can access your application at:
http://homework.otus

# Проверим что поды запустились без ошибок

nfilippov@MacBook-Pro-Nikolay kubernetes-templating % kubectl get po -n homework                                 
NAME                               READY   STATUS    RESTARTS   AGE
controllers-pod-7f48d8cf89-778p2   1/1     Running   0          23m
controllers-pod-7f48d8cf89-fw5jl   1/1     Running   0          23m
controllers-pod-7f48d8cf89-hxt66   1/1     Running   0          23m

# Задание 2

# Найдем необходимую версию Kafka

nfilippov@MacBook-Pro-Nikolay kubernetes-templating % helm search repo bitnami/kafka --versions    
NAME            CHART VERSION   APP VERSION     DESCRIPTION                                       
bitnami/kafka   31.3.1          3.9.0           Apache Kafka is a distributed streaming platfor...
bitnami/kafka   31.3.0          3.9.0           Apache Kafka is a distributed streaming platfor...
bitnami/kafka   31.2.0          3.9.0           Apache Kafka is a distributed streaming platfor...
.....
bitnami/kafka   26.2.0          3.6.0           Apache Kafka is a distributed streaming platfor...
bitnami/kafka   26.1.0          3.6.0           Apache Kafka is a distributed streaming platfor...
bitnami/kafka   26.0.1          3.6.0           Apache Kafka is a distributed streaming platfor...
bitnami/kafka   26.0.0          3.6.0           Apache Kafka is a distributed streaming platfor...
bitnami/kafka   25.3.5          3.5.1           Apache Kafka is a distributed streaming platfor...

# Создадим файл helmfile.yaml

repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami

releases:
  - name: kafka-prod
    namespace: prod
    chart: bitnami/kafka
    version: 25.3.5
    values:
      - brokers: 5
      - image:
          tag: 3.5.2-debian-11-r0
      - auth:
          clientProtocol: SASL_PLAINTEXT
          interBrokerProtocol: SASL_PLAINTEXT
    
  - name: kafka-dev
    namespace: dev
    chart: bitnami/kafka
    values:
      - brokers: 1
      - auth:
          clientProtocol: PLAINTEXT
          interBrokerProtocol: PLAINTEXT
          enabled: falsehelmfile apply

# Развернем Kafka с этим helmfile.yaml

nfilippov@MacBook-Pro-Nikolay kubernetes-templating % helmfile apply
...
dev, kafka-dev-scripts, ConfigMap (v1) has been added:
+ # Source: kafka/templates/scripts-configmap.yaml
+ apiVersion: v1
+ kind: ConfigMap
+ metadata:
+   name: kafka-dev-scripts
+   namespace: "dev"
+   labels:
+     app.kubernetes.io/instance: kafka-dev
+     app.kubernetes.io/managed-by: Helm
+     app.kubernetes.io/name: kafka
+     app.kubernetes.io/version: 3.9.0
+     helm.sh/chart: kafka-31.3.1
+ data:
+   kafka-init.sh: |-
+     #!/bin/bash
+ 
+     set -o errexit
+     set -o nounset
+     set -o pipefail
+ 
+     error(){
+       local message="${1:?missing message}"
+       echo "ERROR: ${message}"
+       exit 1
+     }
+ 
+     retry_while() {
+         local -r cmd="${1:?cmd is missing}"
+         local -r retries="${2:-12}"
+         local -r sleep_time="${3:-5}"
+         local return_value=1
+ 
+         read -r -a command <<< "$cmd"
+         for ((i = 1 ; i <= retries ; i+=1 )); do
+             "${command[@]}" && return_value=0 && break
+             sleep "$sleep_time"
+         done
+         return $return_value
+     }
+ 
+     replace_in_file() {
+         local filename="${1:?filename is required}"
+         local match_regex="${2:?match regex is required}"
+         local substitute_regex="${3:?substitute regex is required}"
+         local posix_regex=${4:-true}
+ 
+         local result
+ 
+         # We should avoid using 'sed in-place' substitutions
+         # 1) They are not compatible with files mounted from ConfigMap(s)
+         # 2) We found incompatibility issues with Debian10 and "in-place" substitutions
+         local -r del=$'\001' # Use a non-printable character as a 'sed' delimiter to avoid issues
+         if [[ $posix_regex = true ]]; then
+             result="$(sed -E "s${del}${match_regex}${del}${substitute_regex}${del}g" "$filename")"
+         else
+             result="$(sed "s${del}${match_regex}${del}${substitute_regex}${del}g" "$filename")"
+         fi
+         echo "$result" > "$filename"
+     }
+ 
+     kafka_conf_set() {
+         local file="${1:?missing file}"
+         local key="${2:?missing key}"
+         local value="${3:?missing value}"
+ 
+         # Check if the value was set before
+         if grep -q "^[#\\s]*$key\s*=.*" "$file"; then
+             # Update the existing key
+             replace_in_file "$file" "^[#\\s]*${key}\s*=.*" "${key}=${value}" false
+         else
+             # Add a new key
+             printf '\n%s=%s' "$key" "$value" >>"$file"
+         fi
+     }
+ 
+     replace_placeholder() {
+       local placeholder="${1:?missing placeholder value}"
+       local password="${2:?missing password value}"
+       local -r del=$'\001' # Use a non-printable character as a 'sed' delimiter to avoid issues with delimiter symbols in sed string
+       sed -i "s${del}$placeholder${del}$password${del}g" "$KAFKA_CONFIG_FILE"
+     }
+ 
+     append_file_to_kafka_conf() {
+         local file="${1:?missing source file}"
+         local conf="${2:?missing kafka conf file}"
+ 
+         cat "$1" >> "$2"
+     }
+ 
+     configure_external_access() {
+       # Configure external hostname
+       if [[ -f "/shared/external-host.txt" ]]; then
+         host=$(cat "/shared/external-host.txt")
+       elif [[ -n "${EXTERNAL_ACCESS_HOST:-}" ]]; then
+         host="$EXTERNAL_ACCESS_HOST"
+       elif [[ -n "${EXTERNAL_ACCESS_HOSTS_LIST:-}" ]]; then
+         read -r -a hosts <<<"$(tr ',' ' ' <<<"${EXTERNAL_ACCESS_HOSTS_LIST}")"
+         host="${hosts[$POD_ID]}"
+       elif [[ "$EXTERNAL_ACCESS_HOST_USE_PUBLIC_IP" =~ ^(yes|true)$ ]]; then
+         host=$(curl -s https://ipinfo.io/ip)
+       else
+         error "External access hostname not provided"
+       fi
+ 
+       # Configure external port
+       if [[ -f "/shared/external-port.txt" ]]; then
+         port=$(cat "/shared/external-port.txt")
+       elif [[ -n "${EXTERNAL_ACCESS_PORT:-}" ]]; then
+         if [[ "${EXTERNAL_ACCESS_PORT_AUTOINCREMENT:-}" =~ ^(yes|true)$ ]]; then
+           port="$((EXTERNAL_ACCESS_PORT + POD_ID))"
+         else
+           port="$EXTERNAL_ACCESS_PORT"
+         fi
+       elif [[ -n "${EXTERNAL_ACCESS_PORTS_LIST:-}" ]]; then
+         read -r -a ports <<<"$(tr ',' ' ' <<<"${EXTERNAL_ACCESS_PORTS_LIST}")"
+         port="${ports[$POD_ID]}"
+       else
+         error "External access port not provided"
+       fi
+       # Configure Kafka advertised listeners
+       sed -i -E "s|^(advertised\.listeners=\S+)$|\1,EXTERNAL://${host}:${port}|" "$KAFKA_CONFIG_FILE"
+     }
+     configure_kafka_sasl() {
+ 
+       # Replace placeholders with passwords
+       replace_placeholder "interbroker-password-placeholder" "$KAFKA_INTER_BROKER_PASSWORD"
+       replace_placeholder "controller-password-placeholder" "$KAFKA_CONTROLLER_PASSWORD"
+       read -r -a passwords <<<"$(tr ',;' ' ' <<<"${KAFKA_CLIENT_PASSWORDS:-}")"
+       for ((i = 0; i < ${#passwords[@]}; i++)); do
+           replace_placeholder "password-placeholder-${i}\"" "${passwords[i]}\""
+       done
+     }
+ 
+     export KAFKA_CONFIG_FILE=/config/server.properties
+     cp /configmaps/server.properties $KAFKA_CONFIG_FILE
+ 
+     # Get pod ID and role, last and second last fields in the pod name respectively
+     POD_ID=$(echo "$MY_POD_NAME" | rev | cut -d'-' -f 1 | rev)
+     POD_ROLE=$(echo "$MY_POD_NAME" | rev | cut -d'-' -f 2 | rev)

+     # Configure node.id and/or broker.id
+     if [[ -f "/bitnami/kafka/data/meta.properties" ]]; then
+         if grep -q "broker.id" /bitnami/kafka/data/meta.properties; then
+           ID="$(grep "broker.id" /bitnami/kafka/data/meta.properties | awk -F '=' '{print $2}')"
+           kafka_conf_set "$KAFKA_CONFIG_FILE" "node.id" "$ID"
+         else
+           ID="$(grep "node.id" /bitnami/kafka/data/meta.properties | awk -F '=' '{print $2}')"
+           kafka_conf_set "$KAFKA_CONFIG_FILE" "node.id" "$ID"
+         fi
+     else
+         ID=$((POD_ID + KAFKA_MIN_ID))
+         kafka_conf_set "$KAFKA_CONFIG_FILE" "node.id" "$ID"
+     fi
+     replace_placeholder "advertised-address-placeholder" "${MY_POD_NAME}.kafka-dev-${POD_ROLE}-headless.dev.svc.cluster.local"
+     if [[ "${EXTERNAL_ACCESS_ENABLED:-false}" =~ ^(yes|true)$ ]]; then
+       configure_external_access
+     fi
+     configure_kafka_sasl
+     if [ -f /secret-config/server-secret.properties ]; then
+       append_file_to_kafka_conf /secret-config/server-secret.properties $KAFKA_CONFIG_FILE
+     fi
dev, kafka-dev-user-passwords, Secret (v1) has been added:
+ # Source: kafka/templates/secrets.yaml
+ apiVersion: v1
+ kind: Secret
+ metadata:
+   labels:
+     app.kubernetes.io/instance: kafka-dev
+     app.kubernetes.io/managed-by: Helm
+     app.kubernetes.io/name: kafka
+     app.kubernetes.io/version: 3.9.0
+     helm.sh/chart: kafka-31.3.1
+   name: kafka-dev-user-passwords
+   namespace: dev
+ data:
+   client-passwords: '++++++++ # (10 bytes)'
+   controller-password: '++++++++ # (10 bytes)'
+   inter-broker-password: '++++++++ # (10 bytes)'
+   system-user-password: '++++++++ # (10 bytes)'
+ type: Opaque


Upgrading release=kafka-prod, chart=bitnami/kafka, namespace=prod
Upgrading release=kafka-dev, chart=bitnami/kafka, namespace=dev
Release "kafka-prod" does not exist. Installing it now.
NAME: kafka-prod
LAST DEPLOYED: Wed Feb  5 17:26:46 2025
NAMESPACE: prod
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: kafka
CHART VERSION: 25.3.5
APP VERSION: 3.5.1

** Please be patient while the chart is being deployed **

Kafka can be accessed by consumers via port 9092 on the following DNS name from within your cluster:

    kafka-prod.prod.svc.cluster.local

Each Kafka broker can be accessed by producers via port 9092 on the following DNS name(s) from within your cluster:

    kafka-prod-controller-0.kafka-prod-controller-headless.prod.svc.cluster.local:9092
    kafka-prod-controller-1.kafka-prod-controller-headless.prod.svc.cluster.local:9092
    kafka-prod-controller-2.kafka-prod-controller-headless.prod.svc.cluster.local:9092

The CLIENT listener for Kafka client connections from within your cluster have been configured with the following security settings:
    - SASL authentication

To connect a client to your Kafka, you need to create the 'client.properties' configuration files with the content below:

security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="user1" \
    password="$(kubectl get secret kafka-prod-user-passwords --namespace prod -o jsonpath='{.data.client-passwords}' | base64 -d | cut -d , -f 1)";

To create a pod that you can use as a Kafka client run the following commands:

    kubectl run kafka-prod-client --restart='Never' --image docker.io/bitnami/kafka:3.5.2-debian-11-r0 --namespace prod --command -- sleep infinity
    kubectl cp --namespace prod /path/to/client.properties kafka-prod-client:/tmp/client.properties
    kubectl exec --tty -i kafka-prod-client --namespace prod -- bash

    PRODUCER:
        kafka-console-producer.sh \
            --producer.config /tmp/client.properties \
            --broker-list kafka-prod-controller-0.kafka-prod-controller-headless.prod.svc.cluster.local:9092,kafka-prod-controller-1.kafka-prod-controller-headless.prod.svc.cluster.local:9092,kafka-prod-controller-2.kafka-prod-controller-headless.prod.svc.cluster.local:9092 \
            --topic test

    CONSUMER:
        kafka-console-consumer.sh \
            --consumer.config /tmp/client.properties \
            --bootstrap-server kafka-prod.prod.svc.cluster.local:9092 \
            --topic test \
            --from-beginning

Listing releases matching ^kafka-prod$
kafka-prod      prod            1               2025-02-05 17:26:46.775892 +0300 MSK    deployed        kafka-25.3.5    3.5.1      

Release "kafka-dev" does not exist. Installing it now.
NAME: kafka-dev
LAST DEPLOYED: Wed Feb  5 17:26:48 2025
NAMESPACE: dev
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: kafka
CHART VERSION: 31.3.1
APP VERSION: 3.9.0

Did you know there are enterprise versions of the Bitnami catalog? For enhanced secure software supply chain features, unlimited pulls from Docker, LTS support, or application customization, see Bitnami Premium or Tanzu Application Catalog. See https://www.arrow.com/globalecs/na/vendors/bitnami for more information.

** Please be patient while the chart is being deployed **

Kafka can be accessed by consumers via port 9092 on the following DNS name from within your cluster:

    kafka-dev.dev.svc.cluster.local

Each Kafka broker can be accessed by producers via port 9092 on the following DNS name(s) from within your cluster:

    kafka-dev-controller-0.kafka-dev-controller-headless.dev.svc.cluster.local:9092
    kafka-dev-controller-1.kafka-dev-controller-headless.dev.svc.cluster.local:9092
    kafka-dev-controller-2.kafka-dev-controller-headless.dev.svc.cluster.local:9092

The CLIENT listener for Kafka client connections from within your cluster have been configured with the following security settings:
    - SASL authentication

To connect a client to your Kafka, you need to create the 'client.properties' configuration files with the content below:

security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="user1" \
    password="$(kubectl get secret kafka-dev-user-passwords --namespace dev -o jsonpath='{.data.client-passwords}' | base64 -d | cut -d , -f 1)";

To create a pod that you can use as a Kafka client run the following commands:

    kubectl run kafka-dev-client --restart='Never' --image docker.io/bitnami/kafka:3.9.0-debian-12-r6 --namespace dev --command -- sleep infinity
    kubectl cp --namespace dev /path/to/client.properties kafka-dev-client:/tmp/client.properties
    kubectl exec --tty -i kafka-dev-client --namespace dev -- bash

    PRODUCER:
        kafka-console-producer.sh \
            --producer.config /tmp/client.properties \
            --bootstrap-server kafka-dev.dev.svc.cluster.local:9092 \
            --topic test

    CONSUMER:
        kafka-console-consumer.sh \
            --consumer.config /tmp/client.properties \
            --bootstrap-server kafka-dev.dev.svc.cluster.local:9092 \
            --topic test \
            --from-beginning

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - controller.resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

Listing releases matching ^kafka-dev$
kafka-dev       dev             1               2025-02-05 17:26:48.572685 +0300 MSK    deployed        kafka-31.3.1    3.9.0      


UPDATED RELEASES:
NAME         NAMESPACE   CHART           VERSION   DURATION
kafka-prod   prod        bitnami/kafka   25.3.5          2s
kafka-dev    dev         bitnami/kafka   31.3.1          4s