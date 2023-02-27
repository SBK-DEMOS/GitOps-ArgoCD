# Introduction

Troubleshooting can be divided into mainly two levels.  

1. At K8S Cluster level  
2. Applications level  

# Troubleshooting Cluster

1. Check health of the K8S cluster APIs using the command  
```  
kubectl get --raw='/readyz?verbose'  
```  
2. Check if nodes are registered correctly using command  **kubectl get nodes**. Make sure all nodes are in Ready state. 
   If any node has status other than **Ready**, Try to get more info using commands **kubectl describe node ${node-name}** check at Events section.
   Get the config settings of node using **kubectl get node ${node-name} -o yaml**. Check for any syntax / configuration issues. 
   
3. Check clusterwide events with **kubectl get events**. Try to get overall health of cluster using **kubectl cluster-info dump**.  
   By default it dumps to the stdout, Optionally we can choose json/yaml/go formats with -o and save it to a file.  
   Check if any events/status reported at the cluster-dump file. Logs for all pods of kube-system namespace wil be reported between **START logs** & **END logs**          lines.


4. If still need to dig the cluster, logs of individul components should be checked. 
   Check if services(ex:kubelet) are managed by systemd using **systemctl --type=service** or **systemctl --type=service --state=active**. 
   Use journalctl(ex:**journalctl -xeu kubelet**) to examine the logs.
   Logs of pods/contaners & containerd.log will be available under /var/log at individual nodes for troubleshooting.   
   
5. If a node can't be accessed with ssh, **kubectl debug node(ex: kubectl debug node/${node-name} -it --image=ubuntu)** command to deploy a Pod to a Node that you want to troubleshoot.
   Some commands will need privileged access, use **--privileged** flag to have access to superuser rights.

6. Check for other issues like VM down, Kubelet is up, proper CNI setup, Network setup(sufficient number IPs are available for new PODs) and storage unavailability etc.  
   * Check if VM is up at monitoring tool, using ping/telnet ${node-name}.domain.com.  
   * If VM is reachbale, login using SSH and check since when it is up. command "uptime" or "uptime -s".
   * Check for logs at /var/log/messages
   * Check the memory usage with "free -m"
   * Check if server load is fine using "Top", check Load average & %Cpu(s) etc 
   * Check Kubelet is running or not. **ps -aux| grep kubelet** or **systemctl status kubelet**.   
   * CNI: Even CNI pod is running there can be issues with CNI config file. Check CNI configuration at /etc/cni/net.d

# Troubleshooting Applications

1. Deployments: 
   Applications are managed by Deployments/statefulsets. So, check the deployment status using **kubectl get deploy -n ${namespace}**.  
   Output should show below info: 
    NAME lists the names of the Deployments in the namespace.
    READY displays how many replicas of the application are available to your users. It follows the pattern ready/desired.
    UP-TO-DATE displays the number of replicas that have been updated to achieve the desired state.
    AVAILABLE displays how many replicas of the application are available to your users.
    AGE displays the amount of time that the application has been running.
	
	* Check deployment rollout status using **kubectl rollout status deployment/${deployment-name}**  
	* Run **kubectl describe deployment ${deployment-name}** to get details of the Deployment.  
	
2. ReplicaSet:
   Every Deployment will have corresponding ReplicaSet/s created. To see the ReplicaSet (rs) created by the Deployment, run **kubectl get rs**  
   ReplicaSet output shows the following fields:

    **NAME** lists the names of the ReplicaSets in the namespace.  
    **DESIRED** displays the desired number of replicas of the application, which you define when you create the Deployment. This is the desired state.  
    **CURRENT** displays how many replicas are currently running.  
    **READY** displays how many replicas of the application are available to your users.  
    **AGE** displays the amount of time that the application has been running.  
	
	ReplicaSet is always formatted as **${deployment-name}-{HASH}**. This name will become the basis for the Pods which are created.
	The HASH string is the same as the pod-template-hash label on the ReplicaSet. run "kubectl get pods --show-labels" to see 

	
