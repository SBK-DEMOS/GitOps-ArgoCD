Requirements

To follow this tutorial you’ll need the following. The version number shows what I’ve used for this tutorial: 

* A Kubernetes cluster (minikube works well) 
* kubectl (kubeconfig is pointing to the target k8s cluster)  
* A public git repository 

1. Make sure K8S cluster is ready.  

2. Install ArgoCD or K8S. 
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml  
```

3. Fetch ArgoCD default login credentials(username: admin) 

Linux: 
```
kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo  
```

Windows: 
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"  
```
Decode the encoded admin user password from above step using any online tools.  

4. Expose ArgoCD service to the external world.  
If using cloud based K8S:  
```  
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
5. Deploy root app (app-of-apps) in ArgoCD. Refer [root-app.yaml](https://github.com/SBK-DEMOS/GitOps-ArgoCD/blob/main/1.AutoDeploy/child-apps/root-app.yaml)  
```  
kubectl apply -f root-app.yml
```  


