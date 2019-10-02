# Create Source
```
operator-sdk new example-operator
cd example-operator
operator-sdk add api --api-version=app.example.com/v1alpha1 --kind=AppService
operator-sdk add controller --api-version=app.example.com/v1alpha1 --kind=AppService
operator-sdk olm-catalog gen-csv --csv-version=0.0.1

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

# Install
```
oc apply -R -f install/
```