

# Exposing Splunk Services 

Use the following steps to deploy the nginx-ingress controller, which whill manage creation of AWS Network Load Balancer to manage access to the cluster. 

NOTE: This doc follows the aws-eks-splunk-operator-install document and precedes the 'tls-configuration-letsencrypt.md document.

##  NGINX 

1. Install NGINX Ingress Controller 
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/aws/deploy.yaml
```

2. Create the ingress resource  

Create the following file, swapping out the "host" field for a domain you control. 
You will need to create a CNAME record on your domain to point at the NLB created by the controller in step 1


```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-standalone
  namespace: splunk-operator
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/default-backend: splunk-standalone-standalone-service
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  rules:
  - host: splunk.defnotavirus.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: splunk-s1-standalone-service
            port:
              number: 8000
```

kubectl apply -f nginx-ingress.yaml


## Pure AWS 
(unsolved)
Installing the controller: https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html

1. Create a IAM Policy 
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

2. Create IAM Role 
```
eksctl create iamserviceaccount \
  --region=us-east-2
  --cluster=splunk-dev-5 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::413112485471:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
  ```

3. Install AWS Load Balancer Controller
```
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=splunk-dev-5 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller 
```

*The deployed chart doesn't receive security updates automatically. You need to manually upgrade to a newer chart when it becomes available. When upgrading, change install to upgrade in the previous command, but run the following command to install the TargetGroupBinding custom resource definitions before running the previous command.

4. Validate controller is installed 
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```