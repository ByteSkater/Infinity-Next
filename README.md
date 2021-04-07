# Check Point Infinity Next Repository Overview <br/>(with added support for helm chart-based deployment in AWS EKS using NLB)

Check Point Infinity Next repository.
The repository contains:
* Tools and scripts that can be used for Public Cloud solutions
* Terraform, Helmchart and CloudFormation files
* Community-supported content

## 2021-04-07 Purpose of this fork by ByteSkater
**(see also [/contrib/helmIngressController/README.md](/contrib/helmIngressController/README.md))**

This repo is a fork from https://github.com/CheckPointSW/Infinity-Next.git .
There was a limitation in the existing cpappsec-0.1.2.tgz helm chart (see /deployments, sources: /contrib/helmIngressController) that didn't allow successful Check Point AppSec deployment in AWS EKS environments, I added two lines in /contrib/helmIngressController/templates/ingress.yaml that now allow also successful deployments of the helm chart in AWS EKS, see following details.
I left the rest of the repo untouched.

Here's what to do step-by-step *in addition* to the official documentation further below:

* Manually create a few Elastic IP (EIP) addresses in your AWS account to be used with the EKS ELB, which will be deployed as part of the K8S service definition in the helm chart (you need to create one EIP per ELB subnet (for the AZ redundancy), typically this would be 3 EIPs) 
* Create a public DNS A record (e.g. in Route53) referencing all of those previously created public IPs (EIPs) with a DNS name of your choice
(alternatively you can probably also work with entries in your local HOSTS file, if you consider it sufficient to have connectivity from your local machine only, but the DNS-based approach is more realistic and comfortable)
* In the /contrib/helmIngressController/templates/service.yaml the Load-Balancer type was adjusted from Classic Load Balancer (default in AWS for EKS services of type “loadbalancer”) to Network Loadbalancer (NLB) to get support for EIP assignments during LB creation. This fixes the issue that the ELBs (=CLBs) DNS name and corresponding IP addresses per AZ in AWS are usually only available AFTER the helm chart was deployed.
Additional benefit: These public EIPs also will survive deletion and reapplication of the Helm chart and you do not need to readjust your Host A entry in DNS again after redeployments which comes in handy e.g. when used in demos or lab environments. 
To implement the change in the service.yaml (as part of the helm chart) the following two lines were added with additional annotations to set the ELB type to NLB and to reference the EIPallocs.
<br/>**Important: You need to replace eipalloc-00000, eipalloc-11111, eipalloc-22222 below with your own, previously created 3 eipalloc values:**
```
service.beta.kubernetes.io/aws-load-balancer-type: nlb
service.beta.kubernetes.io/aws-load-balancer-eip-allocations: eipalloc-00000,eipalloc-11111,eipalloc-22222
```
* After you adjusted the line above you need to pack the helm chart as a .tgz file (you can use “tar –cvzf”) so that you can invoke it in the next step.
* Apply your customized helm chart by handing over the previously created DNS name in the following parameter: "--set appURL= ...." (and use it of course in the AppSec WebUI configuration)

## Related Products and Solutions
* Infinity Next
* CloudGuard
* IoT Firmware Scanner

## References
* Infinity Portal is available at https://portal.checkpoint.com
* Infinity Next documentation is available at https://sc1.checkpoint.com/documents/Infinity_Portal/WebAdminGuides/EN/Infinity-Next-Admin-Guide/Topics-Infinity-Next/Overview-Infinity-Next.htm 
* Infinity Next CheckMates community is available at https://community.checkpoint.com/t5/Infinity-Next/bd-p/infinity-next
