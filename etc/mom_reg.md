



```bash
NAMESPACE=demo-project
MODEL_REGISTRY_NAME=fraud-model-registry
MARIADB_NAME=mariadb-fraud
```



```bash
oc create secret generic my-registry-password \
--from-literal=MYSQL_USER=admin \
--from-literal=MYSQL_PASSWORD=adminpass \
--from-literal=MYSQL_DATABASE=mydatabase \
-n rhoai-model-registries
```

```bash
cat << EOF >> maria-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:  
  name: ${MARIADB_NAME}-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeMode: Filesystem
EOF
```

```bash
oc new-app registry.redhat.io/rhel8/mariadb-105:latest \
    --name=${MARIADB_NAME} \
    -e MYSQL_USER=admin \
    -e MYSQL_PASSWORD=adminpass \
    -e MYSQL_DATABASE=sampledb \
  -n rhoai-model-registries
```

```bash
oc apply -f maria-pvc.yaml -n rhoai-model-registries
oc set volumes deployment/${MARIADB_NAME} \
  --add \
  --name=${MARIADB_NAME}-data \
  --claim-mode='ReadWriteOnce' \
  --claim-name=${MARIADB_NAME}-data \
  -m /var/lib/mysql/data \
  -n rhoai-model-registries
```



```bash
MYSQL_NAME=${MARIADB_NAME} \
MYSQL_SECRET=my-registry-password \
MYSQL_DATABASE=sampledb \
MYSQL_USER=admin \
MODEL_REGISTRY_NAME=${MODEL_REGISTRY_NAME} \
APPS_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
```

```bash
cat << EOF >>  model-registry.yaml
apiVersion: modelregistry.opendatahub.io/v1beta1
kind: ModelRegistry
metadata:
  annotations:
    openshift.io/description: ''
    openshift.io/display-name: ${MODEL_REGISTRY_NAME}
  name: ${MODEL_REGISTRY_NAME}
  namespace: rhoai-model-registries  
spec:
  grpc:
    port: 9090
  oauthProxy:
    port: 8443
    routePort: 443
    serviceRoute: enabled
  mysql:
    database: ${MYSQL_DATABASE}
    host: ${MYSQL_NAME}.rhoai-model-registries.svc.cluster.local
    passwordSecret:
      key: MYSQL_PASSWORD
      name: ${MYSQL_SECRET}
    port: 3306
    skipDBCreation: false
    username: ${MYSQL_USER}
  rest:    
    serviceRoute: enabled
EOF
```

```bash
oc create -f model-registry.yaml -n rhoai-model-registries 
```



```bash
cat << EOF >>  rb-model-registry.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dashboard-permissions
  namespace: rhoai-model-registries  
  labels:
    app: ${MODEL_REGISTRY_NAME}
    app.kubernetes.io/component: model-registry
    app.kubernetes.io/name: ${MODEL_REGISTRY_NAME}
    app.kubernetes.io/part-of: model-registry
    component: model-registry
    opendatahub.io/dashboard: 'true'
    opendatahub.io/rb-project-subject: 'true'
subjects:
  - kind: Group
    apiGroup: rbac.authorization.k8s.io
    name: "system:serviceaccounts:${NAMESPACE}"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: registry-user-${MODEL_REGISTRY_NAME}
EOF
```

```
oc apply -f rb-model-registry.yaml -n rhoai-model-registries 
```

