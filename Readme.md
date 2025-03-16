# To ensure that all pods in the namespace are secure, a best practice is to create a default network policy. This avoids accidentally exposing an app or version that doesn’t have policy defined.

# CHECKOUT EXAMPLE3 - advanced policy in k8 networking

# EXAMPLE 1 
Create deny-all default ingress and egress network policy
```kubectl apply -f create_namespace.yaml ```
```  kubectl apply -f deny_all_ingress-egress.yaml ```


Follow the link at https://docs.tigera.io/calico/latest/network-policy/get-started/kubernetes-policy/kubernetes-demo
Run the below commands
``` kubectl create -f https://docs.tigera.io/files/00-namespace.yaml ```
``` kubectl create -f https://docs.tigera.io/files/01-management-ui.yaml ```
``` kubectl create -f https://docs.tigera.io/files/02-backend.yaml ```
``` kubectl create -f https://docs.tigera.io/files/03-frontend.yaml ```
``` kubectl create -f https://docs.tigera.io/files/04-client.yaml ```

Should be visible at NodeIP at port 30002 in GUI http://NodeIP:30002/ 

Running following commands will prevent all access to the frontend, backend, and client Services.

``` kubectl create -n stars -f https://docs.tigera.io/files/default-deny.yaml ```
``` kubectl create -n client -f https://docs.tigera.io/files/default-deny.yaml ```
 
Apply the following YAML files to allow access from the management UI.
```  kubectl create -f https://docs.tigera.io/files/allow-ui.yaml ```
```  kubectl create -f https://docs.tigera.io/files/allow-ui-client.yaml ```

Create the backend-policy.yaml file to allow traffic from the frontend to the backend
``` kubectl create -f https://docs.tigera.io/files/backend-policy.yaml ```

Expose the frontend service to the client namespace
``` kubectl create -f https://docs.tigera.io/files/frontend-policy.yaml ```

Clean up all the setup using 
``` kubectl delete ns client stars management-ui ```


# EXAMPLE 2 
Before running it ensure that default-deny in given or All namespace is NOT active
 ``` kubectl get networkpolicies -n policy-demo ``` 
 ``` NAME           POD-SELECTOR   AGE ``` 
 ``` default-deny   <none>         39m ``` 
 ``` kubectl delete networkpolicies default-deny -n policy-demo ``` 
 ``` networkpolicy.networking.k8s.io "default-deny" deleted ``` 

 ``` kubectl create ns policy-demo ``` 
Create some nginx pods in the policy-demo namespace.
 ``` kubectl create deployment --namespace=policy-demo nginx --image=nginx ``` 
Expose them through a service.
 ``` kubectl expose --namespace=policy-demo deployment nginx --port=80 ``` 
Ensure the nginx service is accessible.
 ``` kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh ``` 
From inside the access pod, attempt to reach the nginx service.
 ``` wget -q nginx -O - ``` 
You should see a response from nginx. Great! Our service is accessible. You can exit the pod now.

Now APPLY all DENY at namespace level
NExt turn on isolation in our policy-demo namespace.
```
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
  namespace: policy-demo
spec:
  podSelector:
    matchLabels: {}
EOF
```
This will prevent all access to the nginx service. We can see the effect by trying to access the service again.
 ``` kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh ``` 
This should open up a shell session inside the access pod, as shown below.
Waiting for pod policy-demo/access-xxxxxx-xxxxx to be running, status is Pending, pod ready: false
If you don't see a command prompt, try pressing enter.
/ #

Now from within the busybox access pod execute the following command to test access to the nginx service.
 ``` wget -q --timeout=5 nginx -O - ``` 
The request should time out after 5 seconds.
wget: download timed out

Allow access using n/w policy
Create a network policy access-nginx with the following contents:
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-nginx
  namespace: policy-demo
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
    - from:
      - podSelector:
          matchLabels:
            run: access
EOF
We should NOW be able to access the service from the access pod.
 ``` kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh  ``` 
 ``` wget -q --timeout=5 nginx -O -  ``` 

However, we still cannot access the service from a pod without the label run: access. We can verify this as follows.
 ``` kubectl run --namespace=policy-demo cant-access --rm -ti --image busybox /bin/sh ``` 

