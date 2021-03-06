# Kong Ingress on Oracle Container Engine for Kubernetes (OKE)

## Requirements

- An Oracle Container Engine for Kubernetes (OKE) cluster.
  - The easiest way to do this is would be to create a 'Quick Cluster' with default
   settings as described [here](https://docs.cloud.oracle.com/iaas/Content/ContEng/Tasks/contengcreatingclusterusingoke.htm#create-quick-cluster)
- A working `kubectl`  that is setup to connect to your cluster on OKE.
  - For instance, you can get your kubeconfig to access the cluster you created by using the following command
  - ```bash
    oci ce cluster create-kubeconfig --cluster-id <cluster_OCID>. --file $HOME/.kube/config  --region <region_name>
    ```

See [documentation](https://docs.cloud.oracle.com/iaas/Content/ContEng/Tasks/contengdownloadkubeconfigfile.htm) for more details

## Deploying the Kong Ingress Controller

We first need to ensure that the user has the sufficient privileges to modify the cluster.
Once the user permissions are setup, we can deploy the kubernetes objects that make up the Kong Ingress Controller.

### Update User Permissions

For most operations on Kubernetes clusters created and managed by Container Engine for Kubernetes,
Oracle Cloud Infrastructure Identity and Access Management (IAM) provides access control.
A user's permissions to access clusters comes from the groups to which they belong.
The permissions for a group are defined by policies. Policies define what actions members of a group
can perform, and in which compartments. Users can then access clusters and perform operations based
on the policies set for the groups they are members of.

For setting up the Kong Ingress Controller, we need to add cluster objects like `ClusterRole` and `ClusterRoleBinding`.
IAM controls which Kubernetes object create/delete/view operations a user can perform
on all clusters within a compartment or tenancy, and we need to enable the user performing the
setup to have the required privileges to modify the cluster.

```bash
  kubectl create clusterrolebinding cluster-admin-user --clusterrole=cluster-admin --user=<user_OCID>
```

or alternatively :

```yaml

echo -n "
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: cluster-admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: User
  name: <OCID of the user performing setup>
  namespace: kube-system" | kubectl apply -f -

```

For more information on Access Control on OKE, see our [documentation](https://docs.cloud.oracle.com/iaas/Content/ContEng/Concepts/contengaboutaccesscontrol.htm)

### Deploy Kong

The command below sets up all the required Kubernetes objects in your cluster.
 Note that the YAML file used below creates a new namespace `kong` and sets up the objects in this new namespace.

```bash

kubectl apply -f https://raw.githubusercontent.com/jeevanjoseph/kubernetes-ingress-controller/master/deploy/single/all-in-one-postgres.yaml

```

Should display:

```bash

namespace/kong created
customresourcedefinition.apiextensions.k8s.io/kongplugins.configuration.konghq.com created
customresourcedefinition.apiextensions.k8s.io/kongconsumers.configuration.konghq.com created
customresourcedefinition.apiextensions.k8s.io/kongcredentials.configuration.konghq.com created
customresourcedefinition.apiextensions.k8s.io/kongingresses.configuration.konghq.com created
service/postgres created
statefulset.apps/postgres created
serviceaccount/kong-serviceaccount created
clusterrole.rbac.authorization.k8s.io/kong-ingress-clusterrole created
role.rbac.authorization.k8s.io/kong-ingress-role created
rolebinding.rbac.authorization.k8s.io/kong-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/kong-ingress-clusterrole-nisa-binding created
service/kong-ingress-controller created
deployment.extensions/kong-ingress-controller created
service/kong-proxy created
deployment.extensions/kong created
job.batch/kong-migrations created

```

Note that it may take upto a couple of minutes for all the components to be provisioned. If you have the kube proxy running, you should be able to use the dahsboard to track the status visually.

You can now retrieve the associated IP for the Service `kong-proxy`
(or you can use  directly your static IP if you used one):

`kubectl get services -n kong` should display :

```bash

NAME                      TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)                      AGE
kong-ingress-controller   NodePort       10.96.69.10    <none>            8001:30001/TCP               15m
kong-proxy                LoadBalancer   10.96.149.59   129.146.215.232   80:30084/TCP,443:30982/TCP   15m
postgres                  ClusterIP      10.96.63.109   <none>            5432/TCP                     15m

```

We see that OKE has provisioned a loadbalancer on Oracle Cloud Infrastructure to front the traffic to the ingress controller.
In this case, its public IP is `129.146.215.232`. We can validate the ingress controller quickly by 

```bash

curl 129.146.215.232

```

Should display:

```bash

{"message":"no route and no API found with those values"}

```

### Test your deployment

- Deploy a dummy application :
  
  ```bash

  kubectl create namespace dummy && kubectl apply -f https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/manifests/dummy-application.yaml -n dummy

  ```

- Add an Ingress:

  ```yaml

  echo -n "
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: dummy
    namespace:  dummy
    annotations:
      kubernetes.io/ingress.class: "kong"
  spec:
    rules:
      - http:
          paths:
            - path: "/dummy"
              backend:
                serviceName: http-svc
                servicePort: http" | kubectl apply -f -

  ```

Now `curl -v http://129.146.215.232/dummy` should display some information, including headers showing that the response came back though Kong.

```bash
*   Trying 129.146.215.232...
* TCP_NODELAY set
* Connected to 129.146.215.232 (129.146.215.232) port 80 (#0)
> GET /dummy HTTP/1.1
> Host: 129.146.215.232
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
< Date: Fri, 15 Mar 2019 19:20:01 GMT
< Server: echoserver
< X-Kong-Upstream-Latency: 0
< X-Kong-Proxy-Latency: 1
< Via: kong/1.0.3
<


Hostname: http-svc-66ffffc458-q9594

Pod Information:
        node name:      129.146.119.120
        pod name:       http-svc-66ffffc458-q9594
        pod namespace:  dummy
        pod IP: 10.244.1.107

Server values:
        server_version=nginx: 1.13.3 - lua: 10008

Request Information:
        client_address=10.244.1.114
        method=GET
        real path=/dummy
        query=
        request_version=1.1
        request_uri=http://10.244.1.107:8080/dummy

Request Headers:
        accept=*/*
        connection=keep-alive
        host=10.244.1.107:8080
        user-agent=curl/7.54.0
        x-forwarded-for=10.244.2.0
        x-forwarded-host=129.146.215.232
        x-forwarded-port=8000
        x-forwarded-proto=http
        x-real-ip=10.244.2.0

Request Body:
        -no body in request-
```

## Bonus: Expose the Kong admin API

If you want to expose the Kong admin API,
you must configure Kong correctly via `KONG_ADMIN_LISTEN` and add an Ingress:

```yaml

echo -n "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kong-admin
  namespace:  kong
  annotations:
    kubernetes.io/ingress.class: "kong"
spec:
  rules:
    - http:
        paths:
          - path: "/"
            backend:
              serviceName: kong-ingress-controller
              servicePort: 8001" | kubectl apply -f -

```
