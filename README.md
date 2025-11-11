# Manage workflows with the Dapr extension for Azure Kubernetes Service (AKS)
With Dapr Workflows, you can easily orchestrate messaging, state management, and failure-handling logic across various microservices. Dapr Workflows can help you create long-running, fault-tolerant, and stateful applications.

## Running this sample

### Before you begin

1. Clone the sample project.
2. [Create an AKS cluster.](https://learn.microsoft.com/azure/aks/learn/quick-kubernetes-deploy-cli)
3. [Install the Dapr extension on your AKS cluster.](https://learn.microsoft.com/azure/aks/dapr#prerequisites)
4. **Ensure you have the Azure Kubernetes Service RBAC Admin role** or equivalent permissions to access your AKS cluster.

### Deploy to AKS

1. **Configure kubectl for your AKS cluster.**

   ```bash
   az aks get-credentials --resource-group <your-resource-group> --name <your-cluster-name>
   ```

   **Verify you're connected to the correct cluster:**
   ```bash
   kubectl config current-context
   kubectl config get-contexts
   kubectl get nodes
   ```

   If you see an unexpected cluster name (like `minikube`) in the current context, make sure to switch to your AKS cluster:
   ```bash
   kubectl config use-context <your-aks-cluster-context>
   ```

2. **Make sure that Dapr is installed on the cluster.** If not, run `dapr init -k`.

3. **Install Redis on the cluster.**

   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm install redis bitnami/redis
   kubectl apply -f Deploy/redis.yaml
   ```

4. **Install the sample app.**

   ```bash
   kubectl apply -f Deploy/deployment.yaml
   ```

5. **Expose the Dapr sidecar (only for testing!) and the sample app.**

   ```bash
   kubectl apply -f Deploy/service.yaml
   export SAMPLE_APP_URL=$(kubectl get svc/workflows-sample -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   export DAPR_URL=$(kubectl get svc/workflows-sample-dapr -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   ```

### Run the sample workflow

1. **Before starting the workflow, make an API call to the sample app to restock items in the inventory.**

   ```bash
   curl -X GET $SAMPLE_APP_URL/stock/restock
   ```

2. **Start the workflow.**

   ```bash
   curl -i -X POST $DAPR_URL/v1.0-alpha1/workflows/dapr/OrderProcessingWorkflow/start \
     -H "Content-Type: application/json" \
     -H "dapr-app-id: dwf-app" \
     -d '{"Name": "Paperclips", "TotalCost": 99.95, "Quantity": 1}'
   ```

   This will return a response with an auto-generated instance ID:
   ```json
   {"instanceID":"<generated-id>"}
   ```

3. **Check the status of the workflow** (optional, as completed workflows may not be queryable).

   ```bash
   curl -i -X GET $DAPR_URL/v1.0-alpha1/workflows/dapr/OrderProcessingWorkflow/<instance-id> \
     -H "dapr-app-id: dwf-app"
   ```

4. **Monitor workflow execution** by checking the application logs:

   ```bash
   kubectl logs -l run=workflows-sample -c workflows-sample --tail=20
   ```

   You should see logs showing the workflow progression:
   ```
   Processing payment: <instance-id> for 1 Paperclips at $99.95
   Payment for request ID '<instance-id>' processed successfully
   There are now 99 Paperclips left in stock
   Order <instance-id> has completed!
   ```

## Related links
- [Dapr Workflow documentation](https://docs.dapr.io/developing-applications/building-blocks/workflow/)
- [Dapr extension for AKS and Arc-enabled Kubernetes documentation](https://learn.microsoft.com/en-us/azure/aks/dapr-overview)

## Troubleshooting

### Connection Issues

**Error: "connection to the server was refused"**
- **Check your kubectl context**: Run `kubectl config current-context` to see which cluster you're connected to
- **Verify available contexts**: Run `kubectl config get-contexts` to see all available clusters
- **Switch to the correct context**: If you see the wrong cluster (e.g., `minikube` instead of your AKS cluster), switch contexts:
  ```bash
  kubectl config use-context <your-aks-cluster-context>
  ```
- **Re-configure cluster access**: If your AKS cluster isn't listed, re-run the credentials command:
  ```bash
  az aks get-credentials --resource-group <your-resource-group> --name <your-cluster-name> --overwrite-existing
  ```

### Workflow API Issues

**Error: "failed getting app id either from the URL path or the header dapr-app-id"**
- **Missing header**: Ensure you include the `dapr-app-id: dwf-app` header in all workflow API requests
- **Incorrect URL format**: Use the correct URL pattern: `/v1.0-alpha1/workflows/dapr/OrderProcessingWorkflow/start`

**Error: "No orchestrator task named 'X' was found"**
- **Wrong URL structure**: This usually indicates the instance ID is being interpreted as the workflow name
- **Use correct API format**: Ensure you're using `/workflows/dapr/OrderProcessingWorkflow/start` not `/workflows/OrderProcessingWorkflow/{instance-id}/start`

### Workflow Execution Failures

**Error: "Object reference not set to an instance of an object" in ReserveInventoryActivity**
- **Missing inventory data**: Run the restock command first: `curl -X GET $SAMPLE_APP_URL/stock/restock`
- **Incorrect JSON format**: Ensure your input JSON doesn't have a nested `"input"` wrapper. Use:
  ```json
  {"Name": "Paperclips", "TotalCost": 99.95, "Quantity": 1}
  ```
  Not:
  ```json
  {"input": {"Name": "Paperclips", "TotalCost": 99.95, "Quantity": 1}}
  ```

**Workflow shows logs with null values**
- **Input parsing issue**: Check that you're using the correct JSON format without the `"input"` wrapper
- **Old workflow instances**: If you see logs with null values, they might be from previous failed attempts with incorrect input format

### Service Discovery Issues

**Cannot get external IP for services**
- **LoadBalancer provisioning**: Azure LoadBalancers can take several minutes to provision external IPs
- **Check service status**: Run `kubectl get svc` and look for `<pending>` in the EXTERNAL-IP column
- **Wait for provisioning**: Use `kubectl get svc -w` to watch for IP assignment
- **Alternative access**: Use port-forwarding as a temporary workaround:
  ```bash
  kubectl port-forward svc/workflows-sample 8080:80
  # Then use localhost:8080 instead of the external IP
  ```

### Debugging Commands

**Check application logs**:
```bash
kubectl logs -l run=workflows-sample -c workflows-sample --tail=50 --timestamps=true
```

**Check Dapr sidecar logs**:
```bash
kubectl logs -l run=workflows-sample -c daprd --tail=50
```

**Verify Redis connectivity**:
```bash
kubectl get pods -l app.kubernetes.io/name=redis
kubectl get components
```

**Check workflow execution in Dapr logs**:
```bash
kubectl logs -l run=workflows-sample -c daprd | grep -i workflow
```