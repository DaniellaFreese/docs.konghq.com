To install {{ site.kgo_product_name }} use `kubectl apply`:

```bash
kubectl apply -f {{site.links.web}}/assets/gateway-operator/v{{page.version}}/crds.yaml --server-side
kubectl apply -f {{site.links.web}}/assets/gateway-operator/v{{page.version}}/all_controllers.yaml
```

You can wait for the operator to be ready using `kubectl wait`:

```bash
kubectl -n kong-system wait --for=condition=Available=true --timeout=120s deployment/gateway-operator-controller-manager
```
