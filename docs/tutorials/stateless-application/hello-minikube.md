---

title: Hello Minikube
redirect_from:
- "/docs/hellonode/"
- "/docs/hellonode.html"
---

{% capture overview %}

The goal of this tutorial is for you to turn a simple Hello World Node.js app
into an application running on Kubernetes. The tutorial shows you how to
take code that you have developed on your machine, turn it into a Docker
container image and then run that image on [Minikube](/docs/getting-started-guides/minikube).
Minikube provides a simple way of running Kubernetes on your local machine for free.

{% endcapture %}

{% capture objectives %}

* Run a hello world Node.js application.
* Deploy the application to Minikube.
* View application logs.
* Update the application image.


{% endcapture %}

{% capture prerequisites %}

* For OS X, you need [Homebrew](https://brew.sh) to install the `xhyve`
driver.

* [NodeJS](https://nodejs.org/en/) is required to run the sample application.

* Install Docker. On OS X, we recommend
[Docker for Mac](https://docs.docker.com/engine/installation/mac/).


{% endcapture %}

{% capture lessoncontent %}

## Create a Minikube cluster

This tutorial uses [Minikube](https://github.com/kubernetes/minikube) to
create a local cluster. This tutorial also assumes you are using
[Docker for Mac](https://docs.docker.com/engine/installation/mac/)
on OS X. If you are on a different platform like Linux, or using VirtualBox
instead of Docker for Mac, the instructions to install Minikube may be
slightly different. For general Minikube installation instructions, see
the [Minikube installation guide](/docs/getting-started-guides/minikube/).

Use `curl` to download and install the latest Minikube release:

```shell
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

Use Homebrew to install the xhyve driver and set its permissions:

```shell
brew install docker-machine-driver-xhyve
sudo chown root:wheel $(brew --prefix)/opt/docker-machine-driver-xhyve/bin/docker-machine-driver-xhyve
sudo chmod u+s $(brew --prefix)/opt/docker-machine-driver-xhyve/bin/docker-machine-driver-xhyve
```

Download the latest version of the `kubectl` command-line tool, which you can
use to interact with Kubernetes clusters:

```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```
Determine whether you can access sites like [https://cloud.google.com/container-registry/](https://cloud.google.com/container-registry/) directly without a proxy, by opening a new terminal and using
```shell
export http_proxy=""
export https_proxy=""
curl https://cloud.google.com/container-registry/ 
```

If NO proxy is required, start the Minikube cluster: 

```shell
minikube start --vm-driver=xhyve
```
If a proxy server is required, use the following method to start Minikube cluster with proxy setting:

```shell
minikube start --vm-driver=xhyve --docker-env HTTP_PROXY=http://your-http-proxy-host:your-http-proxy-port  --docker-env HTTPS_PROXY=http(s)://your-https-proxy-host:your-https-proxy-port
```

The `--vm-driver=xhyve` flag specifies that you are using Docker for Mac. The
default VM driver is VirtualBox.

Now set the Minikube context. The context is what determines which cluster
`kubectl` is interacting with. You can see all your available contexts in the
`~/.kube/config` file.

```shell
kubectl config use-context minikube
```

Verify that `kubectl` is configured to communicate with your cluster:

```shell
kubectl cluster-info
```

## Create your Node.js application

The next step is to write the application. Save this code in a folder named `hellonode`
with the filename `server.js`:

{% include code.html language="js" file="server.js" ghlink="/docs/tutorials/stateless-application/server.js" %}

Run your application:

```shell
node server.js
```

You should be able to see your "Hello World!" message at http://localhost:8080/.

Stop the running Node.js server by pressing **Ctrl-C**.

The next step is to package your application in a Docker container.

## Create a Docker container image

Create a file, also in the `hellonode` folder, named `Dockerfile`. A Dockerfile describes
the image that you want to build. You can build a Docker container image by extending an
existing image. The image in this tutorial extends an existing Node.js image.

{% include code.html language="conf" file="Dockerfile" ghlink="/docs/tutorials/stateless-application/Dockerfile" %}

This recipe for the Docker image starts from the official Node.js LTS image
found in the Docker registry, exposes port 8080, copies your `server.js` file
to the image and start the Node.js server.

Because this tutorial uses Minikube, instead of pushing your Docker image to a
registry, you can simply build the image using the same Docker host as
the Minikube VM, so that the images are automatically present. To do so, make
sure you are using the Minikube Docker daemon:

```shell
eval $(minikube docker-env)
```

**Note:** Later, when you no longer wish to use the Minikube host, you can undo
this change by running `eval $(minikube docker-env -u)`.

Build your Docker image, using the Minikube Docker daemon:

```shell
docker build -t hello-node:v1 .
```

Now the Minikube VM can run the image you built.

## Create a Deployment

A Kubernetes [*Pod*](/docs/user-guide/pods/) is a group of one or more Containers,
tied together for the purposes of administration and networking. The Pod in this
tutorial has only one Container. A Kubernetes
[*Deployment*](/docs/user-guide/deployments) checks on the health of your
Pod and restarts the Pod's Container if it terminates. Deployments are the
recommended way to manage the creation and scaling of Pods.

Use the `kubectl run` command to create a Deployment that manages a Pod. The
Pod runs a Container based on your `hello-node:v1` Docker image:

```shell
kubectl run hello-node --image=hello-node:v1 --port=8080
```

View the Deployment:


```shell
kubectl get deployments
```

Output:


```shell
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
hello-node   1         1         1            1           3m
```

View the Pod:


```shell
kubectl get pods
```

Output:


```shell
NAME                         READY     STATUS    RESTARTS   AGE
hello-node-714049816-ztzrb   1/1       Running   0          6m
```

View cluster events:

```shell
kubectl get events
```

View the `kubectl` configuration:

```shell
kubectl config view
```

For more information about `kubectl`commands, see the
[kubectl overview](/docs/user-guide/kubectl-overview/).

## Create a Service

By default, the Pod is only accessible by its internal IP address within the
Kubernetes cluster. To make the `hello-node` Container accessible from outside the
Kubernetes virtual network, you have to expose the Pod as a
Kubernetes [*Service*](/docs/user-guide/services/).

From your development machine, you can expose the Pod to the public internet
using the `kubectl expose` command:

```shell
kubectl expose deployment hello-node --type=LoadBalancer
```

View the Service you just created:

```shell
kubectl get services
```

Output:

```shell
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
hello-node   10.0.0.71    <pending>     8080/TCP   6m
kubernetes   10.0.0.1     <none>        443/TCP    14d
```

The `--type=LoadBalancer` flag indicates that you want to expose your Service
outside of the cluster. On cloud providers that support load balancers,
an external IP address would be provisioned to access the Service. On Minikube,
the `LoadBalancer` type makes the Service accessible through the `minikube service`
command.

```shell
minikube service hello-node
```

This automatically opens up a browser window using a local IP address that
serves your app and shows the "Hello World" message.

Assuming you've sent requests to your new web service using the browser or curl,
you should now be able to see some logs:

```shell
kubectl logs <POD-NAME>
```

## Update your app

Edit your `server.js` file to return a new message:

```javascript
response.end('Hello World Again!');

```

Build a new version of your image:

```shell
docker build -t hello-node:v2 .
```

Update the image of your Deployment:

```shell
kubectl set image deployment/hello-node hello-node=hello-node:v2
```

Run your app again to view the new message:

```shell
minikube service hello-node
```

## Clean up

Now you can clean up the resources you created in your cluster:

```shell
kubectl delete service hello-node
kubectl delete deployment hello-node
```

Optionally, stop Minikube:

```shell
minikube stop
```

{% endcapture %}


{% capture whatsnext %}

* Learn more about [Deployment objects](/docs/user-guide/deployments/).
* Learn more about [Deploying applications](/docs/user-guide/deploying-applications/).
* Learn more about [Service objects](/docs/user-guide/services/).

{% endcapture %}

{% include templates/tutorial.md %}
