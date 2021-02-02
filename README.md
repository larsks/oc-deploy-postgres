# Deploy postgres and another service into kubernetes

## To deploy

1. Edit `kustomization.yaml` to set the `namespace` appropriately.
1. Create a secret named `reporting-sec` with the following
   attributes:

   ```
   POSTGRES_DB: reporting
   POSTGRES_USER: reporting
   POSTGRES_PASSWORD: <password of your choice>
   ```

1. Run:

    ```
    oc apply -k .

## Details

This process is driven by the `kustomization.yaml` file in this
directory, which looks like this:

```
namespace: lars-sandbox
commonLabels:
  app: reporting

resources:
  - postgres-deployment.yaml
  - postgres-pvc.yaml
  - postgres-service.yaml
  - phppgadmin-deployment.yaml
  - phppgadmin-service.yaml

configMapGenerator:
  - name: reporting-db-initdb
    files:
      - init_rpt.sql
```

The first three lines...

```
namespace: lars-sandbox
commonLabels:
  app: reporting
```

...identify common attributes that will be applied to
all generates resources. In otherwords, if we a resource definition
that looks like this:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: reporting-db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  volumeMode: Filesystem
```

When we run `oc apply -k`, the generates resource will instead look
like this:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: reporting
  name: reporting-db-pvc
  namespace: lars-sandbox
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeName: pvc-ee4504a0-e430-427f-b096-a9893b60ddcc
```

The `namespace` attribute controls into which namespace our resources
will be deployed, and the common `app: reporting` label can be used to
identify resources that belong to this particular application (vs
other unrelated things deployed in the same OpenShift project).
resources:

The `resources` stanza...

```
resources:
  - postgres-deployment.yaml
  - postgres-pvc.yaml
  - postgres-service.yaml
  - phppgadmin-deployment.yaml
  - phppgadmin-service.yaml
```

...describes a set of static resource files that will be used to
create OpenShift resources.

Finally, the `configMapGenerator` stanza...

```
configMapGenerator:
  - name: reporting-db-initdb
    files:
      - init_rpt.sql
```

...is used to create a `ConfigMap` from the contents of the
`init_rpt.sql` file.  We mount this into `/docker-entrypoint-initdb.d`
inside the Postgres container:


```
[...]
            - mountPath: /docker-entrypoint-initdb.d
              name: postgres-initdb
[...]
      volumes:
        - name: postgres-initdb
          configMap:
            name: reporting-db-initdb
```
