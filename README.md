# skiff
a tool for provisioning [Minikube](https://github.com/kubernetes/minikube) and deploying [Helm](https://helm.sh/) charts.

The goal is to stand up a development environment with a single command, so developers can start working on the application right away without spending time configuring boilerplate stuff.  It's essentially a wrapper script for [Minikube](https://github.com/kubernetes/minikube) and [Helm](https://helm.sh/) that attempts set up a default configuration and handle errors gracefully during the process.
### TL;DR
To get started, copy `skiff` to a location in your `$PATH` (such as `/usr/local/bin`), optionally add a `.skiffrc` to the root of your project directory, and run the following command:  
```skiff launch```

All Skiff commands are idempotent, meaning they can be run multiple times without changing the state of the system beyond what is intended.

---

### How It Works
Skiff will install Minikube, Helm, and other required software.  If you are using a Mac, it assumes you are using [Homebrew](https://brew.sh/) to install software.  If you're not, you'll need to run the setup steps individually (see the "Reconfigure Options" section in the help menu).  It will also install [VirtualBox](https://www.virtualbox.org/) instead of [HyperKit](https://github.com/moby/hyperkit) in order to mount an NFS share inside Minikube.

To provide access to the application, Skiff sets up a local DNS resolver that maps *.yourdomain.test to Minikube's IP address.  If your application is web-based, Skiff will generate a self-signed TLS certificate and install it in Kubernetes along with the [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/), allowing browser access.  Other non-HTTP/HTTPS services are outside the scope of this tool and need to be set up as NodePort services in your Helm chart.

Although Minikube mounts the `/home` or `/Users` directory internally by default, anecdotal reports suggest mounting a directory via NFS has better performance, so Skiff will mount your code directory inside Minikube at `/mnt/${project_name}` by default.  From there, your Helm charts can mount any part of this volume as a [`hostPath`](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath).  Minikube mounts this volume in read-write mode, so it's up to you to specify `readOnly: true` in your Helm charts if you want to prevent containers from making unwanted changes to your local filesystem.

If your container images reside in a private Docker registry, Skiff will prompt you for your credentials and store them as a Kubernetes secret so that it can pull the images when you deploy.

Once your application is up and running, you can edit the code directly and it will be reflected immediately in the container(s) without the need for any sync process.

---

### Configuration
Skiff can be customized with a `.skiffrc` file in the root of your project containing the following parameters (none are explicitly required):

| Parameter | Description |
| --------- | ----------- |
| `project_dir` | defaults to the root of your project |
| `project_name` | defaults to the name of your project directory |
| `chart_dir` | path to your Helm charts; defaults to `./helm/${project_name}` |
| `chart_values` | path to an optional `values.yaml` file for overriding chart values |
| `nfs_source_dir` | the directory to mount within Minikube; defaults to `{project_dir}` |
| `nfs_dest_dir` | path where code directory will be mounted in Minikube, defaults to `/mnt/${project_name}` |
| `ingress_controller` | whether to install the NGINX Ingress Controller (default: `disabled`); requires `domain` to be set |
| `domain` | private domain in the form "yourdomain.test" |
| `docker_registry` | URI of a private Docker registry, if used |

---

### Command Reference
```skiff launch```  
Provision a complete environment from scratch

```skiff deploy```  
Deploy Helm chart to Minikube

`skiff delete`  
Delete a deployed application from Minikube

`skiff status`  
Show the cluster status

`skiff logs {pod-query}`  
Tail logs on one or more pods.  Skiff installs the excellent Kubernetes log utility [Stern](https://github.com/wercker/stern) as a dependency.  Stern accepts regular expressions and a wide range of options.  Rather than try to reproduce its functionality, Skiff simply passes all of its command line arguments to Stern.

`skiff ssh [application]`  
Get shell access on a container.  This is already possible using [`kubectl`](https://kubernetes.io/docs/reference/kubectl/kubectl/) but you need to know the name of the pod, which is dynamically generated.  This just saves you an extra step and some tedious copy/paste.  If the pod contains a sidecar (for example, an NGINX container proxying requests to a PHP container), this command will default to the container that matches the application name.  You can instead SSH into the other container by appending its name, i.e. `skiff ssh my-app nginx`.  If you don't specify any application, Skiff will log you into the Minikube instance by default.

`skiff anchor`  
Shut down the Minikube instance when not in use

`skiff sink`  
Completely destroy the deployment and Minikube instance

`skiff reconfigure {certificate|dns|ingress|nfs|registry}`  
Reconfigure one of the setup steps

---

### FAQ
**_"Which operating systems are supported?"_**  
At the moment, Skiff only works on macOS but it is written in an attempt to be POSIX-compliant, with the intention of supporting major Linux distributions in the future.

**_"How about a GUI?"_**  
Sure--you can always run [`minikube dashboard`](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/).

**_"It's broken!"_**  
Kubernetes is undergoing rapid development and it's not uncommon for breaking changes to appear after an update.  Please open an issue or pull request.

**_"Why is it called 'skiff?'"_**  
This is a tool for managing a small, personal Kubernetes environment for one developer.  Keeping with the nautical theme of container software such as Docker and Kubernetes, a skiff is a small boat typically for a single person or small crew.