Now from within the busybox cant-access pod execute the following command to test access to the nginx service.Now from within the busybox cant-access pod execute the following command to test access to the nginx service.
 ``` wget -q --timeout=5 nginx -O - ``` 
The request should time out.
wget: download timed out
wget -q --timeout=5 nginx -O -
The request should time out.
wget: download timed out

You can clean up the demo by deleting the demo namespace.
 ``` kubectl delete ns policy-demo ``` 

# EXAMPLE 3

Create set up pods
============================
 ``` kubectl create ns advanced-policy-demo ``` 
 ``` kubectl create deployment --namespace=advanced-policy-demo nginx --image=nginx ``` 
 ``` kubectl expose --namespace=advanced-policy-demo deployment nginx --port=80 ``` 

Verify access - allowed all ingress and egress which is ENABLED by default if there is no default deny policy
========================================================
 ``` kubectl run --namespace=advanced-policy-demo access --rm -ti --image busybox /bin/sh ``` 
Waiting for pod advanced-policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false
If you don't see a command prompt, try pressing enter.
/ #
 ``` wget -q --timeout=5 nginx -O -  ```  
It should return the HTML of the nginx welcome page.
 ``` wget -q --timeout=5 google.com -O - ``` 
  ``` nslookup nginx ```   it means DNS at 53 UDP is also working
It should return the HTML of the google.com home page.
``` nslookup google ```  it means DNS at 53 UDP is also working

Deny all ingress traffic
============================
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: advanced-policy-demo
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Ingress
EOF


Verify access - denied all ingress and allowed all egress
========================================================
from Busybox now try to access nginx , so that its busy box -> nginx ingress trafic check
 ``` wget -q --timeout=5 nginx -O - ``` 
wget: download timed out

 ``` wget -q --timeout=5 google.com -O - ``` 
We can see that the ingress access to the nginx service is denied as ingress in given namespace is restricted while egress access to outbound internet is still allowed.

Next , Allow ingress traffic to Nginx
============================
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
  namespace: advanced-policy-demo
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
    - from:
      - podSelector:
          matchLabels: {}
EOF

Verify access - allowed nginx ingress
============================
``` wget -q --timeout=5 nginx -O - ``` 
Now nginx access is successufl
 ``` wget -q --timeout=5 google.com -O - ``` 
 Also accessible as there was no change in egress

Deny all egress traffic
============================
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: advanced-policy-demo
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
EOF

Verify access - denied all egress
============================
We can see that this is the case by switching over to our "access" pod in the namespace and attempting to nslookup nginx or wget google.com
 ``` nslookup nginx ```  Fails as egress not allowed
 ``` wget -q --timeout=5 google.com -O - ``` 
wget: bad address 'google.com'
  ``` wget -q --timeout=5 nginx -O -  ``` 
  bad address 'nginx'

Allow ONLY DNS egress traffic
============================

```kubectl label namespace kube-system name=kube-system ```

kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-access
  namespace: advanced-policy-demo
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

EOF

Verify access - allowed DNS access
============================
``` nslookup nginx ``
It should return something like the following.

Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

``` nslookup google.com ``` 
Should also WORK now

NOTE: Even though DNS egress traffic is now working, all other egress traffic from all pods in the advanced-policy-demo namespace is still blocked. Therefore the HTTP egress traffic from the wget calls will still fail.
``` wget -q --timeout=5 google.com -O - ```
wget: can't connect to remote host (172.xx.xx.xx): Connection refused
``` wget -q --timeout=5 nginx -O - ```
wget: can't connect to remote host (10.xx.xx.xx): Connection refused

Allow egress traffic to nginx
============================
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-advance-policy-ns
  namespace: advanced-policy-demo
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: nginx
EOF
Verify access - allowed egress access to nginx
============================
``` wget -q --timeout=5 nginx -O - ``` 
It should return the HTML of the nginx welcome page.
``` wget -q --timeout=5 google.com -O - ``` 
wget: download timed out,Connection refused 
Because only egress from pods with app: nginx is allowed.

Clean UP 
``` kubectl delete ns advanced-policy-demo ``` 
