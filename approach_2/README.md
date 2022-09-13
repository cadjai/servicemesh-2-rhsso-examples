# Approach 2

This is a modified version of the original approach 2 where instead of using RH SSO as the oauth2-proxy we are using the OpenShift oauth server as an oauth-proxy. 
The configuration steps remain the same, except that there is no need to install and configure RHSSO or to have gone through the other approaches. Also there is no need to update the oauth-proxy sidecar to reflect your cluster configuration since there is no reference to route in there. However, you might still want to update the manifest to reflect your configuration like adjusting the openshift-sar, permission or others valid oauth-proxy arguments you may want to use. 
In the commands below the namespace used is istio-system. Feel free to update it to match the namespace you are using for your Service Mesh Control Plane.

## Prerequisites
To configure the OpenShift oauth-proxy, a few things need to also be configured. 

### Configure user and permissions for the applicable identity provider used in your cluster. 
1. Get the list of configured Identity Providers for you cluster 
```
$ oc get oauth cluster -o jsonpath='{.spec.identityProviders}{"\n"}' | jq .
```
2. Chose the IDP to use for this. We will assume that you have htpasswd configured in the next step. Feel free to adapt it to the configuration you have in your cluster.

3. Retrieve and update the current htpasswd from the existing secret so that it includes our test user localuser
   1. Find the current htpasswd secret
      ```
      $ oc get secret -n openshift-config --no-headers | grep htpass | awk '{print $1}'
      ```
   2. Extract the current htpasswd file to be updated later
      ```
      $ oc extract secret/$(oc get secret -n openshift-config --no-headers | grep htpass | awk '{print $1}') -n openshift-config --to=/tmp/
      ```
   3. Add the localuser to the retrieved htpasswd file
      ```
      $ htpasswd -bB /tmp/htpasswd localuser localuser
      ```
   4. Update the htpasswd secret so that it includes our localuser
      ```
      $ oc create secret generic $(oc get secret -n openshift-config --no-headers | grep htpass | awk '{print $1}') \
          --from-file=htpasswd=/tmp/htpasswd --dry-run=client -o yaml -n openshift-config | oc replace -f -
      ```
   
4. Grant the localuser permission to access the sample app namespace (it is assumed that the namespace was already created and the app deployed)
```
$ oc adm policy add-role-to-user admin localuser -n bookinfo
```
It was assumed that the namespace where the sample app was deployed is bookinfo. If not update the above command before running it


### Create a random cookie secret to be used by the oauth-proxie sidecar in the service mesh control plane namespace. 
```
$ oc -n istio-system create secret generic bookinfo-cookie-secret --from-literal=session-secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c32)
```
Again above it was assumed that the SMCP is deployed into istio-system. If not update the above command before running it.
 

### Create a service account to be used by the oauth-proxie sidecar in the service mesh control plane namespace in case you don't want to use the default serviceaccount used by the ingress gateway (istio-ingressgateway-service-account). 
```
$ oc -n istio-system create serviceaccount bookinfo
```
Again above it was assumed that the SMCP is deployed into istio-system. If not update the above command before running it.


### Annotate the service account created above (or the default serviceaccount used by the ingress gateway istio-ingressgateway-service-account) with a redirect URI to match the ingress gatway route to be used by the oauth-proxie sidecar in the service mesh control plane namespace. 
```
$ oc -n istio-system annotate serviceaccount bookinfo serviceaccounts.openshift.io/oauth-redirecturi.first="https://$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}'")
```

### Grant the istio-gateway-service-account auth-delegator permissions in the service mesh control plane namespace if you will be using Openshift Subject Access Review (SAR) within the oauth-proxy sidecar. 
```
$ oc adm policy add-cluster-role-to-user system:auth-delegator system:serviceaccount:istio-system:istio-ingressgateway-service-account 
```


## Check that the sample app is deployed and that you can access it using curl 
```
$ export GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')
$ curl -I http://$GATEWAY_URL/productpage
HTTP/1.1 200 OK
$ curl -s http://$GATEWAY_URL/productpage | grep -o "<title>.*</title>"
```

## Inject the oauth-proxy into the ingress gateway and make appropriate updates 

### Inject oauth-proxy container inside the Istio Ingress Gateway pod
First, make any appropriate or necessary changes to the oauth-proxy sidecar manifest in `01_patch-istio-ingressgateway-deploy.yaml` .

Then, patch the Istio Ingress Gateway deployment:
```
$ oc patch deploy istio-ingressgateway -n istio-system --patch "$(cat 01_patch-istio-ingressgateway-deploy.yaml)"
```

### Check the istio-ingressgateway pod has now 2 containers:
```
$ oc get pod -n istio-system
[...]
istio-ingressgateway-64b876ff55-98nzl   2/2     Running   0          30s
[...]
```

### Redirect Istio Ingress Gateway route to the oauth-proxy container
In order to enforce authentication, the default ingress gateway route must now target the oauth-proxy container instead of the istio-proxy container (which is the original container of the Istio Ingress Gateway pod). Once authenticated, the oauth-proxy container will forward the request locally to the istio-proxy container since they are in the same pod.

### Add the oauth-proxy port to the Istio Ingress Gateway service
```
$ oc patch svc istio-ingressgateway -n istio-system --patch "$(cat 02_patch-istio-ingressgateway-svc.yaml)" 
service/istio-ingressgateway patched
```

### Set edge TLS termination for the Istio Ingress Gateway route
```
$ oc patch route istio-ingressgateway -n istio-system --patch "$(cat 03_patch-istio-ingressgateway-route-edge.yaml)" 
route.route.openshift.io/istio-ingressgateway patched
```

### Make the Istio Ingress Gateway route target the oauth2-proxy container
```
$ oc patch route istio-ingressgateway -n istio-system --patch "$(cat 04_patch-istio-ingressgateway-route-oauth.yaml)" 
route.route.openshift.io/istio-ingressgateway patched
```

### Test the OpenShift Oauth-proxy redirection workflow to access bookinfo
In a browser, in a private navigation mode, open `https://$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')/productpage` .
You are redirected to our OpenShift Login page, and if you authenticate using `localuser:localuser` you are then successfully redirected to the `bookinfo` application.
