# Container-Scanning-Using-Anchore-Engine-in-K8-clusters

## Introduction

The Anchore Engine is an open-source project that provides a centralized service for inspection, analysis, and certification of container images.

With a deployment of Anchore Engine running in your environment, container images are downloaded and analyzed from Docker V2 compatible container registries and then evaluated to perform security, compliance, and best practices enforcement checks.

https://anchore.com/blog/anchore-engine/#:~:text=The%20Anchore%20Engine%20is%20an,Swarm%2C%20Rancher%20or%20Amazon%20ECS.


## Installation

The preferred method for deploying Anchore on Kubernetes is with Helm.

```
helm repo add anchore https://charts.anchore.io 
helm install anchore anchore/anchore-engine -n anchore
```

The above commands will install anchore engine in namespace `anchore`.

```
~$ helm ls -n anchore

NAME    NAMESPACE REVISION UPDATED                                 STATUS   CHART                 APP VERSION
anchore anchore   1        2021-09-29 05:59:31.592661398 +0000 UTC deployed anchore-engine-1.14.6 0.10.2


~$ kubectl get deployments -n anchore
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
anchore-anchore-engine-analyzer      1/1     1            1           69d
anchore-anchore-engine-api           1/1     1            1           69d
anchore-anchore-engine-catalog       1/1     1            1           69d
anchore-anchore-engine-policy        1/1     1            1           69d
anchore-anchore-engine-simplequeue   1/1     1            1           69d
anchore-postgresql                   1/1     1            1           69d
```


Also we are creating a service to expose the api.

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
    cloud.google.com/backend-config: '{"default": "{{anchore-bc"}'
    cloud.google.com/load-balancer-type: Internal
    networking.gke.io/internal-load-balancer-allow-global-access: "true"
  name: anchore-engine
spec:
  type: LoadBalancer
  ports:
  - port: 8228
    targetPort: 8228
    protocol: TCP
  selector:
    app: anchore-anchore-engine
    component: api
  sessionAffinity: None
```

```
~$ kubectl get svc -n anchore
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
anchore-engine                       LoadBalancer   10.135.14.121   10.61.0.86    8228:32677/TCP   69d
```

## Anchore CLI

The Anchore Engine exposes a REST API however the easiest way to interact with the Anchore Engine is through the Anchore CLI which can be installed using Python PiP.

To install anchore-cli client :

`pip3 install --user --upgrade anchorecli`

The Anchore CLI can be configured using command line options, environment variables or a configuration file.

Lets add environment variable (username and password) to configure anchore-cli so that it will use these credentials to connect to anchore engine

```
export ANCHORE_CLI_URL=http://<anchore_engine_svc_url>:8228/v1
export ANCHORE_CLI_USER=admin
##Anchore Engine Password can be retrievied from K8 secret##
export ANCHORE_CLI_PASS=$(kubectl get secrets/anchore-anchore-engine-admin-pass -n anchore --template={{.data.ANCHORE_ADMIN_PASSWORD}} | base64 -d)
```

Now let’s add our docker artifactory to anchore engine so that it can pull the image from the jfrog registry.

`anchore-cli registry add <docker-url> <Read User> <Read Password>`

Adding image to anchore engine for analysis

`anchore-cli image add <docker-url>/<path>:<tag>`

Once Analysis is completed (It will take couple of minutes for the analysis to complete), we can see the result here:

`anchore-cli image vuln <docker-url>/<path>:<tag> all`

For more details: GitHub — anchore/anchore-cli: Simple command-line client to the Anchore Engine service

## Conclusion

Since the world has moved to microservices and containers, it is very important to keep the security practices in place. One of the important process is continuously scanning the deployed docker images.

Anchore engine is a great tool that I am using for this purpose. It will regularly scan and update us with the vulnerabilities.
Hope this helps you all to keep your images safe !
