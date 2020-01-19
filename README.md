# IBM UrbanCode Deploy deployment on RedHat Openshift

Step by step guide to deploy UCD on RedHat Openshift

## Prerequisites

1. Deploy MariaDB from openshift catalog (Please make sure it is not ephemral and has persistance)

> Note:- mariadb wont work, follow the procedure below to fix.

- Navigate to project

```
$ oc project projectname
```

- Add scc policy to your current project

```
$ oc adm policy add-scc-to-user privileged -z default
```

- Change the deployment config of MariaDB

```
$ oc edit dc deploymentconfigname

securityContext:
    privileged: true
```

> After this step, your mariadb will be working.

2. Create PV and PVC to copy mysql-jdbc.jar file in host path filesystem

```YAML
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ucd-ext-lib
  labels:
    volume: ucd-appdata-vol
spec:
  storageClassName: "ibmc-block-bronze"
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt"
```

```YAML
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ucd-appdata-vol
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      volume: ucd-appdata-vol
```

3. create a busybox pod and mount the volume

```YAML
kind: Pod
apiVersion: v1
metadata:
  name: volume
spec:
  volumes:
    - name: volume
      persistentVolumeClaim:
       claimName: ucd-ext-lib-volc
  containers:
    - name: debugger
      image: busybox
      command: ['sleep', '360000']
      volumeMounts:
        - mountPath: "/mnt"
          name: volume
```

4. Exec into the volume pod

```sh
$ kubectl exec -it volume /bin/sh
$ kubectl cp mysql-jdbc.jar <namespace>/volume:/mnt
```

5. create ucd-appdata-vol pv and pvc

```YAML
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ucd-appdata-vol
  labels:
    volume: ucd-appdata-vol
spec:
  storageClassName: "ibmc-block-bronze"
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt"
```

```YAML
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ucd-appdata-volc
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 20Gi
  selector:
    matchLabels:
      volume: ucd-appdata-volc
```

6. Create SecurityContextConstraints yaml

```YAML
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  annotations:
  name: ibm-ucd-prod-scc
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegedContainer: false
allowedCapabilities: []
allowedFlexVolumes: []
defaultAddCapabilities: []
defaultPrivilegeEscalation: false
forbiddenSysctls:
  - "*"
fsGroup:
  type: MustRunAs
  ranges:
  - max: 65535
    min: 1
readOnlyRootFilesystem: false
requiredDropCapabilities:
- ALL
runAsUser:
  type: MustRunAsNonRoot
seccompProfiles:
- docker/default
seLinuxContext:
  type: RunAsAny
supplementalGroups:
  type: MustRunAs
  ranges:
  - max: 65535
    min: 1
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
priority: 0
```

6. Set up helm

```
$ oc new-project tiller
$ oc project tiller
$ export TILLER_NAMESPACE=tiller
```

for MAC

```
$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-darwin-amd64.tar.gz | tar xz
$ cd darwin-amd64
$ ./helm init --client-only
```

for Linux

```
$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.9.0-linux-amd64.tar.gz | tar xz
$ cd linux-amd64
$ ./helm init --client-only
```

- Install the Tiller server

```
$ oc process -f https://github.com/openshift/origin/raw/master/examples/helm/tiller-template.yaml -p TILLER_NAMESPACE="${TILLER_NAMESPACE}" -p HELM_VERSION=v2.9.0 | oc create -f -
$ oc policy add-role-to-user edit "system:serviceaccount:${TILLER_NAMESPACE}:tiller"
```

## Create secret

```YAML
apiVersion: v1
kind: Secret
metadata:
  name: MyRelease-secrets
type: Opaque
data:
  initpassword: <mariadbpassword in base64>
  dbpassword: <mariadbpassword in base64>
```

## Install

1. change values.yaml accordingly

```
$ helm install ibm-ucd-prod/ --name="ucd"
```
