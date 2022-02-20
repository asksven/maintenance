# canary with nginx

## Step 1

We want to implement a canary deployment sending 50% of the traffic to version v1 and 50% to version v2 of our app.

To deploy the app:

```
cd ops/manifests
kubectl create ns maintenance
kubectl apply -f .
```

curl -s https://maintenance.asksven.io

shows alternatively:

```
<head/>
<body>
    <h1>This is v1</h1>
</body>
```

and 

```
<head/>
<body>
    <h1>This is v2</h1>
</body>
```

## Lets's add a maintenance page

```
curl -s --cookie "getmaintenance=always" https://maintenance.asksven.io
```

directs every request to the maintenance page, and

```
curl -s --cookie "getmaintenance=never" https://maintenance.asksven.io
```

directs the requests to  v1 or v2; so does

```
curl -s https://maintenance.asksven.io
```

## Let's go to maintenance mode

To deploy the maintenance mode:

```
kubectl create ns maintenance
kubectl apply -f ./force-maintenance
```

In maintenance mode we want to have the ingress `maintenance-ingress` receiving all the traffic **except** if the cookie `skipmaintenance` is set to `always`; otherwise the traffic will go to the v1 service.

1. When opened in the browser the maintenance page is shown
1. Adding a cookie `skipmaintenance` with the value `always` in the devtools will forward to v1

Go out of maintenance:


```
kubectl create ns maintenance
kubectl apply -f ./force-maintenance
```

**Note**: since we can not determine win what sequence the desird state will be reached we may face errors like this:

```
Error from server (BadRequest): error when applying patch:
{"metadata":{"annotations":{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"networking.k8s.io/v1\",\"kind\":\"Ingress\",\"metadata\":{\"annotations\":{\"ingress.kubernetes.io/browser-xss-filter\":\"true\",\"ingress.kubernetes.io/content-type-nosniff\":\"true\",\"ingress.kubernetes.io/custom-response-headers\":\"Server: Incognito    \\n\",\"ingress.kubernetes.io/force-hsts\":\"true\",\"ingress.kubernetes.io/frame-deny\":\"true\",\"ingress.kubernetes.io/hsts-include-subdomains\":\"true\",\"ingress.kubernetes.io/hsts-max-age\":\"15724800\",\"ingress.kubernetes.io/referrer-policy\":\"no-referrer\"},\"name\":\"app-ingress\",\"namespace\":\"maintenance\"},\"spec\":{\"ingressClassName\":\"nginx-asksven-io\",\"rules\":[{\"host\":\"maintenance.asksven.io\",\"http\":{\"paths\":[{\"backend\":{\"service\":{\"name\":\"app-service\",\"port\":{\"number\":80}}},\"path\":\"/\",\"pathType\":\"Prefix\"}]}}]}}\n","nginx.ingress.kubernetes.io/canary":null,"nginx.ingress.kubernetes.io/canary-by-cookie":null}}}
to:
Resource: "networking.k8s.io/v1, Resource=ingresses", GroupVersionKind: "networking.k8s.io/v1, Kind=Ingress"
Name: "app-ingress", Namespace: "maintenance"
for: "app-ingress.yaml": admission webhook "validate.nginx.ingress.kubernetes.io" denied the request: host "maintenance.asksven.io" and path "/" is already defined in ingress maintenance/maintenance-ingress
```

telling us there can not be two ingress objects for a given hostname where not at least one has the annotation `nginx.ingress.kubernetes.io/canary: "true"`. This could for example happen if (when going out of maintenance) `app-ingress` is created before `maintenance-ingress`: the latter previously did not have the annotation, leading to having two ingresses without the annotation. To mitigate that when going out of maintenance run:

```
kubectl -n maintenance apply -f ./maintenance-ingress.yaml
kubectl -n maintenance apply -f .
```



