# Introduction

Troubleshooting can be divided into two parts. 
1. At K8S Cluster level
2. Applications level

# Troubleshooting Cluster

1. Check health of the K8S cluster APIs using the command  ** kubectl get --raw='/readyz?verbose' **  

2. Check if nodes are registered correctly using command ** kubectl get nodes **. Make sure all nodes are in Ready state. 
   If any node has status other than **Ready**, Try to get more info using commands ** kubectl describe node <node-name> ** check at Events section.
   Get the config settings of node using ** kubectl get node <node-name> -o yaml ** . Check for any syntax / configuration issues. 
   
3. Check clusterwide events **kubectl get events**. Try to get overall health of cluster using ** kubectl cluster-info dump ** . By default it dumps to the stdout, Optionally we can choose json/yaml/go formats with -o. 
   Check if any events/status reported at the cluster-dump file. Logs for all pods of kube-system namespace wil be reported between START logs & END logs lines.


4. If still need to dig the cluster, logs of individul components should be checked. 
   Check if services(ex:kubelet) are managed by systemd using **systemctl --type=service** or **systemctl --type=service --state=active**. Use journalctl(ex:journalctl -xeu kubelet) to examine the logs.
   Logs of pods/contaners & containerd.log will be available under /var/log at individual nodes for troubleshooting.   
   
5. If a node can't be accessed with ssh, **kubectl debug node(ex: kubectl debug node/<node-name> -it --image=ubuntu)** command to deploy a Pod to a Node that you want to troubleshoot.
   Some commands will need privileged access, use ** --privileged ** flag to have access to superuser rights.

6. Check for other issues like VM down, Kubelet is up, proper CNI setup, Network setup(sufficient number IPs are available for new PODs) and storage unavailability etc.  
   * Check if VM is up at monitoring tool, using ping/telnet <node-name>.domain.com.  
   * If VM is reachbale, login using SSH and check since when it is up. command "uptime" or "uptime -s".
   * Check the memory usage with "free -m"
   * Check if server load is fine using "Top", check Load average & %Cpu(s) etc 
   * Check Kubelet is running or not. "ps -aux| grep kubelet" or "systemctl status kubelet".  
   * CNI: Even CNI pod is running there can be issues with CNI config file. Check CNI configuration at /etc/cni/net.d

# Troubleshooting Applications

1. Deployments: 
   Applications are managed by Deployments/statefulsets. So, check the deployment status using **kubectl get deploy -n <namespace>**.  
   Output should show below info: 
    NAME lists the names of the Deployments in the namespace.
    READY displays how many replicas of the application are available to your users. It follows the pattern ready/desired.
    UP-TO-DATE displays the number of replicas that have been updated to achieve the desired state.
    AVAILABLE displays how many replicas of the application are available to your users.
    AGE displays the amount of time that the application has been running.
	
	* Check deployment rollout status using **kubectl rollout status deployment/<deployment-name>**  
	* Run "kubectl describe deployment <deployment-name>" to get details of the Deployment.  
	
2. ReplicaSet:
   Every Deployment will have corresponding ReplicaSet/s created. To see the ReplicaSet (rs) created by the Deployment, run **kubectl get rs**  
   ReplicaSet output shows the following fields:

    NAME lists the names of the ReplicaSets in the namespace.
    DESIRED displays the desired number of replicas of the application, which you define when you create the Deployment. This is the desired state.
    CURRENT displays how many replicas are currently running.
    READY displays how many replicas of the application are available to your users.
    AGE displays the amount of time that the application has been running.
	
	ReplicaSet is always formatted as <DEPLOYMENT-NAME>-<HASH>. This name will become the basis for the Pods which are created.
	The HASH string is the same as the pod-template-hash label on the ReplicaSet. run "kubectl get pods --show-labels" to see 

	
3. PODS: 
   Pods in Deployment are managed by Replicaset. There should be same number of PODs available as shown by replicaset.
   Run "kubectl get pods" to see the list of pods and their details.  
   Few other commands for Debugging pods :  
   "kubectl describe pods <POD_NAME> -n <namespace>"  ==> Gives full details of POD(configuration,status,events)
   "kubectl get pod <POD_NAME> -n <namespace> -o yaml/json" ==> Save pod configuration to a local file.
   "kubectl get events -n <namespace>"  ==> Check systemwide events related to POD's namespace.  
   "kubectl logs <POD_NAME> -c <CONTAINER_NAME> -n <namespace>" ==> To look at the logs of affected container. 
   "kubectl logs --previous <POD_NAME> -c <CONTAINER_NAME> -n <namespace>"  ==> To check previous container's crash log.  
   "kubectl exec $<POD_NAME> -c <CONTAINER_NAME> -- ${CMD} ${ARG1}    ==> To run any bash commands in container.  

4. Debugging Services:  Services group PODs using labels. Selector of service should match the POD label.
   Few commands :  
   kubectl get svc -n argocd			==> Get the details of service 
   kubectl get endpoints <SERVICE_NAME>   ==> Check if endpoints match the number of pods and POD's IP,ports
   nslookup <SERVICE_NAME>  			==> Check if service is accessible using DNS
   ps auxw | grep kube-proxy			==> If svc is up/running
   curl <SERVICE_IP>:PORT				==> Try to access SVC from any node.  



# Observations/Report for Alpha cluster using given log files  

1. mission-control-alpha-production.log : 
   command used "kubectl describe deployment -n mission-control-alpha-production input-generator"  
   Replicas:               1 desired | 1 updated | 1 total | 0 available | 1 unavailable     ==> Indicates that desired POD is not available
   
   Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      False   MinimumReplicasUnavailable		
  
  The Deployment's .status.conditions says Replicaset was already created but MinimumReplicasUnavailable. Indicates POD is not up/running.  

  Events:          <none> ==> Indicates Deployment was previously existed,successful. 
                             Kubernetes node is just started,waiting/waited for PODs to be available and POD is not available. 
                             Check replicaset status and pod status(kubectl describe pod <pod-name>	-n <ns> and check Events section for clue.  
							 Could be an issue due to kubelet not up or NetworkPluginNotReady(CNI) is not ready  

   **Troubleshooting commands/steps:**  
     kubectl get deploy -n mission-control-alpha-production input-generator            ==> Check under READY,AVAILABLE,AGE 
	 kubectl get rs input-generator-6bb9f97576 -n mission-control-alpha-production  ==> Get the name of pods  
	 kubectl -n mission-control-alpha-production describe pod <pod-name>   ==> check Status,State,Node-Selectors,Tolerations,Events for any clue  
	 kubectl logs <POD_NAME> -c <CONTAINER_NAME> -n mission-control-alpha-production  ==> Check if any errors in logs  
	 kubectl get events -n mission-control-alpha-production   ==> Check all the events related to APP namespace.
	 kubectl get events --sort-by=.metadata.creationTimestamp       ==> Check clusterwide events order by timestamp
	 
	 systemctl status kubelet , ps aux | grep kubelet       ==> Check if kubelet is up & running 
	 journalctl -xeu kubelet  								==> Check kubelete logs for any 
	 
   
   
							 
							
  
  
   

