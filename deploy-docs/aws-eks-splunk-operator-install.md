# Deploying EKS Cluster and Splunk Operator 

NOTE: This doc precedes the aws-eks-splunk-operator-install.md and tls-configuration-letsencrypt.md document.

## Pre-Reqs
1. AWS account with appropriate permissions 
2. awscli installed and configured
3. kubectl and eksctl installed/up-to-date

This document largely draws on the splunk-operater docs found here: https://splunk.github.io/splunk-operator/

## Actions 
1. Create the AWS EKS configuration file 

The following config file will create a single nodeGroup with 2 m5.large instances. 

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: splunk-dev
  region: us-east-2

iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: eks-ebs-csi-driverrole 
      namespace: kube-system
    attachPolicyARNs:
    - "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"

addons: 
- name: "aws-ebs-csi-driver"
  resolveConflicts: overwrite

nodeGroups:
  - name: splunk-dev-node-group
    instanceType: m5.large
    desiredCapacity: 2
    volumeSize: 80
    iam:
      withAddonPolicies:
        ebs: true
```

2. Deploy an EKS cluster in AWS 
```
eksctl create cluster -f <config.yaml>
``` 

3. Deploy the splunk-operator manifest
```
kubectl create -n splunk-operator  -f https://github.com/splunk/splunk-operator/releases/download/2.2.1/splunk-operator-namespace.yaml
```

Use `kubectl` to validate that the splunk-operator pod is up and running
```
splunk-k8s % kubectl get pods -n splunk-operator     
NAME                                                  READY   STATUS    RESTARTS   AGE
splunk-operator-controller-manager-5c8987564c-lv5sk   2/2     Running   0          30m
```


4. Deploy a standalone instance: 

```
cat <<EOF | kubectl apply -n splunk-operator -f -
apiVersion: enterprise.splunk.com/v4
kind: Standalone
metadata:
  name: s1
  finalizers:
  - enterprise.splunk.com/delete-pvc
EOF
```

5. Create simple port forward to enable Splunk UI access from outside the cluster 

```
kubectl port-forward -n splunk-operator splunk-s1-standalone-0 8000
```

6. Pull auto-generated Splunk secrets
```
kubectl get secret -n splunk-operator splunk-s1-standalone-secret-v1 -o jsonpath='{.data.password}' | base64 -d 
```

7. Enable access from outside the EKS Cluster (public) 
NOTE: Ensure the policy that is applied to the EKS cluster nodes allows inbound traffic from your IP address

```
kubectl patch svc splunk-s1-standalone-service -n splunk-operator -p '{"spec": {"type": "NodePort"}}' 
kubectl patch svc splunk-s1-standalone-service -n splunk-operator -p '{"spec":{"externalIPs":["18.222.200.164"]}}' 
```

8. Find the port that the application is published on by looking at mapping of PORTS in kubectl output:
Need to figure out how to grab this with jsonpath/yaml filter.. in the example below, the web interface (port 8000) is mapped to port 31243 on the node. 

```
splunk-k8s % kubectl get svc -n splunk-operator                                                           
NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP      PORT(S)                                                       AGE
splunk-operator-controller-manager-service   ClusterIP   10.100.237.141   <none>           8080/TCP,8081/TCP                                             76m
splunk-s1-standalone-headless                ClusterIP   None             <none>           8000/TCP,8088/TCP,8089/TCP,9997/TCP                           39m
splunk-s1-standalone-service                 NodePort    10.100.55.4      18.222.200.164   8000:31243/TCP,8088:30483/TCP,8089:32064/TCP,9997:30238/TCP   39m
```

9. Clean up 

To delete the cluster, use the following command: 

```
eksctl delete cluster -f <config.yaml>
```