3. PODS: 
   Pods in Deployment are managed by Replicaset. There should be same number of PODs available as shown by replicaset.
   Run "kubectl get pods" to see the list of pods and their details.  
   Few other commands for Debugging pods :  
   **kubectl describe pods POD-NAME -n namespace**  ==> Gives full details of POD(configuration,status,events)  
   **kubectl get pod POD-NAME -n namespace -o yaml/json** ==> Save pod configuration to a local file.  
   **kubectl get events -n namespace**  ==> Check systemwide events related to POD's namespace.  
   **kubectl logs POD-NAME -c CONTAINER-NAME -n namespace** ==> To look at the logs of affected container.  
   **kubectl logs --previous POD_NAME -c CONTAINER-NAME -n namespace**  ==> To check previous container's crash log.   
   **kubectl exec PODNAME -c CONTAINER-NAME -- CMD ARG1**    ==> To run any bash commands in container.  

4. Debugging Services:  Services group PODs using labels. Selector of service should match the POD label.  
   Few commands :  
   **kubectl get svc -n ${namespace}**			==> Get the details of service  
   **kubectl get endpoints ${SERVICE_NAME}**   ==> Check if endpoints match the number of pods and POD's IP,ports  
   **nslookup ${SERVICE_NAME}** 			==> Check if service is accessible using DNS  
   **ps auxw | grep kube-proxy**			==> If svc is up/running  
   **curl ${SERVICE_IP}:PORT**				==> Try to access SVC from any node.  



# Observations/Report for Alpha cluster using given log files  

1. mission-control-alpha-production.log :  
   Command used **kubectl describe deployment -n mission-control-alpha-production input-generator**  
   
   Replicas:               1 desired | 1 updated | 1 total | 0 available | **1 unavailable**    ==> Indicates that desired POD is not available
   
```  
   Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable  
  Available      False   MinimumReplicasUnavailable  	
```  
  The Deployment's .status.conditions says Replicaset was already created but Available has Status: False, Reason **MinimumReplicasUnavailable**. Indicates POD is not up/running.  

```  
  Events:          <none>  
```  
It Indicates Deployment was previously existed,successful and no changes performed to deployments since last revision.    
Kubernetes node is just started,waiting/waited for PODs to be available and POD is not available.  
In this case we need to get more details to get the root cause.  
check replicaset status and pod status **kubectl describe pod $pod-name -n mission-control-alpha-production** and check Events section for clue. Could be an issue due to kubelet not up or NetworkPluginNotReady(CNI) is not ready.   

   **Troubleshooting commands/steps:**  
       **kubectl get deploy -n mission-control-alpha-production input-generator**            ==> Check under READY,AVAILABLE,AGE  
       **kubectl get rs input-generator-6bb9f97576 -n mission-control-alpha-production**  ==> Get the name of pods  
       **kubectl -n mission-control-alpha-production describe pod ${pod-name}**   ==> check Status,State,Node-Selectors,Tolerations,Events for any clue   
       **kubectl logs POD_NAME -c CONTAINER_NAME -n mission-control-alpha-production**  ==> Check if any errors in logs  
       **kubectl get events -n mission-control-alpha-production**   ==> Check all the events related to APP namespace.  
       **kubectl get events --sort-by=.metadata.creationTimestamp**      ==> Check clusterwide events order by timestamp  
	 
   **systemctl status kubelet** or **ps aux | grep kubelet**      ==> Check if kubelet is up and running  
   **journalctl -xeu kubelet** 				==> Check kubelete logs for any errors.  
	 
   If POD description shows below Event, wait for few mins for CNI to be UP.  
```   
 Events:
  Type     Reason           Age                    From     Message
  ----     ------           ----                   ----     -------
  Warning  NetworkNotReady  20h (x2 over 20h)      kubelet  network is not ready: container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized  
```  

2. telemetry-api.log :  
```  
   kubectl describe deployment telemetry-api -n telemetry-mars-production  
```  
Replicas:   1 desired | 1 updated | 1 total | 1 available | 0 unavailable    ==> Indicates APP is UP and running.  

```  
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   telemetry-api-79d4799bff (1/1 replicas created)
Events:          <none>

```  
The above outputs indicate APP pods are self-healed.  

just check and confirm using **kubectl describe pod PODNAME -n telemetry-mars-production** , check at Events section.  


3. telemetry-beacon-listener.log:  

