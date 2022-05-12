### Helm

Helm is a package manager and application management tool for Kubernetes that packages multiple Kubernetes resources into a single logical deployment unit called a Chart.

### Install Helm
```
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

Check version

```
helm version --short
```

Download stable Chart respository 

```
helm repo add stable https://charts.helm.sh/stable
```

List Charts installed

```
helm search repo stable
```

Bash completion for the Helm command

```
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```

## Deploy Nginx With Helm
Update Helm’s local list of Charts

```
helm repo add stable https://charts.helm.sh/stable
helm repo update
```

## Search Repos

Lists all charts

```
helm search repo
```

Search for specific chart: helm search repo *chart name*
for example nginx:

```
helm search repo nginx
```

There are several products for each Chart so make sure you choose the correct one. For example the basic nginx it is under bitnami/nginx

## Install Nginx
Use the helm utility to install the bitnami/nginx chart and specify the name mywebserver for the Kubernetes deployment. Consult the helm install documentation or run the helm install --help command to figure out the syntax.

```
helm install mywebserver bitnami/nginx
```

Check that the install worked.

```
kubectl get all
```

more detailed check

```
kubectl describe deployment mywebserver
```

To get URL for service

```
kubectl get service mywebserver-nginx -o wide
```

Copy the value for `EXTERNAL-IP`, open a new tab in your web browser, and paste it in.

## Cleanup
To check what is running
```
helm list
```

Unistall the deployment

```
helm uninstall mywebserver
```

