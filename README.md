## How to secure OpenShift routes exposed through an Azure Front-Door

When you are using an Azure Front Door with public IP address-based origins, you should ensure that traffic flows through your Front Door instance.
Microsoft Azure documentation instructs to use two tecniques in order to make sure that the traffic origns from the proper Front Door:

- [IP Address filtering](https://learn.microsoft.com/en-us/azure/frontdoor/origin-security?tabs=app-service-functions&pivots=front-door-standard-premium#ip-address-filtering), basically is to use the `AzureFrontDoor.Backend` service tag to coinfigure the network security group rules
- [Front Door Identifier](https://learn.microsoft.com/en-us/azure/frontdoor/origin-security?tabs=app-service-functions&pivots=front-door-standard-premium#front-door-identifier) When Front Door makes a request to your origin, it adds the `X-Azure-FDID` request header. Your origin should inspect the header on incoming requests, and reject requests where the value doesn't match your Front Door profile's identifier.

When an Azure Front Door with public IP address-based origins is used, Microsoft suggests to use **both** mentioned methods above.

The IP Address filtering can be performed at the cloud infrastructure level, as documented by Microsot.
The Front Door Identifier must be verified at the application level checking the HTTP header `X-Azure-FDID`.

At the moment of writing Red Hat OpenShift does not provide any feature to check the Front Door Identifier, thus the check should be delegated to the application itself or an API Gateway in front of the cluster.

There is an RFE (ID RFE-3490) to implement the Front Door Identifier check directly into the ingress router:
If a route is annotated with `haproxy.router.openshift.io/azure-front-door-id` only the requests with the corresponding `X-Azure-FDID` header are allowed.
At the moment the RFE is targetting the OCP version 4.14.

Until the mentioned RFE is not implemented the only way to validate the Front Door Identifier at the OpenShift Ingress level is to deploy a customized router.
The custom router will use a modified HAProxy configuration template that implements the feature.
The customization applies an HAProxy ACL 

**NOTE:** Before deploying a custom router, please make sure to read and understand the Disclaimer section of this document.

### Deploying a custom router

In order to deploy a customized router there are two options:

#### Replacing the default ingress router

This approach basically replaces the default ingress router with a customized router instance.
A prerequisite is to disable the Ingress Router Operator in order to apply the customizations without being reconciliated by the controller.
Once the Ingress Router Operator is scaled to zero and the default ingress router deployment is un managed, it is possible to apply all the customizations.

If the `LoadBalancer` service exposing the router is recreated with a different IP address when the Ingress Router Operator is disabled, the DNS entry for the `*.apps.<cluster>.<domain>` is not automatically updated.

#### Adding a router shard with the customizations

This different approach does not requires to disable the Ingress Router Operator because the customized router is installed through a custom `Deployment` as a router shard.
Using a router shard requires an additional application FQDN (the so colled ingress wildcard domain).
The DNS entry for the FQDN needs to be managed manually or via the External DNS operator.

#### Comparision

| | Default Ingress replacement | Rotuer Sharding |
| --- | --- | --- |
| Needs to disable an operator | yes | no |
| Requires an additional URL | yes | no |
| DNS management | suggested | needed |
| Impacts on infrastructural workloads | yes | no |

## Deploy the test application:

Before starting deploying a customized router we will need an application to be used as a test.

1. Create the app:

```
$ oc new-app https://github.com/sclorg/cakephp-ex
```

2. Expose the service:

Depending if you are using a router shard you may need to set the proper route hostname

```
$ ROUTE_HOSTNAME=cachephp-ex.shard.ocp.example.com
$ oc expose svc cakephp-ex --hostname ${ROUTE_HOSTNAME}
```

3. Annotate the route in order to be secured: 

```
$ oc annotate route cakephp-ex haproxy.router.openshift.io/azure-front-door-id=1234
```

4. Optional: in case you are using a shard, you will need to annotate the route in order to "land" on the proper router instance, `type=shard` is just an example here, the label must match the `routeSelector` of the shard:

```
$ oc label route cakephp-ex -n cakephp-ex type=sharded
```

### Tesing the application

If the `haproxy.router.openshift.io/azure-front-door-id` annotation is present and you are not sendig the header `X-Azure-FDID` with a proper value you must get a `403 Forbidden`.

The following curl command should work with the annotation in the example above.

```
curl -H 'X-Azure-FDID: 1234' http://${ROUTE_HOSTNAME}
```

## Creating an unmanaged router shard

### Segregate the default router

**ACTION GOAL:** run the default router pods on a predefinite set of worker nodes. Is not needed if you are willing to execute the default router and the shard on the same set of nodes

In the following steps we will segregate the default router to be hosted only on nodes with the `router-sharded: yes`.

Add the `nodePlacement` to the default IngressController:

```
oc edit IngressController default -n openshift-ingress-operator

[...]
spec:
  nodePlacement:
    nodeSelector:
      matchLabels:
        router-sharded: "no"
        node-role.kubernetes.io/worker: ""
[...]
```

### Excluding the sharded routes from the default router

**ACTION GOAL:** exclude from the default router all the routes with a specific label

We also need to skip the routes with `type: sharded` label from the default router.

Skip the routes with `type: sharded` from the default router, in this way the default router will ignore them:

```
oc patch \
  -n openshift-ingress-operator \
  IngressController/default \
  --type='merge' \
  -p '{"spec":{"routeSelector":{"matchExpressions":[{"key":"type","operator":"NotIn","values":["sharded"]}]}}}'
```

### Create a shard from the Ingress Operator

**ACTION GOAL:** let the Ingress Operator to create a shard (a new `Deployment` of a router). This can be useful to have all the resources in order to manually create a shard.

This section documents how to create a shard using the Ingress Operator, deployng an additional router on the cluster.

**NOTE:** the deployed router cannot be customized because is managed by the Ingress operator, every change will be reconciliated.
This is useful just to have all the needed resources to use as a skeleton for the manual deployment.

If we need to segregate the routers (default and the shard) on different sets of nodes, is possible to define a `nodePlacement` section.
With the `routeSelector` field, the shard will take care *only* of the routes with label `type: sharded`:

```
$ export DOMAIN_NAME=shard.ocp.example.com

$ oc create -f - <<EOF
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: sharded
  namespace: openshift-ingress-operator
spec:
  domain: ${DOMAIN_NAME}
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/worker: ""
        router-sharded: "yes"
  routeSelector:
    matchLabels:
      type: sharded
status: {}
EOF
```

You can now backup all the objects created by the Ingress Operator: `oc adm inspect ns/openshift-ingress --dest-dir=./openshift-ingress.backup`

Before creating a manual shard you will need to remove the managed shard: `oc delete IngressController -n openshift-ingress-operator sharded`

### How to manually create a shard

**ACTION GOAL:** install a `Deployment` and all the needed resources to execute an Ingress Router. This router is not managed by the Ingress Controller, thus can be customized.

In order to be able to customize the router configuration of a shard, the router must not be managed by the Ingress Controller, otherwise every change will be reconciliated by the operator.
For that reason we have to deploy an "unmanaged" router. This repo contains an Helm chart with all the needed resources.
The Helm templates are created out of shard from a 4.9.11 OCP cluster, deploying this Helm chart on a different version requires a double-check of the resources.

1. Clone this repo `git clone https://github.com/pbertera/OCP-Securing-Azure-FD.git && cd OCP-Securing-Azure-FD`
2. Configure the `helm/values.yaml`:
    - `ingressFQDN` is the wildcard application FQDN for the shard
    - `tlsCert` and `tlsKey` are the certificate and the key used by the router (must match the `ingressFQDN`)
    - `routeLabels` defines the route selector that this shard will use
    - `image` is the router image, you can get it from the default router: `oc get deploy -n openshift-ingress router-default -o jsonpath="{.spec.template.spec.containers[0].image}"`
    - `useInternalLB` if an Internal Load Balancer should be instantiate or not, accepted values are `aws`, `azure, `gcp`.
    - `service` section to custimize the  external service used to expose the router
       - `annotations` annotations to add to the service
       - `type` the  service type
       - `allocateLoadBalancerNodePorts` if the service `spec.allocateLoadBalancerNodePorts` sould be set, to be used only with service type `LoadBalancer`
       - `httpNodePort` and `httpsNodePort` the node ports to set if service type is `NodePort`
    - `useHostNetwork` if the router pod should be exposed with `hostNetwork`
    - `httpRouterPort`, `httpsRouterPort` `metricsRouterPort` ports to use on the router pod
    - `replicas` configure the number of router pods
    - `customHAProxyTemplate` section for a custom HAProxy template file
        - `useCustomTemplate` if a custom template should be used (true of false)
        - `file` the file to use as custom template, you can obtain the default one with `oc rsh -n openshift-ingress deploy/${ROUTER_NAME} cat haproxy-config.template > haproxy-config.template`
3. Install the chart: `helm install -n openshift-ingress sharded ./helm/`
4. Verify the pods are running: `oc get pods,svc,deploy -n openshift-ingress`

The helm chart deploys a `LoadBalancer` service to expose the router.
We need to create a DNS A record for the wildcard application FQDN of the shard pointg to the the external IP of the load balancer (see the next sections for that).

The chart will deploy a router into the `openshift-ingress` namespace which is not handled by the Ingress Operator.
If the `customHAProxyTemplate.useCustomTemplate` is not set the deployed router will not use a custom template.

## Default router customization

As already highlited in order to customize the default router instance the router deplyment must not be managed by the Ingress Operator, thus as a first step we have to make the router unmanaged.

### Making the default router unmanaged

**ACTION GOAL:** Instruct the Cluster Version operator to don't reconcile the Ingress Operator deployment, so we can scale to zero (disable) the Ingress Operator.

If you want to customize the default ingress router you have to make it unmanaged, thus all the changes will be not reconciliated by the Ingress Operator.

1. Create an override to make the ingress operator unmanaged

```
cat <<EOF >version-patch.yaml
- op: add
  path: /spec/overrides
  value:
  - kind: Deployment
    group: apps
    name: ingress-operator
    namespace: openshift-ingress-operator
    unmanaged: true
EOF
```

2. Apply the patch:

```
oc patch clusterversion version --type json -p "$(cat version-patch.yaml)"
```

3. Scale down the Ingress operator:

```
oc scale deploy -n openshift-ingress-operator ingress-operator --replicas 0
```

### Router customization in order to secure requests coming from an Azure Front Door

**ACTION GOAL:** apply the HAProxy template customizations in order to secure Fron Door requests as per Microsoft instructions.

1. Get the deployments names:

```
oc get deploy -n openshift-ingress
```

Take the appropriate deployment, depending on which router you want to customize: the default or an unmanaged shard.

2. Get the default template:

```
oc rsh -n openshift-ingress deploy/${ROUTER_NAME} cat haproxy-config.template > haproxy-config.template
```

2. Apply the patch:

```
--- haproxy-config.template	2023-02-17 10:16:36.396787723 +0100
+++ haproxy-config.template.fd	2023-02-17 10:19:03.618627532 +0100
@@ -509,6 +509,10 @@
         {{- with $value := clipHAProxyTimeoutValue (firstMatch $timeSpecPattern (index $cfg.Annotations "haproxy.router.openshift.io/timeout-tunnel")) }}
   timeout tunnel  {{ $value }}
         {{- end }}
+        {{- with $azureFrontDoorAnnotation := index $cfg.Annotations "haproxy.router.openshift.io/azure-front-door-id" }}
+  acl azurefrontdoor req.hdr(X-Azure-FDID) -m str {{ $azureFrontDoorAnnotation }}
+  http-request deny unless azurefrontdoor
+        {{- end }}
 
         {{- if isTrue (index $cfg.Annotations "haproxy.router.openshift.io/rate-limit-connections") }}
   stick-table type ip size 100k expire 30s store conn_cur,conn_rate(3s),http_req_rate(10s)
```

3. Create the configmap with the template:

```
oc create configmap router-template --from-file=haproxy-config.template -n openshift-ingress
```

4. Mount the new template:

```
oc -n openshift-ingress set volume deploy/${ROUTER_NAME} --add --overwrite --name=template-volume --mount-path=/var/lib/haproxy/conf/custom  --source='{"configMap": { "name": "router-template"}}'
```

5. Set the `TEMPLATE_FILE` varialbe to use the new template

```
oc -n openshift-ingress set env deploy/${ROUTER_NAME} TEMPLATE_FILE=/var/lib/haproxy/conf/custom/haproxy-config.template
```

## DNS Management

If you replace the default ingress router and the LoadBalancer service exposing the router is not deleted/re-created a DNS record resolving the widcard domain should be already availabe.

In case you are deploying the custom router as a shard you will need to create the DNS record for the shard wildcard domain.
When the LoadBalancer service exposing the router is deleted and re-created the DNS record must be updated with the proper public load balancer IP address.

A solution to automate the management of the wildcard DNS record is to use the [External DNS operator](https://docs.openshift.com/container-platform/4.10/networking/external_dns_operator/understanding-external-dns-operator.html).

**NOTE:** Azure DNS zone management with External DNS Operator is Technology Preview on OpenShift 4.10, and Generally Available in 4.11.

1. Create the External DNS namespace:

```
$ oc create ns external-dns
```

2. Apply the needed RBAC:

```
$ oc apply -f https://raw.githubusercontent.com/openshift/external-dns-operator/release-0.1/config/rbac/extra-roles.yaml
```

3. Install the ExternalDNS operator:

```
$ oc create ns external-dns-operator

$ cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: external-dns-operator
  namespace: external-dns-operator
spec:
  targetNamespaces:
  - external-dns-operator
EOF

$ cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: external-dns-operator
  namespace: external-dns-operator
spec:
  channel: alpha
  installPlanApproval: Automatic
  name: external-dns-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

4. Fetch the values from azure-credentials secret present in kube-system namespace.

```
$ CLIENT_ID=$(oc get secrets azure-credentials  -n kube-system  --template={{.data.azure_client_id}} | base64 -d)
$ CLIENT_SECRET=$(oc get secrets azure-credentials  -n kube-system  --template={{.data.azure_client_secret}} | base64 -d)
$ RESOURCE_GROUP=$(oc get secrets azure-credentials  -n kube-system  --template={{.data.azure_resourcegroup}} | base64 -d)
$ SUBSCRIPTION_ID=$(oc get secrets azure-credentials  -n kube-system  --template={{.data.azure_subscription_id}} | base64 -d)
$ TENANT_ID=$(oc get secrets azure-credentials  -n kube-system  --template={{.data.azure_tenant_id}} | base64 -d)
```

5. Login with the `az` CLI

```
$ az login --service-principal -u "${CLIENT_ID}" -p "${CLIENT_SECRET}" --tenant "${TENANT_ID}"
```

6. Get the list of the DNS zones

```
$ az network dns zone list --resource-group $RESOURCE_GROUP
```

7. If the DNS zone for the shard is not present, create it:

```
$ az network dns zone create --resource-group  $RESOURCE_GROUP --name ${INGRESS_FQDN}
```

7. Create the `ExternalDNS` resource (replace the `spec.zone` field with the proper zone ID from the previous command)

```
$ cat << EOF | oc apply -f -
apiVersion: externaldns.olm.openshift.io/v1alpha1
kind: ExternalDNS
metadata:
  name: shard-wildcard-azure
spec:
  zones:
  - "/subscriptions/XXXX/resourceGroups/YYYY-rg/providers/Microsoft.Network/dnszones/sharded-apps.ocp.example.com"
  provider:
    type: Azure
  domains:
  - name: azure.opentlc.com
    filterType: Include
    matchType: Exact
  source:
    type: Service 
    service:
      serviceType:
        - LoadBalancer
    labelFilter: 
      matchLabels:
        external-dns.sharded-widcard/publish: "yes"
    hostnameAnnotation: "Allow"
    fqdnTemplate:
    - '*.sharded-apps.ocp.example.com'
EOF
```

8. Apply the label to service in order to create the DNS records:

```
$ oc label svc -n openshift-ingress router-sharded-custom-oc-router external-dns.sharded-widcard/publish=yes
```