```  
   kubectl describe deployment beacon-listener -n telemetry-mars-production  
```  
Replicas:   1 desired | 1 updated | 1 total | 1 available | 0 unavailable    ==> Indicates APP is UP and running.  

```  
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   beacon-listener-8495cbbb94 (1/1 replicas created)
Events:          <none>

```  
The above outputs indicate APP pods are self-healed.  

just check and confirm using **kubectl describe pod PODNAME -n telemetry-mars-production** , check at Events section.  


4. telemetry-beacon-listener-8495cbbb94-5d7pm.log:  
   Command **kubectl describe pod beacon-listener-8495cbbb94-5d7pm -n telemetry-mars-production**   

```  
Status:         Failed
Reason:         Evicted
Message:        The node was low on resource: ephemeral-storage. Container istio-proxy was using 120Ki, which exceeds its request of 0. Container beacon-listener was using 316Ki, which exceeds its request of 0.  
```  

Reason: Pods are being evicted and are getting an ephemeral storage error.  

Solution:  Define ResourceQuota or Limit range or at resources section of Pod spec.  

Example:
```  
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-resource-quotas
  namespace: my-application-namespace
spec:
  hard:
    limits.ephemeral-storage: 2Mi
    requests.ephemeral-storage: 1Mi
    ...

---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limit-ranges
  namespace: my-application-namespace
spec:
  limits:
  - default:
      cpu: 100m
      memory: 128Mi
      ephemeral-storage: "2Mi"
    defaultRequest:
      cpu: 25m
      memory: 64Mi
      ephemeral-storage: "1Mi" 
   type: Container
```  

5. cattle-system-cattle-cluster-agent.log:  

Command **kubectl describe deployment cattle-cluster-agent -n cattle-system **

```  
Selector:               app=cattle-cluster-agent
Replicas:               2 desired | 2 updated | 2 total | 1 available | 1 unavailable
---

Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      False   MinimumReplicasUnavailable
OldReplicaSets:  <none>
NewReplicaSet:   cattle-cluster-agent-547cb64d75 (2/2 replicas created)
Events:          <none>

```  
From the above output we can understand that pods under cattle-cluster-agent-547cb64d75 are Unavailable.  

kubectl get rs cattle-cluster-agent-547cb64d75 -n cattle-system    ==> Get the 
kubectl get pods -n cattle-system -l app=cattle-cluster-agent      ==> Get the details of pods with label app=cattle-cluster-agent  

kubectl get logs -f PODNAME                                      ==> Check the logs of POD
kubectl logs -f -n cattle-system cattle-cluster-agent-547cb64d75-5h7Ih

6. cattle-system-cattle-cluster-agent-547cb64d75-5h7Ih.log:  

Command **kubectl logs -f -n cattle-system cattle-cluster-agent-547cb64d75-5h7Ih**  

```  
W0110 05:46:13.543408      56 warnings.go:80] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+  
W0112 15:12:22.287473      56 warnings.go:80] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0112 15:20:33.289336      56 warnings.go:80] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+

```  

Solution:  
PodSecurityPolicy is deprecated, use [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/).    


```  
W0112 15:11:52.330069      56 warnings.go:80] batch/v1beta1 CronJob is deprecated in v1.21+, unavailable in v1.25+; use batch/v1 CronJob
W0112 15:21:10.331648      56 warnings.go:80] batch/v1beta1 CronJob is deprecated in v1.21+, unavailable in v1.25+; use batch/v1 CronJob
```  

Solution:  
**batch/v1beta1** API version of CronJob is deprecated. 
Migrate manifests and API clients to use the **batch/v1** API version, available since v1.21.  

**kubectl get job JOBNAME -o yaml**   ==> Get the Job configuration in yaml and modify **apiVersion: batch/v1beta1** to **apiVersion: batch/v1**

Example:  
```  
apiVersion: batch/v1
kind: CronJob
```  
7. cattle-system-cattle-cluster-agent-547cb64d75-w2s9v.log:  

command used **kubectl logs -f -n cattle-system cattle-cluster-agent-547cb64d75-w2s9v**  

