[![Release](https://img.shields.io/github/v/release/piraeusdatastore/piraeus-operator)](https://github.com/piraeusdatastore/piraeus-operator/releases)
![](https://github.com/piraeusdatastore/piraeus-operator/workflows/check%20and%20build%20piraeus-operator/badge.svg)

# Piraeus Operator

The Piraeus Operator manages
[LINSTOR](https://github.com/LINBIT/linstor-server) clusters in Kubernetes.

All components of the LINSTOR software stack can be managed by the operator and
associated Helm chart:
* DRBD
* etcd cluster for LINSTOR
* LINSTOR
* LINSTOR CSI driver
* CSI snapshotter
* Stork scheduler with LINSTOR integration

## Deployment with Helm v3 Chart

The operator can be deployed with Helm v3 chart in /charts as follows:

* Prepare the hosts for DRBD deployment. There are several options:
  * Install DRBD directly on the hosts as
    [documented](https://www.linbit.com/drbd-user-guide/users-guide-9-0/#p-build-install-configure).
  * Install the appropriate kernel headers package for your distribution and
    choose the appropriate
    [kernel module injector](doc/helm-values.adoc#operatorsatellitesetkernelmoduleinjectionimage).
    Then the operator will compile and load the required modules.

- If you are deploying with images from private repositories, create
  a kubernetes secret to allow obtaining the images. This example will create
  a secret named `drbdiocred`:

    ```
    kubectl create secret docker-registry drbdiocred --docker-server=<SERVER> --docker-username=<YOUR_LOGIN> --docker-email=<YOUR_EMAIL> --docker-password=<YOUR_PASSWORD>
    ```

    The name of this secret must match the one specified in the Helm values, by
    passing `--set drbdRepoCred=drbdiocred` to helm.

* Configure storage for the LINSTOR etcd instance.
  There are various options for configuring the etcd instance for LINSTOR:
  * Use an existing storage provisioner with a default `StorageClass`.
  * [Use `hostPath` volumes](#linstor-etcd-hostpath-persistence).
  * Disable persistence for basic testing. This can be done by adding `--set
    etcd.persistentVolume.enabled=false` to the `helm install` command below.

- Configure a basic storage setup for LINSTOR:
  * Create storage pools from available devices. Recommended for simple set ups. [Guide](doc/storage.md#preparing-physical-devices)
  * Create storage pools from existing LVM setup. [Guide](doc/storage.md#configuring-storage-pool-creation)

  Read [the storage guide](doc/storage.md) and configure as needed.

- Read the [guide on securing the deployment](doc/security.md) and configure as needed.

- Read up on [optional components](doc/optional-components.md) and configure as needed.

- Finally create a Helm deployment named `piraeus-op` that will set up
  everything.

    ```
    helm install piraeus-op ./charts/piraeus
    ```

  A full list of all available options to pass to helm can be found [here](./doc/helm-values.adoc).

### LINSTOR etcd `hostPath` persistence

You can use the included Helm templates to create `hostPath` persistent volumes.
Create as many PVs as needed to satisfy your configured etcd `replicas`
(default 1).

Create the `hostPath` persistent volumes, substituting cluster node names
accordingly in the `nodes=` option. For 1 replica:

```
helm install linstor-etcd ./charts/pv-hostpath --set "nodes={<NODE>}"
```

For 3 replicas:

```
helm install linstor-etcd ./charts/pv-hostpath --set "nodes={<NODE0>,<NODE1>,<NODE2>}"
```

Persistence for etcd is enabled by default.

### Using an existing database

LINSTOR can connect to an existing PostgreSQL, MariaDB or etcd database. For
instance, for a PostgresSQL instance with the following configuration:

```
POSTGRES_DB: postgresdb
POSTGRES_USER: postgresadmin
POSTGRES_PASSWORD: admin123
```

The Helm chart can be configured to use this database instead of deploying an
etcd cluster by adding the following to the Helm install command:

```
--set etcd.enabled=false --set "operator.controller.dbConnectionURL=jdbc:postgresql://postgres/postgresdb?user=postgresadmin&password=admin123"
```

### Image mirror for CN users

The chart contains a values file prepared for chinese users [`values.cn.yaml`](./charts/piraeus/values.cn.yaml).
It replaces the default image locations with images hosted on daocloud.io.

### Pod resources

You can configure [resource requests and limits] for all deployed containers. Take a look at
this [example chart configuration.](./examples/resource-requirements.yaml)

[resource requests and limits]: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/


### Terminating Helm deployment

To protect the storage infrastructure of the cluster from accidentally deleting vital components, it is necessary
to perform some manual steps before deleting a Helm deployment.

1. Delete all volume claims managed by piraeus components.
   You can use the following command to get a list of volume claims managed by Piraeus.
   After checking that non of the listed volumes still hold needed data, you can delete them using the generated
   `kubectl delete` command.

   ```
   $ kubectl get pvc --all-namespaces -o=jsonpath='{range .items[?(@.metadata.annotations.volume\.beta\.kubernetes\.io/storage-provisioner=="linstor.csi.linbit.com")]}kubectl delete pvc --namespace {.metadata.namespace} {.metadata.name}{"\n"}{end}'
   kubectl delete pvc --namespace default data-mysql-0
   kubectl delete pvc --namespace default data-mysql-1
   kubectl delete pvc --namespace default data-mysql-2
   ```

   **WARNING** These volumes, once deleted, cannot be recovered.

2. Delete the LINSTOR controller and satellite resources.

   Deployment of LINSTOR satellite and controller is controlled by the `linstorsatelliteset` and `linstorcontroller`
   resources. You can delete the resources associated with your deployment using `kubectl`

   ```
   kubectl delete linstorsatelliteset <helm-deploy-name>-ns
   kubectl delete linstorcontroller <helm-deploy-name>-cs
   ```

   After a short wait, the controller and satellite pods should terminate. If they continue to run, you can
   check the above resources for errors (they are only removed after all associated pods terminate)

3. Delete the Helm deployment.

   If you removed all PVCs and all LINSTOR pods have terminated, you can uninstall the helm deployment

   ```
   helm uninstall piraeus-op
   ```

   However due to the Helm's current policy, the Custom Resource Definitions named
   `linstorcontroller` and `linstorsatelliteset` will __not__ be deleted by the
   command.

   More information regarding Helm's current position on CRD's can be found
   [here](https://helm.sh/docs/topics/chart_best_practices/custom_resource_definitions/#method-1-let-helm-do-it-for-you).

## Deployment without using Helm v3 chart

### Configuration

The operator must be deployed within the cluster in order for it to have access
to the controller endpoint, which is a kubernetes service.

### Kubernetes Secret for Repo Access

If you are deploying with images from a private repository, create a kubernetes
secret to allow obtaining the images.  Create a secret named `drbdiocred` like
this:

```
kubectl create secret docker-registry drbdiocred --docker-server=<SERVER> --docker-username=<YOUR LOGIN> --docker-email=<YOUR EMAIL> --docker-password=<YOUR PASSWORD>
```

### Deploy Operator

First you need to create the resource definitions

```
kubectl create -f charts/piraeus/crds/piraeus.linbit.com_linstorcsidrivers_crd.yaml
kubectl create -f charts/piraeus/crds/piraeus.linbit.com_linstorsatellitesets_crd.yaml
kubectl create -f charts/piraeus/crds/piraeus.linbit.com_linstorcontrollers_crd.yaml
```

Then, take a look at the files in [`deploy/piraeus`](./deploy/piraeus) and make changes as
you see fit. For example, you can edit the storage pool configuration by editing
[`operator-satelliteset.yaml`](deploy/piraeus/templates/operator-satelliteset.yaml)
like shown in [the storage guide](./doc/storage.md#configuring-storage-pool-creation).

Now you can finally deploy the LINSTOR cluster with:
```
kubectl create -Rf deploy/piraeus/
```

## Upgrading

Please see the dedicated [UPGRADE document](./UPGRADE.md)

## Contributing

If you'd like to contribute, please visit https://github.com/piraeusdatastore/piraeus-operator
and look through the issues to see if there is something you'd like to work on. If
you'd like to contribute something not in an existing issue, please open a new
issue beforehand.

If you'd like to report an issue, please use the issues interface in this
project's github page.

## Building and Development

This project is built using the operator-sdk (version 0.16.0). Please refer to
the [documentation for the sdk](https://github.com/operator-framework/operator-sdk/tree/v0.16.x).

## License

Apache 2.0
