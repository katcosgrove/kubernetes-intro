# Intro to Kubernetes
### The Speedrun

**Note:** This repo is intended to serve as a companion to a talk I gave. For now, that means I will skip in-depth explanations for some things. At some point in the near future, I will update this to be useful as a standalone resource with a glossary and this warning will be gone. In the /app directory, you will find a small, containerized Flask app. There is no need to build this yourself and put it on a container registry somewhere, but you can if you want. All of the config files you will need are in the /kubernetes directory.


### Tools

**[Docker](https://www.docker.com/products/docker-desktop)**

Minikube requires a container manager. I recommend you go with Docker here, but it also supports Podman, Virtualbox, VMWare, Hyperkit, Hyper-V, KVM, and Parallels.

**[Minikube](https://minikube.sigs.k8s.io/docs/start/)**

Minikube is a local Kubernetes environment. You won’t see this tool used in production environments much (if at all), but it’s great for learning, because it’s always free. 

**[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)**

We also need kubectl. Some people will pronounce this `koob-cuttle` or `cube-control`. This is what allows us to interact with Kubernetes.

**This repository**

Fork and clone this repository. I have provided you with all of the configuration files you'll need to try this yourself.


### Deployment

Confirm that minikube is installed with `minikube version`, then start your cluster with `minikube start`. If you run `kubectl get nodes`, you should see one node called minikube with a status of Ready. Let’s deploy something!

There are a few ways to do this. We can use kubectl to do it right here in the terminal by pointing it at config files written in a format called YAML, or we can use a tool called Helm, which does a lot more for us but is kind of beyond the scope of this repo. For now, we’ll use YAML, and let kubectl handle it for us.

Kubernetes is useful because it allows you to scale your application without quite as much work. To demonstrate that, we’ll deploy a small containerized Flask app with replicas.

`kubernetes/deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deployment
  labels:
    app: hello-kubernetes
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-kubernetes
  template:
    metadata:
      labels:
        app: hello-kubernetes
    spec:
      containers:
      - name: flask
        image: katdemo.jfrog.io/docker/kubernetes-demo:latest
        ports:
        - containerPort: 5000
```

This YAML describes what I want to happen: the name of the pod and application, the number of replicas I want, the image it should run, and the port I need. The Flask application itself is targeting port 5000, so the deployment spec does, too. The image can be hosted on whatever registry you like; mine’s on a JFrog Artifactory instance I am using as a container registry. I’ve allowed anonymous pulls, so you can use the same image. 

From the /kubernetes folder in this repository, run `kubectl apply -f deployment.yaml` to spin it up.

If you run `kubectl get pods` now, you’ll see the Deployment created two pods for our application. To see the deployment, you can run `kubectl get deployments`.

What if you decide you need more replicas, though? Go back to your deployment configuration, bump the value of `replicas` to 5, and apply it again with `kubectl apply -f deployment.yaml`. Run `kubectl get pods` and you’ll see three more pods coming online!


### Exposing the Application

We probably want to actually SEE our app though. The quick and dirty way to do that is port forwarding within the cluster, like this:

`kubectl port-forward deployment/demo-deployment 5000:5000`

Here, we're using kubectl to forward the demo-deployment's port 5000 to our local port 5000. If you go to `localhost:5000` in a browser, you'll see your application.

That’s not very convenient though, and it doesn’t offer much in the way of customization or control. We need to add an ingress controller. I’m going to use NGINX-ingress, because it’s popular and minikube already has an add-on for it, but there are a bunch of other options out there.

Install it by running `minikube addons enable ingress`.

To confirm that the pods are up, run `kubectl get pods -n kube-system` and look for pods that start with ingress-nginx. Note that other Kubernetes environments will put these in the `ingress-nginx` namespace, instead. Now let’s add a CluterIP service.

`service.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 5000
  selector:
    app: demo-deployment
  type: ClusterIP
```

Like always, we have some metadata like a name, we tell it which ports to target (remember our Flask app is targeting port 5000), and we give the name of the deployment we want it to look at.

Apply it with `kubectl apply -f service.yaml`.

The last thing we need to do is define the ingress resource. It’s the final piece that allows our application to be visible outside of the cluster.

`ingress.yaml`
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: demo-service
          servicePort: 8080
```

Just like with all of our other config files, we're giving it some metadata, and telling it what to target. In this case it's the service we defined already, and the service's port.

As before, apply it with `kubectl apply -f service.yaml -n demo`

Now if you run `kubectl get ingress`, you’ll get an IP address. Go there in a browser, and we can see our app!