```  
time="2023-01-12T15:20:45Z" level=info msg="NotBefore: 2021-12-23 11:28:50 +0000 UTC"
time="2023-01-12T15:20:45Z" level=info msg="NotAfter: 2022-12-23 11:28:50 +0000 UTC"
time="2023-01-12T15:20:45Z" level=info msg="SignatureAlgorithm: SHA256-RSA"
time="2023-01-12T15:20:45Z" level=info msg="PublicKeyAlgorithm: RSA"
time="2023-01-12T15:20:45Z" level=fatal msg="Server certificate is not valid, please check if the host has the correct time configured and if the server certificate has a notAfter date and time in the future. Certificate information is displayed above. error: Get \"https://rancher.iceye.edge\": x509: certificate has expired or is not yet valid: current time 2023-01-12T15:20:45Z is after 2022-12-23T11:28:50Z"
```  

current time 2023-01-12T15:20:45Z is after 2022-12-23T11:28:50Z"   ==> It shows certificate is expired. It looks apiserver.crt is  

**kubeadm certs check-expiration**   ==> Check valididy of all certs

Backup CERTs and config files.  

**kubeadm certs renew apiserver** or **kubeadm certs renew all**     ==> Renew all CERTs managed by kubeadm. Check new certs under /etc/kubernetes/pki   

output: Done renewing certificates. You must restart the kube-apiserver, kube-controller-manager, kube-scheduler and etcd, so that they can use the new certificates.  

Restart control plane components.  If CERTs signed/managed by external vendors, Rotate accordingly.  


8. main-pgbackrest-repo1-incr-27863051-vj5ft.log  

command used **kubectl describe pod main-pgbackrest-repo1-incr-27863051-vj5ft -n postgres-operator**  

```  
Controlled By:  Job/main-pgbackrest-repo1-incr-27863051  
```  

It indicates that the pod is part of a Job named main-pgbackrest-repo1-incr-27863051.

**kubectl get jobs**  ==> Get the job name and COMPLETION info.  
**kubectl describe job main-pgbackrest-repo1-incr-27863051**  ==>  Check Completions:, Pods Statuses:, Events: in the job description output  
**kubectl get pods | grep main-pgbackrest-repo1-incr-27863051**  ==>  Get the pod names for job 
 
The pod description(kubectl describe pod main-pgbackrest-repo1-incr-27863051-vj5ft -n postgres-operator) output gives info for troubleshooting.  

```  
Containers:
  pgbackrest:
  ......
    State:          Waiting
      Reason:       ImagePullBackOff  
  ......  

Events:
  Type    Reason   Age                     From     Message
  ----    ------   ----                    ----     -------
  Normal  BackOff  50s (x112031 over 20d)  kubelet  Back-off pulling image "registry.developers.crunchydata.com/crunchydata/crunchy-pgbackrest:centos8-2.36-0"

```  

The above output indicates that Container creation was failed.  
The **ImagePullBackOff** error is a sign that something is blocking Kubernetes(kubelet) from being able to pull the image you want onto a specific node.  

Reasons could be:  

The image or tag doesn’t exist  

You’ve made a typo in the image name or tag  

The image registry requires authentication  

You’ve exceeded a rate or download limit on the registry  

**Solutions:**  

  * If it’s a private registry, check your Secret and Pod. If you’re pulling an image from a private registry, make sure that you’ve created a Secret containing the credentials you need to acceess.  
  * Set the imagePullSecrets field on your Pod. This field tells Kubernetes which Secret it should use, when authenticating to the registry.  

Example:  

```  
spec:
  containers:
    - name: myapp
      image: registry.example.com/myteam/myapp:latest
      imagePullPolicy: Always
  imagePullSecrets:
  - name: secret-name  

```  
  * Are there other Pods already running on that node? If the node is already running other Pods fine, then are they pulling from the same registry? What is different about this Pod?  
  * Try to pull the image locally? Can you pull the image from your own workstation? Try docker pull or podman pull and see if you can fetch the image. This might give you a clue as to why it’s failing.
  * Can the node in the cluster pull the image? Try jumping on to the node itself, via SSH. Can you run a docker pull and get the image directly?  
  

8. minio-isilon-pool-0-1.log:  

command used **kubectl describe -n minio pod isilon-pool-0-1**  

