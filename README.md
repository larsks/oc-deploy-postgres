# Deploy postgres and another service into kubernetes

## Synopsis

### To deploy

1. Edit `kustomization.yaml` to set the `namespace` appropriately.

1. Create a secret named `reporting-sec` that sets `POSTGRES_DB`,
   `POSTGRES_USER`, and `POSTGRES_PASSWORD`:

   ```
   apiVersion: v1
   kind: Secret
   metadata:
       name: reporting-sec
   type: Opaque
   stringData:
       POSTGRES_USER: reporting
       POSTGRES_PASSWORD: <password of your choice>
       POSTGRES_DB: reporting
   ```

1. Run:

    ```
    oc apply -k .
    ```

### To clean up

Run:

  ```
  oc delete -k .
  ```

### Secrets management

Alternately, see the [feature/kustomize][] branch, which shows a more
common mechanism for managing secrets as part of the repository (but
requires installing some additional packages on your local machine).

[feature/kustomize]: https://github.com/larsks/oc-deploy-postgres/tree/feature/kustomize

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

## Accessing postgres from another container

In addition to the Postgres pod itself, this repository deploys a
[phpPgAdmin][] pod as an example application that connects to
Postgres.

The phpPgAdmin service is able to connect to the Postgres service
using the service name we defined `postgres-service.yaml`:

```
apiVersion: v1
kind: Service
metadata:
  name: reporting-db-svc
spec:
  ports:
    - name: postgresql
      port: 5432
      protocol: TCP
      targetPort: 5432
  selector:
    app: reporting
    component: postgres
  type: ClusterIP
```

Deploying this service makes the hostname `reporting-db-svc` available
to pods in our OpenShift project. We configure phpPgAdmin to use our
Postgres pod by setting the `DATABASE_HOST` environment variable, as
described [in the documentation for the image we're
using][bitnami/phppgadmin]:

```
          env:
            - name: DATABASE_HOST
              value: reporting-db-svc
```

[phpPgAdmin]: https://github.com/phppgadmin/phppgadmin
[bitnami/phppgadmin]: https://hub.docker.com/r/bitnami/phppgadmin/
