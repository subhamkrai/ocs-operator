# The OCS Meta Operator

This is the primary operator for Red Hat OpenShift Container Storage (OCS). It
is a "meta" operator, meaning it serves to facilitate the other operators in
OCS by performing administrative tasks outside their scope as well as
watching and configuring their CustomResources (CRs).

## Build

### OCS Operator
The operator is based on the [Operator
SDK](https://github.com/operator-framework/operator-sdk). In order to build the
operator, you first need to install the SDK. [Instructions are
here.](https://github.com/operator-framework/operator-sdk#quick-start)

Once the SDK is installed, the operator can be built via:

```console
$ dep ensure --vendor-only

$ operator-sdk build quay.io/openshift/ocs-operator
```

### Converged CSV

_TODO: Add instructions to build the converged CSV and manifests_

## Install

The OCS operator can be installed into an Openshift cluster using the OLM.

For quick install using pre-built container images, deploy the [deploy-olm.yaml] manifest.

```console
$ oc create -f ./deploy/deploy-with-olm.yaml
```

This will create a custom CatalogSource, a new openshift-storage Namespace, an
OperatorGroup, and a Subcription to the OCS catalog in the openshift-storage
namespace.

Once this is done, a StorageCluster can be created.

```console
$ oc create -f ./deploy/crds/ocs_v1alpha1_storagecluster_cr.yaml
```

### Installation of development builds

To install own development builds of OCS, first build and push the ocs-operator image to your own image repository.

Once the ocs-operator image is pushed, edit the CSV to point to the new image.

```
$ OCS_OPERATOR_IMAGE="custom-ocs-operator-image:with-tag"
$ sed -i "s|quay.io/ocs-dev/ocs-operator:latest|$OCS_OPERATOR_IMAGE" ./deploy/olm-catalog/ocs-operator/0.0.1/ocs-operator.v0.0.1.clusterserviceversion.yaml
```

Then build and upload the catalog registry image.

```console
$ export REGISTRY_NAMESPACE=<quay-username>
$ export CONTAINER_TAG=<some-tag>
$ ./hack/build-registry-bundle.sh
```

Next create the namespace for OCS and create an OperatorGroup for OCS
```console
$ oc create ns openshift-storage

$ cat <<EOF | oc create -f -
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: openshift-storage-operatorgroup
  namespace: openshift-storage
EOF
```

Next add a new CatalogSource using the newly built and pushed registry image.
```console
$ cat <<EOF | oc create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ocs-catalogsource
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/$REGISTRY_NAMESPACE/ocs-registry:$CONTAINER_TAG
  displayName: Openshift Container Storage
  publisher: Red Hat
EOF
```

Finally subscribe to the OCS catalog.
```console
$ cat <<EOF | oc create -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ocs-subscription
  namespace: openshift-storage
spec:
  channel: alpha
  name: ocs-operator
  source: ocs-catalogsource
  sourceNamespace: openshift-marketplace
EOF
```

## Initial Data

When the operator starts, it will create a single OCSInitialization resource. That
will cause various initial data to be created, including default
StorageClasses.

The OCSInitialization resource is a singleton. If the operator sees one that it
did not create, it will write an error message to its status explaining that it
is being ignored.

### Modifying Initial Data

You may modify or delete any of the operator's initial data. To reset and
restore that data to its initial state, delete the OCSInitialization resource. It
will be recreated, and all associated resources will be either recreated or
restored to their original state.