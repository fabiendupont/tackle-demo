
# Tackle - Demo Scenario

***Note** - The steps described below are executed on a Fedora 35 workstation, but will likely work on any recent Linux distribution. The only prerequisites are to enable virtualization extensions in the BIOS/EFI of the machine, to install libvirt and to add our user to the libvirt group.*

The first thing to do is to install Minikube, which provides a certified Kubernetes platform with most of the vanilla features. This allows to simulate a real Kubernetes cluster with a fairly high level of feature parity.

## Installing Minikube

The installation consists in downloading the `minikube` binary into `${HOME}/.local/bin`, so that we don't need to modify the PATH variable. Other options are described in Minikube documentation.

```shell
curl -sL -o ${HOME}/.local/bin/minikube \
    https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

The binary doesn't have the execution permission, so we add it.

```shell
chmod u+x ~/.local/bin/minikube
```

By default, Minikube uses the `kvm` driver with 6,000 MB of memory. This is not enough to run all the services, so we need to increase the allocated memory. In our experience, 10 GB of memory is fine.

```shell
minikube config set memory 10240
```

Then, the following command will download a qcow2 image of Minikube and start a virtual machine from it. It will then wait for the Kubernetes API to be ready.

```shell
minikube start
```

We need to enable the `dashboard`, `ingress` and `olm` addons. The `dashboard` addon installs the dashboard service that exposes the Kubernetes objects in a user interface. The `ingress` addon allows us to create Ingress CRs to expose the Tackle UI and Tackle Hub API. The `olm` addon allows us to use an operator to deploy Tackle.

```shell
minikube addons enable dashboard
minikube addons enable ingress
minikube addons enable olm
```

The following command gives us the IP address assigned to the virtual machine created by Minikube. It will be used later, when interacting with the Tackle Hub API.

```shell
$ minikube ip
192.168.39.23
```

## Configuring kubectl

Minikube allows us to use the kubectl command with `minikube kubectl`. To make the experience more Kubernetes-like, we can set a shell alias to simply use `kubectl`. The following example shows how to do it for Bash on Fedora 35.

```shell
mkdir -p ~/.bashrc.d
```

```shell
cat << EOF > ~/.bashrc.d/minikube
alias kubectl="minikube kubectl --"
EOF
```

```shell
source ~/.bashrc
```

## Accessing the Kubernetes dashboard

We may need to access the dashboard, either simply to see what's happening under the hood, or to troubleshoot an issue. We have already enabled the `dashboard` addon in a previous command.

We can use the `kubectl proxy` command to enable that. The following command sets up the proxy to listen on any network interface (useful for remote access), on the `18080/tcp` port (easy to remember), with requests filtering disabled (less secure, but necessary).

```shell
kubectl proxy \
    --address=0.0.0.0 --port 18080 \
    --disable-filter=true
```

We can now access the dashboard through the proxy.
In the following URL, replace the IP address with your workstation IP address.

`http://192.168.0.1:18080/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/`

## Installing Tackle

We can now deploy Tackle using the Ansible based operator. It is currently not published, so we'll have to use the deploy target of the Makefile provided in the repository.

```shell
git clone https://github.com/fabiendupont/tackle-operator
```

```shell
cd tackle-operator
```

```shell
make deploy
```

This only deploys the `controller-manager` pod, we need to create a Tackle CR to trigger the deployment of the other services. An example is provided in the repository.

```shell
kubectl create -f config/samples/tackle.konveyor.io_v1alpha1_tackle.yaml
```

The deployment takes some time, mainly due to the bootstrapping of Keycloak. We can monitor Keycloak deployment with kubectl wait.

```shell
kubectl wait deployment tackle-keycloak-sso \
    -n tackle-operator --for condition=Available --timeout=5m
```

Once Keycloak is up and running, we can connect to Tackle UI at the IP address noted earlier: `http://192.168.39.23` (adjust to your IP address).

We can now proceed to perform Tackle actions via the user interface:

1. Create a stakeholder:
    - Email: cmburns@globex.io
    - Name: Charles Montgomery Burns
    - Job Function: Business Service Owner / Manager

2. Create a stakeholder group:
    - Name: Retail Business Owners
    - Members: Charles Montgomery Burns

3. Create a business service:
    - Name: Globex Retail
    - Owner: Charles Montgomery Burns

4. Import applications from the CSV file: [applications.csv](files/applications.csv)

5. Assess and Review the application named "Customers"

6. Copy the assessment and review to the applications named "Inventory" and "Orders"

We now have acquired some information about the applications. Some tags were set during the import and we have identified some risks through an assessment and a review. Let's now use some add-ons to better understand the applications.

## Accessing Tackle Hub API

The following steps will require interacting with the Tackle Hub API directly. By default, it is not exposed, so we need to create an Ingress with a /hub prefix and a rewrite rule. This is a limitation of using Minikube, but it's acceptable.

