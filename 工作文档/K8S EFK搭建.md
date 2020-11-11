### K8S EFK搭建

#### EFK架构图

![img](https://img2018.cnblogs.com/blog/1271786/201907/1271786-20190703161317508-973195640.png)

###### 安装

安装elasticsearch 

```shell
#创建名称空间
kubectl create ns dev
#创建elasticsearch pod
kubectl apply -f elasticsearch-master.yaml
#elasticsearch svc
kubectl apply -f elasticsearch_svc.yaml

```

elasticsearch-master.yaml内容

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  annotations:
    esMajorVersion: "8"
    meta.helm.sh/release-name: elasticsearch-1604406461
    meta.helm.sh/release-namespace: dev
  labels:
    app: elasticsearch-master
    app.kubernetes.io/managed-by: Helm
    chart: elasticsearch
    heritage: Helm
    release: elasticsearch-1604406461
  name: elasticsearch-master
  namespace: dev
spec:
  podManagementPolicy: Parallel
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: elasticsearch-master
  serviceName: elasticsearch-master-headless
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: elasticsearch-master
        chart: elasticsearch
        heritage: Helm
        release: elasticsearch-1604406461
      name: elasticsearch-master
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - elasticsearch-master
                - mongo-thirdhub-qa
            topologyKey: kubernetes.io/hostname
      containers:
      - env:
        - name: node.name
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: cluster.initial_master_nodes
          value: elasticsearch-master-0,elasticsearch-master-1,elasticsearch-master-2,
        - name: discovery.seed_hosts
          value: elasticsearch-master-headless
        - name: cluster.name
          value: elasticsearch
        - name: network.host
          value: 0.0.0.0
        - name: ES_JAVA_OPTS
          value: -Xmx1g -Xms1g
        - name: node.data
          value: "true"
        - name: node.ingest
          value: "true"
        - name: node.master
          value: "true"
        - name: node.remote_cluster_client
          value: "true"
        image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0-SNAPSHOT
        imagePullPolicy: IfNotPresent
        name: elasticsearch
        ports:
        - containerPort: 9200
          name: http
          protocol: TCP
        - containerPort: 9300
          name: transport
          protocol: TCP
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - |
              #!/usr/bin/env bash -e
              # If the node is starting up wait for the cluster to be ready (request params: "wait_for_status=green&timeout=1s" )
              # Once it has started only check that the node itself is responding
              START_FILE=/tmp/.es_start_file

              # Disable nss cache to avoid filling dentry cache when calling curl
              # This is required with Elasticsearch Docker using nss < 3.52
              export NSS_SDB_USE_CACHE=no

              http () {
                local path="${1}"
                local args="${2}"
                set -- -XGET -s

                if [ "$args" != "" ]; then
                  set -- "$@" $args
                fi

                if [ -n "${ELASTIC_USERNAME}" ] && [ -n "${ELASTIC_PASSWORD}" ]; then
                  set -- "$@" -u "${ELASTIC_USERNAME}:${ELASTIC_PASSWORD}"
                fi

                curl --output /dev/null -k "$@" "http://127.0.0.1:9200${path}"
              }

              if [ -f "${START_FILE}" ]; then
                echo 'Elasticsearch is already running, lets check the node is healthy'
                HTTP_CODE=$(http "/" "-w %{http_code}")
                RC=$?
                if [[ ${RC} -ne 0 ]]; then
                  echo "curl --output /dev/null -k -XGET -s -w '%{http_code}' \${BASIC_AUTH} http://127.0.0.1:9200/ failed with RC ${RC}"
                  exit ${RC}
                fi
                # ready if HTTP code 200, 503 is tolerable if ES version is 6.x
                if [[ ${HTTP_CODE} == "200" ]]; then
                  exit 0
                elif [[ ${HTTP_CODE} == "503" && "8" == "6" ]]; then
                  exit 0
                else
                  echo "curl --output /dev/null -k -XGET -s -w '%{http_code}' \${BASIC_AUTH} http://127.0.0.1:9200/ failed with HTTP code ${HTTP_CODE}"
                  exit 1
                fi

              else
                echo 'Waiting for elasticsearch cluster to become ready (request params: "wait_for_status=green&timeout=1s" )'
                if http "/_cluster/health?wait_for_status=green&timeout=1s" "--fail" ; then
                  touch ${START_FILE}
                  exit 0
                else
                  echo 'Cluster is not yet ready (request params: "wait_for_status=green&timeout=1s" )'
                  exit 1
                fi
              fi
          failureThreshold: 3
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 3
          timeoutSeconds: 5
        resources:
          limits:
            cpu: "1"
            memory: 2Gi
          requests:
            cpu: "1"
            memory: 2Gi
        securityContext:
          capabilities:
            drop:
            - ALL
          procMount: Default
          runAsNonRoot: true
          runAsUser: 1000
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /usr/share/elasticsearch/data
          name: elasticsearch-master
      dnsPolicy: ClusterFirst
      enableServiceLinks: true
      initContainers:
      - command:
        - sysctl
        - -w
        - vm.max_map_count=262144
        image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0-SNAPSHOT
        imagePullPolicy: IfNotPresent
        name: configure-sysctl
        resources: {}
        securityContext:
          privileged: true
          procMount: Default
          runAsUser: 0
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
      terminationGracePeriodSeconds: 120
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:
  - metadata:
      creationTimestamp: null
      name: elasticsearch-master
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
      storageClassName: pub-nfs-sc
      volumeMode: Filesystem
    status:
      phase: Pending

```

###### 注意自动申请PVC需要提前配置好nfs-client-provisioner，参考《k8s nfs自动申请pv pvc配置》文档

elasticsearch_svc.yaml文件内容

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: elasticsearch-1604406461
    meta.helm.sh/release-namespace: dev
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
  labels:
    app: elasticsearch-master
    app.kubernetes.io/managed-by: Helm
    chart: elasticsearch
    heritage: Helm
    release: elasticsearch-1604406461
  name: elasticsearch-master-headless
  namespace: dev
spec:
  type: NodePort
  ports:
  - name: http
    port: 9200
    protocol: TCP
    targetPort: 9200
  - name: transport
    port: 9300
    protocol: TCP
    targetPort: 9300
  publishNotReadyAddresses: true
  selector:
    app: elasticsearch-master
  sessionAffinity: None
```

安装kibana

```shell
#部署kibana
kubectl apply -f kibana.yaml
#创建kibana svc
kubectl apply -f kibana_svc.yaml
```



kibana.yaml文件内容

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    meta.helm.sh/release-name: kibana
    meta.helm.sh/release-namespace: dev
  labels:
    app: kibana
    app.kubernetes.io/managed-by: Helm
    heritage: Helm
    release: kibana
  name: kibana-kibana
  namespace: dev
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: kibana
      release: kibana
  strategy:
    type: Recreate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: kibana
        release: kibana
    spec:
      containers:
      - env:
        - name: ELASTICSEARCH_HOSTS
          value: http://elasticsearch-master:9200
        - name: SERVER_HOST
          value: 0.0.0.0
        - name: NODE_OPTIONS
          value: --max-old-space-size=1800
        image: docker.elastic.co/kibana/kibana:8.0.0-SNAPSHOT
        imagePullPolicy: IfNotPresent
        name: kibana
        ports:
        - containerPort: 5601
          protocol: TCP
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - |
              #!/usr/bin/env bash -e

              # Disable nss cache to avoid filling dentry cache when calling curl
              # This is required with Kibana Docker using nss < 3.52
              export NSS_SDB_USE_CACHE=no

              http () {
                  local path="${1}"
                  set -- -XGET -s --fail -L

                  if [ -n "${ELASTICSEARCH_USERNAME}" ] && [ -n "${ELASTICSEARCH_PASSWORD}" ]; then
                    set -- "$@" -u "${ELASTICSEARCH_USERNAME}:${ELASTICSEARCH_PASSWORD}"
                  fi

                  STATUS=$(curl --output /dev/null --write-out "%{http_code}" -k "$@" "http://localhost:5601${path}")
                  if [[ "${STATUS}" -eq 200 ]]; then
                    exit 0
                  fi

                  echo "Error: Got HTTP code ${STATUS} but expected a 200"
                  exit 1
              }

              http "/app/kibana"
          failureThreshold: 3
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 3
          timeoutSeconds: 5
        resources:
          limits:
            cpu: "1"
            memory: 2Gi
          requests:
            cpu: "1"
            memory: 2Gi
        securityContext:
          capabilities:
            drop:
            - ALL
          procMount: Default
          runAsNonRoot: true
          runAsUser: 1000
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 1000
      terminationGracePeriodSeconds: 30


```

kibana_svc.yaml文件内容

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: kibana
    meta.helm.sh/release-namespace: dev
  labels:
    app: kibana
    app.kubernetes.io/managed-by: Helm
    heritage: Helm
    release: kibana
  name: kibana-kibana
  namespace: dev
spec:
  ports:
  - name: http
    port: 5601
    protocol: TCP
    targetPort: 5601
  selector:
    app: kibana
    release: kibana
  sessionAffinity: None
  type: NodePort
```

安装filebeat

```shell
#部署filebeat
kubectl apply -f filebeat.yaml
```

filebeat.yaml文件内容

```yaml
kind: List
apiVersion: v1
items:
- apiVersion: v1
  kind: ConfigMap
  metadata:
    namespace: dev
    name: filebeat-config
    labels:
      k8s-app: filebeat
      kubernetes.io/cluster-service: "true"
      app: filebeat-config
  data:
    filebeat.yml: |
      filebeat.inputs:
      - type: container
      paths:
        - /var/log/containers/*.log
      processors:
        - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: "/var/log/containers/"
      output.elasticsearch:
        hosts: ["10.0.0.234:32281"]
      logging.level: info        
- apiVersion: extensions/v1beta1
  kind: DaemonSet 
  metadata:
    namespace: dev
    name: filebeat
    labels:
      k8s-app: filebeat
      kubernetes.io/cluster-service: "true"
  spec:
    template:
      metadata:
        name: filebeat
        labels:
          app: filebeat
          k8s-app: filebeat
          kubernetes.io/cluster-service: "true"
      spec:
        containers:
        - image: docker.elastic.co/beats/filebeat:8.0.0-SNAPSHOT
          name: filebeat
          args: [
            "-c", "/home/filebeat-config/filebeat.yml",
            "-e",
          ]
          securityContext:
            runAsUser: 0
          volumeMounts:
          - name: filebeat-storage
            mountPath: /var/log/containers
          - name: varlogpods
            mountPath: /var/log/pods
          - name: varlibdockercontainers
            mountPath: /var/lib/docker/containers
          - name: "filebeat-volume"
            mountPath: "/home/filebeat-config"
        volumes:
          - name: filebeat-storage
            hostPath:
              path: /var/log/containers
          - name: varlogpods
            hostPath:
              path: /var/log/pods
          - name: varlibdockercontainers
            hostPath:
              path: /var/lib/docker/containers
          - name: filebeat-volume
            configMap:
              name: filebeat-config
- apiVersion: rbac.authorization.k8s.io/v1beta1
  kind: ClusterRoleBinding
  metadata:
    name: filebeat
  subjects:
  - kind: ServiceAccount
    name: filebeat
    namespace: dev
  roleRef:
    kind: ClusterRole
    name: filebeat
    apiGroup: rbac.authorization.k8s.io
- apiVersion: rbac.authorization.k8s.io/v1beta1
  kind: ClusterRole
  metadata:
    name: filebeat
    labels:
      k8s-app: filebeat
  rules:
  - apiGroups: [""] # "" indicates the core API group
    resources:
    - namespaces
    - pods
    verbs:
    - get
    - watch
    - list
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: filebeat
    namespace: dev
    labels:
      k8s-app: filebeat

```

filebeat配置参考：https://www.elastic.co/guide/en/beats/filebeat/current/configuring-howto-filebeat.html