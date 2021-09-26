# eks-nginx-ingress
This document will guide you through one of the approach (best so far) to utilise AWS Load Balancer ( NLB) with Nginx ingress controller 

# Configuring Nginx controller on EKS

### 1. Choose the right fit you.
##### There are 3 approaches that can be use to configure AWS Load Balancer with Nginx controller on EKS (K8s) enviroment:
#### a. Autodeploy AWS NLB
In AWS we can use a Network load balancer (NLB) to expose the NGINX Ingress controller behind a Service of Type=LoadBalancer using the "annotation service.beta.kubernetes.io/aws-load-balancer-type: nlb"
```
Example:
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: 'true'
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
```
> kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.1/deploy/static/provider/aws/deploy.yaml

#### b. Autodeploy AWS Load Balancer (ELB) with TLS termination
The problem is that it creates a "Classic Load Balancer" instaed of ELB. 
**There are similar issues reported on github- https://github.com/kubernetes/ingress-nginx/issues/6292**

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
    service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: '60'
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: 'true'
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-west-2:XXXXXXXX:certificate/XXXXXX-XXXXXXX-XXXXXXX-XXXXXXXX
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: https
    service.beta.kubernetes.io/aws-load-balancer-type: elb
```
>wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.1/deploy/static/provider/aws/deploy-tls-termination.yaml



Edit the file and change:
```
VPC CIDR in use for the Kubernetes cluster:- proxy-real-ip-cidr: XXX.XXX.XXX/XX
AWS Certificate Manager (ACM) ID:- arn:aws:acm:us-west-1:XXXXXXXX:certificate/XXXXXX-XXXXXXX-XXXXXXX-XXXXXXXX
```
Deploy the manifest:
> kubectl apply -f deploy-tls-termination.yaml

#### c. Bare Metal approach - I will choose this approach over others. If you want to use an ALB/NLB, I recommend change the service type to NodePort instead of LoadBalancer and create the ALB with terraform, cloudformation, or something else.
##### Creating Ingress controller and NodePort service
This will create an Ingress controller ( pod), create a NodePort service and expose the ad-hoc ports.

> kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.1/deploy/static/provider/baremetal/deploy.yaml
```
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.1/deploy/static/provider/baremetal/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
deployment.apps/ingress-nginx-controller created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
serviceaccount/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
```
```
# kubectl get ns
NAME              STATUS   AGE
default           Active   27m
ingress-nginx     Active   29s
kube-node-lease   Active   27m
kube-public       Active   27m
kube-system       Active   27m
# kubectl get po -ningress-nginx
NAME                                       READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-h7hlh       0/1     Completed   0          69s
ingress-nginx-admission-patch-99qh9        0/1     Completed   1          69s
ingress-nginx-controller-db898f6c7-mvz9x   1/1     Running     0          70s
# kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.100.89.81     <none>        80:31848/TCP,443:30099/TCP   9m4s
ingress-nginx-controller-admission   ClusterIP   10.100.108.160   <none>        443/TCP                      9m4s
#
```

##### Creating Production deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production
  labels:
    app: production
  namespace: app 
spec:
  replicas: 1
  selector:
    matchLabels:
      app: production
  template:
    metadata:
      labels:
        app: production
    spec:
      containers:
      - name: production
        image: mirrorgooglecontainers/echoserver:1.10
        ports:
        - containerPort: 8080
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP

---

apiVersion: v1
kind: Service
metadata:
  name: production
  labels:
    app: production
  namespace: app
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: production
---

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: production
  annotations:
    kubernetes.io/ingress.class: nginx
  namespace: app
spec:
  rules:
  - http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            serviceName: production
            servicePort: 80
```
##### Creating Canary deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary
  labels:
    app: canary
  namespace: app 
spec:
  replicas: 1
  selector:
    matchLabels:
      app: canary
  template:
    metadata:
      labels:
        app: canary
    spec:
      containers:
      - name: canary
        image: mirrorgooglecontainers/echoserver:1.10
        ports:
        - containerPort: 8080
        env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
---

apiVersion: v1
kind: Service
metadata:
  name: canary
  labels:
    app: canary
  namespace: app
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: canary
---

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: canary
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "30"
  namespace: app
spec:
  rules:
  - http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            serviceName: canary
            servicePort: 80
```

##### Create a NLB Load Balancer.
##### Creating a Target Group.
##### Mapping this Target Group to the Auto Scaling Group.
##### Create a listener on NLB ( TLS (Secure TCP) and forward to the Target Group created above.
##### Go the Security group of worker nodes and open the port "31848" for all the IPs. "31848" is our ingress controller service HTTP Port.

##### Make requests to the NLB endpoint.

> https://mysaasapp-1fe65deyetdlssmsbd.elb.us-east-1.amazonaws.com/
```
Hostname: canary-6cc5497cfd-66878

Pod Information:
	node name:	ip-172-23-81.101.ec2.internal
	pod name:	canary-6cc5497cfd-66878
	pod namespace:	app
	pod IP:	172.23.81.101

Server values:
	server_version=nginx: 1.13.3 - lua: 10008

Request Information:
	client_address=172.23.81.104
	method=GET
	real path=/
	query=
	request_version=1.1
	request_scheme=http
	request_uri=http://mysaasapp-1fe65deyetdlssmsbd.elb.us-east-1.amazonaws.com:8080/

Request Headers:
	accept=text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
	accept-encoding=gzip, deflate, br
	accept-language=en-GB,en-US;q=0.9,en;q=0.8
	cache-control=max-age=0
	host=mysaasapp-1fe65deyetdlssmsbd.elb.us-east-1.amazonaws.com
	sec-ch-ua=&quot;Google Chrome&quot;;v=&quot;93&quot;, &quot; Not;A Brand&quot;;v=&quot;99&quot;, &quot;Chromium&quot;;v=&quot;93&quot;
	sec-ch-ua-mobile=?0
	sec-ch-ua-platform=&quot;macOS&quot;
	sec-fetch-dest=document
	sec-fetch-mode=navigate
	sec-fetch-site=none
	sec-fetch-user=?1
	upgrade-insecure-requests=1
	user-agent=Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/93.0.4577.63 Safari/537.36
	x-forwarded-for=172.23.81.97
	x-forwarded-host=infosaas-1fe65d89882739bd.elb.us-east-1.amazonaws.com
	x-forwarded-port=80
	x-forwarded-proto=http
	x-forwarded-scheme=http
	x-real-ip=172.23.81.97
	x-request-id=937fc6974d0a47f929df4f7a4a3fabe1
	x-scheme=http

Request Body:
	-no body in request-
```


## License
Apache
