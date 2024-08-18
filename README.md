# Splunk Kubernetes Deployment Guide

- This guide outlines the steps to deploy a Stand AloneSplunk instance using Kubernetes resources. The deployment process is broken down into three YAML files: `splunk_standalone.yml`, `splunk_ingress.yml`, and `splunk_nodeport.yml`. Follow the steps below to deploy each resource and configure your Splunk instance.

## Prerequisites
- Ensure you have a working cluster, for the below example it was tested on `k3s`
- Ensure you have `kubectl` installed and configured to interact with your Kubernetes cluster.
- Ensure you have deployed the [Splunk Operator](https://splunk.github.io/splunk-operator/) 
    - E.g. `kubectl apply -f https://github.com/splunk/splunk-operator/releases/download/2.6.0/splunk-operator-cluster.yaml --server-side  --force-conflicts`
- Ensure the `splunk` namespace exists in your cluster.
- Ensure you have a namespace named `splunk` created
    - E.g. Run `kubectl create ns splunk`

## Step 1: Deploy Splunk Standalone Instance

 - Deploy the Splunk standalone instance using the `splunk_standalone.yml` file.

### Create File `splunk_standalone.yml`
```yml
apiVersion: enterprise.splunk.com/v4
kind: Standalone
metadata:
  name: s1
  namespace: splunk
  finalizers:
  - enterprise.splunk.com/delete-pvc
```

### Run Command:
```bash
kubectl apply -f splunk_standalone.yml
```

What It Does:
- **Standalone Resource:** This deploys a standalone Splunk instance in the splunk namespace. The finalizer ensures that persistent volume claims (PVCs) are properly cleaned up before the resource is deleted.


## Step 2: Deploy Splunk Service and Ingress

- Deploy the service and ingress resources using the splunk_ingress.yml file.

### Create File `splunk_ingress.yml`
```yml
---
apiVersion: v1
kind: Service
metadata:
  name: splunk-svc
  namespace: splunk
spec:
  selector:
    app.kubernetes.io/instance: s1  # Ensure the selector matches the pod label applied by the Standalone resource
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: ClusterIP  # Expose service within the cluster only
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: splunk-ingress
  namespace: splunk
  annotations:
    ingressClassName: "traefik"  # Ensure Traefik is used as the ingress controller
    ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: splunk.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: splunk-svc
            port:
              number: 8000
```

### Run Command:
```bash
kubectl apply -f splunk_ingress.yml
```

What It Does:
- **Service Resource:** Creates a service of type ClusterIP to expose the Splunk instance internally within the Kubernetes cluster.
- **Ingress Resource:** Creates an Ingress resource to allow external access to the Splunk service via splunk.local, using Traefik as the ingress controller.


## Step 3: Deploy Splunk NodePort Service

 - Deploy the NodePort service using the `splunk_nodeport.yml` file.

### Create File `splunk_nodeport.yml`
```yml
apiVersion: v1
kind: Service
metadata:
  name: splunk-svc
  namespace: splunk
spec:
  selector:
    app.kubernetes.io/name: s1
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
      nodePort: 30080  # Example NodePort, accessible on this port from outside
  type: NodePort
```

### Run Command:
```bash
kubectl apply -f splunk_nodeport.yml
```

What It Does:
- **NodePort Service:** Updates the existing service to expose the Splunk instance externally via NodePort 30080, allowing access to Splunk from outside the Kubernetes cluster.


## Step 4: Deploy Splunk NodePort Service

 - Ensure that the pod is labeled correctly by running the following command.

### Run Command:
```bash
kubectl label pod splunk-s1-standalone-0 -n splunk app.kubernetes.io/name=s1 --overwrite
```

What It Does:
- **Pod Label Update:** This command ensures that the Splunk pod has the correct label `(app.kubernetes.io/name=s1)`.
    - This is important for the service to correctly identify and route traffic to the pod.


## Step 5: Retrieve the Splunk Admin Password
### Run Command:
```bash
kubectl get secret splunk-splunk-secret -n splunk -o jsonpath='{.data.password}' | base64 --decode
```

## Step 5a: Update the admin password
### Run Command:
```bash
kubectl create secret generic splunk-splunk-secret --from-literal='password=<new-password>' -n splunk --dry-run=client -o yaml | kubectl apply -f -
```

## Step 6: Access Splunk Web
### Visit your Stand Alone Splunk Instance Below:
- `http://<your local node IP>>:30080`


## Additional Notes
- This guide was put together by using: https://splunk.github.io/splunk-operator/
- For how to expand upon this on other deployment methods visit: https://splunk.github.io/splunk-operator/Examples.html#configuring-splunk-enterprise-deployments
- Splunk Operator GitHub page: https://github.com/splunk/splunk-operator/tree/main
