
# Helm Chart for Check Point CloudGuard AppSec <br/>(with added support for helm chart-based deployment in AWS EKS using NLB)

## 2021-04-07 Purpose of this fork by ByteSkater
This repo is a fork from https://github.com/CheckPointSW/Infinity-Next.git .
There was a limitation in the existing cpappsec-0.1.2.tgz helm chart (see /deployments, sources: /contrib/helmIngressController) that didn't allow successful Check Point AppSec deployment in AWS EKS environments, I added two lines in /contrib/helmIngressController/templates/ingress.yaml that now allow also successful deployments of the helm chart in AWS EKS, see following details.
I left the rest of the repo untouched.

Here's what to do step-by-step *in addition* to the official documentation further below:

* Manually create a few Elastic IP (EIP) addresses in your AWS account to be used with the EKS ELB, which will be deployed as part of the K8S service definition in the helm chart (you need to create one EIP per ELB subnet (for the AZ redundancy), typically this would be 3 EIPs) 
* Create a public DNS A record (e.g. in Route53) referencing all of those previously created public IPs (EIPs) with a DNS name of your choice
(alternatively you can probably also work with entries in your local HOSTS file, if you consider it sufficient to have connectivity from your local machine only, but the DNS-based approach is more realistic and comfortable)
* In the /contrib/helmIngressController/templates/service.yaml the Load-Balancer type was adjusted from Classic Load Balancer (default in AWS for EKS services of type “loadbalancer”) to Network Loadbalancer (NLB) to get support for EIP assignments during LB creation. This fixes the issue that the ELBs (=CLBs) DNS name and corresponding IP addresses per AZ in AWS are usually only available AFTER the helm chart was deployed.
Additional benefit: These public EIPs also will survive deletion and reapplication of the Helm chart and you do not need to readjust your Host A entry in DNS again after redeployments which comes in handy e.g. when used in demos or lab environments. 
To implement the change in the service.yaml (as part of the helm chart) the following two lines were added with additional annotations to set the ELB type to NLB and to reference the EIPallocs.<br/>**Important: You need to replace eipalloc-00000, eipalloc-11111, eipalloc-22222 below with your own, previously created 3 eipalloc values:**
```
service.beta.kubernetes.io/aws-load-balancer-type: nlb
service.beta.kubernetes.io/aws-load-balancer-eip-allocations: eipalloc-00000,eipalloc-11111,eipalloc-22222
```
* After you adjusted the line above you need to pack the helm chart as a .tgz file (you can use “tar –cvzf”) so that you can invoke it in the next step.
* Apply your customized helm chart by handing over the previously created DNS name in the following parameter: "--set appURL= ...." (and use it of course in the AppSec WebUI configuration)

## Overview
Check Point CloudGuard AppSec delivers access control and advanced threat prevention including web and api protection for mission-critical assets.  Check Point CloudGuard AppSec delivers advanced, multi-layered threat prevention to protect customer assets in Kubernetes clusters from web attacks and sophisticated threats based on Contextual AI.

Helm charts provide the ability to deploy a collection of kubernetes services and containers with a single command. This helm chart deploys an Nginx-based (1.19) ingress controller integrated with the Check Point container images that include and Nginx Reverse Proxy container integrated with the Check Point CloudGuard AppSec nano agent container. It is designed to run in front of your existing Kubernetes Application. If you want to integrate the Check Point CloudGuard AppSec nano agent with an ingress controller other than nginx, follow the instructions in the CloudGuard AppSec installation guide. Another option would be to download the helm chart and modify the parameters to match your Kubernetes/Application environment.
## Architecture
The following table lists the configurable parameters of this chart and their default values.

| Parameter                                                  | Description                                                     | Default                                          |
| ---------------------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------ |
| `nanoToken`                                           | Check Point AppSec nanoToken from the CloudGuard Portal(required)                             | `034f3d-96093mf-3k43li... `                                          |
| `appURL`                                           | URL of the application (must resolve to cluster IP address after deployment,required)     | `myapp.mycompany.com`                                          |
| `mysvcname`                                           | K8s service name of your application(required)     | `myapp`                         |
| `mysvcport`                                           | K8s listening port of your service(required)     | `8080`                         |
| `cpappsecnginxingress.properties.imageRepo`                                             | Dockerhub location of the nginx image integrated with Check Point AppSec                     | `checkpoint/infinity-next-nginx`                                              |
| `cpappsecnginxingress.properties.imageTag`                                             | Image Version to use                    | `0.1.148370`                                              |
| `cpappsecnanoagent.properties.imageRepo`                                              | Dockerhub location of the Check Point nano agent image              | `checkpoint/infinity-next-nano-agent`                                           |
| `cpappsecnanoagent.properties.imageTag`                                              | Version to use              | `0.1.148370`                                           |
| `TLS_CERTIFICATE_CRT`                                           | Default TLS Certificate               | `Certificate string`                         |
| `TLS_CERTIFICATE_KEY`                                           | Default TLS Certificate Key               | `Certificate Key string`                         | 