Below is the [`ingress-tackle-hub-api.yml`](files/ingress-tackle-hub-api.yml) file that defines the Ingress:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tackle-hub-api
  namespace: tackle-operator
  labels:
    app.kubernetes.io/name: tackle-hub
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: tackle
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  tls:
    - {}
  rules:
    - http:
        paths:
          - path: /hub(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: tackle-hub-api
                port:
                  number: 8080
```

We create the Ingress.

```shell
kubectl create -f files/ingress-tackle-hub-api.yml
```

The Tackle Hub API is now accessible at `http://192.168.39.23/hub/`.

## Identifying the languages of the applications

When we imported the applications, the languages were not provided as tags. We have an add-on that is based on [GitHub Linguist](https://github.com/github/linguist) and that can identify the most representative language in the application source code repository. The add-on will add the language tag to the application and store the full result in a bucket.

First, we have to declare the add-on, so that we can create a task that uses it.

Below is the [`tackle-addons-discovery-languages.yml`](files/addons/tackle-addons-discovery-languages.yml) file that defines the `discovery-languages` addon.

```yaml
---
apiVersion: tackle.konveyor.io/v1alpha1
kind: Addon
metadata:
  name: discovery-languages
  namespace: tackle-operator
spec:
  image: quay.io/fabiendupont/tackle-addons-discovery-languages:latest
```

We deploy the add-on with `kubectl`.

```shell
kubectl create -f files/addons/tackle-addons-discovery-languages.yml
```

The applications that we have imported have their source code in subfolders of a single Git repository. The add-on supports analyzing a subfolder, simply by adding the path to the task data. Below is the curl command with the JSON payload for the application named "Customers" (id: 1).

Below is the [`discovery-languages/001.customers.json`](files/tasks/discovery-languages/001.customers.json) file that is passed to the API.

```json
{
    "name": "discover-language-customers",
    "addon": "discovery-languages",
    "data": {
        "application": 1,
        "git": {
            "url": "https://github.com/konveyor/mig-demo-apps",
            "path": "/apps/e2e-demo/customers-tomcat-legacy"
        }
    }
}
```

We can create the task with `curl`.

```shell
curl http://192.168.39.23/hub/tasks -X POST -d @files/tasks/discover-languages/001..customers.json
```

We can see in the Kubernetes dashboard that a job has been created and a pod is running. Let's wait for the job to complete.

We can now see in the user interface that the Java tag has been set. But the add-on has also copied the full result of GitHub Linguist in a bucket. We can list the buckets for the application with the API.

```shell
$ curl -s http://192.168.39.23/hub/application-inventory/application/1/buckets
[
  {
    "id": 1,
    "createUser": "addon",
    "updateUser": "",
    "createTime": "2021-12-22T22:24:56.020850488Z",
    "name": "DiscoveryLanguage",
    "path": "/buckets/8cb950df-2bfb-427b-b0c4-9af1a6fc0233",
    "application": 1
  }
]
```

We can then look inside the bucket and find the list of files.

```shell
curl http://192.168.39.23/hub/buckets/1/
```

Currently, this returns an HTML page listing the files in the bucket. Other representations can be considered in the future, such as a JSON array with more metadata about the files stored, like file type, size, permissions, etc...

In the case of the language discovery add-on, there is only one file named `languages.json`. We can see its content with the following API call.

```shell
$ curl -s http://192.168.39.23/hub/buckets/1/languages.json
{
  "Dockerfile": {
    "size": 676,
    "percentage": "5.16"
  },
  "Makefile": {
    "size": 1153,
    "percentage": "8.81"
  },
  "Java": {
    "size": 11262,
    "percentage": "86.03"
  }
}
```

We can call the same add-on on all the applications, in order to know the languages used and decide what the next steps are.

```shell
for i in `ls -1 files/tasks/discovery-languages` ; do
    curl http://192.168.39.23/hub/tasks -X POST -d @files/tasks/discovery-languages/${i}
done
```

## Analyzing the cloud-readiness with Windup

We have identified that most of the applications are written in Java. It would be great to perform a static code analysis with [Windup](https://github.com/windup/windup/) to identify the technical issues that prevent the application from being cloud-ready. We have an add-on that clones the Git repository, creates a bucket and calls the `mta-cli` (downstream of Windup) with the bucket as the output folder.

First, we have to declare the add-on, so that we can create a task that uses it.

Below is the [`tackle-addons-analysis-windup.yml`](files/addons/tackle-addons-analysis-windup.yml) file that defines the `analysis-windup` addon.

```yaml
---
apiVersion: tackle.konveyor.io/v1alpha1
kind: Addon
metadata:
  name: analysis-windup
  namespace: tackle-operator
spec:
  image: quay.io/fabiendupont/tackle-addons-analysis-windup:latest
```

We deploy the add-on with `kubectl`.

```shell
kubectl create -f files/addons/tackle-addons-analysis-windup.yml
```

The applications that we have imported have their source code in subfolders of a single Git repository. The add-on supports analyzing a subfolder, simply by adding the path to the task data. Below is the curl command with the JSON payload for the application named "Customers" (id: 1).

Below is the [`analysis-windup/001.customers.json`](files/tasks/analysis-windup/001.customers.json) file that is passed to the API.

```json
{
    "name": "analyze-windup-customers",
    "addon": "analysis-windup",
    "data": {
        "application": 1,
        "git": {
            "url": "https://github.com/konveyor/mig-demo-apps",
            "path": "/apps/e2e-demo/customers-tomcat-legacy"
        },
        "windup": {
            "targets": [
                "cloud-readiness"
            ]
        }
    }
}
```

We can create the task with `curl`.

```shell
curl http://192.168.39.23/hub/tasks -X POST -d @files/tasks/analysis-windup/001.customers.json
```

Once the task has completed, the result of the analysis is available in the bucket. Because Windup outputs HTML, pointing a browser to the bucket URL displays the Windup dashboard and we can explore the report.