```  
Status:         Pending
IP:             
IPs:            <none>
Controlled By:  StatefulSet/isilon-pool-0
```  
**kubectl get sts -A**   ==> Get all the statefulsets in the cluster.  
**kubectl describe sts isilon-pool-0 -n minio**   ==> Check Replicas: , Pods Status: , Events:  sections in the output of describe sts to understand.  
**kubectl describe -n minio pod isilon-pool-0-1**  ==> To get the details of POD status.  


```  
Events:
  Type     Reason              Age                      From                     Message
  ----     ------              ----                     ----                     -------
  Warning  FailedAttachVolume  57m                      attachdetach-controller  AttachVolume.Attach failed for volume "k8s-10f368fb82" : rpc error: code = FailedPrecondition desc =  runid=226485 export '35' in access zone 'az-iceye' already has other clients added to it, and the access mode is SINGLE_NODE_MULTI_WRITER, thus the request fails
  Warning  FailedAttachVolume  51m (x2 over 53m)        attachdetach-controller  AttachVolume.Attach failed for volume "k8s-980e961bec" : rpc error: code = FailedPrecondition desc =  runid=226504 export '33' in access zone 'az-iceye' already has other clients added to it, and the access mode is SINGLE_NODE_MULTI_WRITER, thus the request fails
  Warning  FailedAttachVolume  27m (x2 over 29m)        attachdetach-controller  AttachVolume.Attach failed for volume "k8s-980e961bec" : rpc error: code = FailedPrecondition desc =  runid=226600 export '33' in access zone 'az-iceye' already has other clients added to it, and the access mode is SINGLE_NODE_MULTI_WRITER, thus the request fails
  Warning  FailedAttachVolume  20m (x117964 over 60d)   attachdetach-controller  (combined from similar events): AttachVolume.Attach failed for volume "k8s-980e961bec" : rpc error: code = FailedPrecondition desc =  runid=226628 export '33' in access zone 'az-iceye' already has other clients added to it, and the access mode is SINGLE_NODE_MULTI_WRITER, thus the request fails
  Warning  FailedAttachVolume  16m                      attachdetach-controller  AttachVolume.Attach failed for volume "k8s-980e961bec" : rpc error: code = FailedPrecondition desc =  runid=226628 export '33' in access zone 'az-iceye' already has other clients added to it, and the access mode is SINGLE_NODE_MULTI_WRITER, thus the request fails
  Warning  FailedAttachVolume  12m                      attachdetach-controller  AttachVolume.Attach failed for volume "k8s-9fac5ccff0" : rpc error: code = FailedPrecondition desc =  runid=226649 export '30' in access zone 'az-iceye' already has other clients added to it, and the access mode is SINGLE_NODE_MULTI_WRITER, thus the request fails
  Warning  FailedAttachVolume  6m36s (x2 over 8m38s)    attachdetach-controller  AttachVolume.Attach failed for volume "k8s-980e961bec" : rpc error: code = FailedPrecondition desc =  runid=226664 export '33' in access zone 'az-iceye' already has other clients added to it, and the access mode is SINGLE_NODE_MULTI_WRITER, thus the request fails
  Warning  FailedMount         5m11s (x38631 over 60d)  kubelet                  (combined from similar events): Unable to attach or mount volumes: unmounted volumes=[data0 data3 data1 data2], unattached volumes=[workload-certs data0 data3 data1 data2 istio-data istio-envoy istio-podinfo kube-api-access-98xxh workload-socket istiod-ca-cert istio-token isilon-tls]: timed out waiting for the condition
  Warning  FailedAttachVolume  2m31s                    attachdetach-controller  AttachVolume.Attach failed for volume "k8s-980e961bec" : rpc error: code = FailedPrecondition desc =  runid=226684 export '33' in access zone 'az-iceye' already has other clients added to it, and the access mode is SINGLE_NODE_MULTI_WRITER, thus the request fails

```  

  * Above logs tell that statefulset pod creation in pending as the ReadWriteOnce volumes are already being used by another node.  
  * ReadWriteOnce mode enables volumes' access to multiple pods running on the same node where as volumes can't be shared with different nodes.  
  * Use ReadOnlyMany or ReadWriteMany access mode for volumes if the volumes are shared by many nodes.  
  * Use ReadWriteOncePod access mode if you want to ensure that only one pod across whole cluster can read that PVC or write to it.  
  
  
  




  





















							 
							
  
  
   