## Prerequisites
*   Properly configured access to a K8s cluster (helm and kubectl working)
*   Helm 3.x installed
*   Access to a repository that contains the Check Point Infinity-Next-Nginx controller and Infinity-Next-Nano-Agent images
*   An account in portal.checkpoint.com with access to CloudGuard AppSec

## Description of Helm Chart components
*   _Chart.yaml_ \- the basic definition of the helm chart being created. Includes helm chart type, version number, and application version number 
*   _values.yaml_ \- the default application values (variables) to be applied when installing the helm chart. In this case, the CP AppSec nano agent token ID, the image repository locations, the type of ingress service being used and the ports, and specific application specifications can be defined in this file. These values can be manually overridden when launching the helm chart from the command line as shown in the example below.
*   _templates/configmap.yaml_ \- configuration information for Nginx.
*   _templates/customresourcedefinition.yaml_ \- CustomResourceDefinitions for the ingress controller.
*   _templates/clusterrole.yaml_ \- specifications of the ClusterRole and ClusterRoleBinding role-based access control (rbac) components for the ingress controller.
*   _ingress-deploy-nano.yaml_ \- container specifications that pull the nginx image that contains the references to the CP Nano Agent.
*   _templates/ingress.yaml_ \- specification for the ingress settings for the application that point to your inbound service.
*   _templates/secrets.yaml_ \- secrets file.
*   _templates/service.yaml_ \- specifications for the ingress controller, e.g. LoadBalancer listening on port 80, forwarding to nodePort 30080 of the application 
## Installing the Chart 
Define your application in the CloudGuard AppSec application of the Check Point Infinity Portal according to the CloudGuard AppSec Deployment Guide section on AppSec (WAAP) Management .

Once the application has been configured in the CloudGuard Portal, retrieve the value for the nanoToken.

Download the latest release of the chart here:
```bash
https://github.com/CheckPointSW/Infinity-Next/tree/main/deployments
```
Next, install the chart with the chosen release name (e.g. `my-release`), run:

```bash
$ helm install my-release cpappsec-0.1.2.tgz --namespace="{your namespace}" --set nanoToken="{your AppSec token string here}" --set appURL="{your appURL}" --set mysvcname="{your app Service Name}" --set mysvcport="{your app service port}" 
```
These are additional optional flags:
```bash
--set cpFog="{Check Point CloudGuard Fog Service}"
--set cpappsecnginxingress.properties.imageRepo="{a different repo}"
--set cpappsecnginxingress.properties.imageTag="{a specific tag/version}"
--set cpappsecnanoagent.properties.imageRepo="{a different repo}"
--set cpappsecnanoagent.properties.imageTag="{a specific tag/version}"
```
## Uninstalling the Chart
To uninstall/delete the `my-release` deployment:
```bash
$ helm delete my-release -n {your namespace}
```
This command removes all the Kubernetes components associated with the chart and deletes the release.

## Configuration

Refer to [values.yaml](values.yaml) for the full run-down on defaults. These are the Kubernetes directives that map to environment variables for the deployment.

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example,

```bash
$ helm install my-release checkpoint/cpappsec-0.1.2.tgz --namespace="myns" --set nanoToken="4339fab-..." --set appURL="myapp.mycompany.com" --set mysvcname="myapp" --set mysvcport="8080" 
```
Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```bash
$ helm install my-release -f values.yaml checkpoint/cpappsec
```
> **Tip**: You can use the default [values.yaml](values.yaml)

### Generating results and Testing the Application

Deploy your Application into the Kubernetes Cluster (e.g., juice-shop.yaml)
Deploy the helm chart
Use kubectl to get the IP address of the cpappsec service.
```
kubectl get all
```

Ensure your DNS or host file maps the application URL to the IP address of the cpappsec service IP. 

Open a Firefox browser tab (If you are testing the juice-shop application, which will not work for the juice-shop example) and go to _**http://{yourapplicationURL}**_

For testing with the juice-shop application, deploy the app in your Kubernetes environment. Click on “Account” and select “Login”. In the username field, enter 
```
select * from ‘1’=’1’ --
```
Enter any password and click “Log In”.  For the normal juice-shop app without CloudGuard AppSec, the results show a compromised application. With CloudGuard AppSec protection, you will see the login blocked.

Review the log files in the Infinity Portal to see the Results
* * *
