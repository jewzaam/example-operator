# Create Source
```
operator-sdk new example-operator
cd example-operator
operator-sdk add api --api-version=app.example.com/v1alpha1 --kind=AppService
operator-sdk add controller --api-version=app.example.com/v1alpha1 --kind=AppService
operator-sdk olm-catalog gen-csv --csv-version=0.0.1

cp deploy/crds/*crd.yaml deploy/olm-catalog/example-operator/0.0.1

sed -i 's|REPLACE_IMAGE|quay.io/nmalik/example-operator:latest|g' deploy/operator.yaml
sed -i 's|placeholder|example-operator|g' $(find deploy/ -name '*clusterserviceversion.yaml')
```

# Build & Push Images
```
operator-sdk build quay.io/nmalik/example-operator
docker build -f build/Dockerfile.catalog_registry -t quay.io/nmalik/example-operator-registry .

docker push quay.io/nmalik/example-operator
docker push quay.io/nmalik/example-operator-registry
```

# Install: OCP 4.1
```
oc delete ns example-operator
oc apply -R -f install/openshift-4.1/
```

# Install: OCP 4.2
```
oc delete ns example-operator
oc apply -R -f install/openshift-4.2/
```

## Verify Privilege Escalation
The whole point of making this install via a `CatalogSource` and `OperatorGroup` was to check assumptions on the feature added in OCP 4.2 that allows you to tie an `OperatorGroup` to a `ServiceAccount` for operator installation.  Instead of using cluster-admin, it uses that SA.

In this test, we'll be checking that the operator that's installed *fails* because OLM, using the provided SA, cannot grant the RBAC requested.

We do this by taking the `Role` assigned to the `ServiceAccount` (which is simply exactly what the operator requests) and removing sometime.

```
oc delete ns example-operator
oc apply -R -f install/openshift-4.2-broken-rbac/
```

You should see the install fail.  You can view this in the `InstallPlan`:

```
$ oc -n example-operator get installplan -o json | jq -r '.items[].status.conditions'
[
  {
    "lastTransitionTime": "2019-10-02T20:38:01Z",
    "lastUpdateTime": "2019-10-02T20:38:01Z",
    "message": "error creating csv example-operator.v0.0.1: clusterserviceversions.operators.coreos.com is forbidden: User \"system:serviceaccount:example-operator:operatorgroup-sa\" cannot create resource \"clusterserviceversions\" in API group \"operators.coreos.com\" in the namespace \"example-operator\"",
    "reason": "InstallComponentFailed",
    "status": "False",
    "type": "Installed"
  }
]
```

If you do see it installed, we can prove it shouldn't be able to do that by:
1. delete the `RoleBinding` to `exmaple-operator` SA
1. login as the SA `operatorgroup-sa`
1. attempt to grant the role to `example-operator` SA

```
ROLE_NAME=$(oc -n example-operator get roles -l olm.owner=example-operator.v0.0.1 -o jsonpath='{.items[].metadata.name}')

ROLEBINDING_NAME=$(oc -n example-operator get rolebindings -l olm.owner=example-operator.v0.0.1 -o jsonpath='{.items[].metadata.name}')

oc get rolebinding $ROLEBINDING_NAME -o yaml > /tmp/example-operator.rolebinding.yaml
oc delete -f /tmp/example-operator.rolebinding.yaml

TOKEN=$(oc -n example-operator get secret $(oc -n example-operator get secret --no-headers | grep operatorgroup-sa-token | head -n1 | awk '{print $1}') -o jsonpath='{.data.token}' | base64 --decode)
API_SERVER=$(oc get infrastructures cluster -o jsonpath='{.status.apiServerURL}')
oc login $API_SERVER --token=$TOKEN

oc create -f /tmp/example-operator.rolebinding.yaml
```