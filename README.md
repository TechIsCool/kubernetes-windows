# Kubernetes on Windows - Notes

**NOTE**: My Goal is to get Windows working with AWS EKS or at least a cluster running in AWS. Some items defined in configuration will be targeted at AWS specifically. 

### Versions

Current supported versions: 

- Windows Server 2016 - 1803

Targeted for General Availability:

- Window Server 2019 - 1809?

### People that are working on the #sig-windows

- [Slack](http://slack.k8s.io/) - #sig-windows
- [Project Board - #sig-windows](https://github.com/PatrickLang/k8s-project-management/projects/1)
- [End to End Windows Testing for Azure](https://github.com/e2e-win/e2e-win-prow-deployment)
- [#SIG Windows Meetings - Youtube](https://www.youtube.com/playlist?list=PL69nYSiGNLP2OH9InCcNkWNu2bl-gmIU4)
  - [Meeting Notes - Google Docs](https://docs.google.com/document/d/1Tjxzjjuy4SQsFSUVXZbvqVb64hjNAG5CQX8bK7Yda9w/edit#heading=h.kbz22d1yc431)
- [kubernetes/community/sig-windows](https://github.com/kubernetes/community/tree/master/sig-windows)
- [PatrickLang/kubernetes-windows-dev](https://github.com/PatrickLang/kubernetes-windows-dev)

### Shared Works for Windows

- [containernetworking/plugins](https://github.com/containernetworking/plugins)
- [coreos/falnnel-cni](https://github.com/coreos/flannel-cni)

### Troubleshooting

- [Microsoft Docs](https://docs.microsoft.com/en-us/virtualization/windowscontainers/kubernetes/common-problems)

### Kubernetes Notes and helpful links

- [Checking Kube CIDR for flannel](https://prefetch.net/blog/2018/02/20/generating-kubernetes-pod-cidr-networks-with-kubectl-and-jq/)
- [#sig-windows github Issues](https://github.com/kubernetes/kubernetes/labels/sig%2Fwindows)

#### Docker Notes

- [Cleanup of Dead Docker Container](https://odino.org/spring-cleaning-of-your-docker-containers/)

## Implementations of k8s for windows

### Microsoft SDN

- [Microsoft/SDN](https://github.com/Microsoft/SDN)

This is the core repo that MS uses for publishing stuff for CNI. 

When working to setup my first k8s cluster I read over this repo and it felt a bit old and none functional. I tried using their flannel.exe and it did not work in my tests. Overall not sure where to go without some more commits from them. The kublet.exe stuff works correctly.

### Ptylenda

- [k8s-for-windows](https://github.com/ptylenda/kubernetes-for-windows/tree/master/ansible/roles)

I have implemented this from the ground up. It seems to work alright but I struggle getting flannel.exe working correctly. Nodes register and function correctly but they restart due to pod-sandbox needing to be reconfigured continually. This seems to even happen with SDN/Rancher if the flannel conf is defined wrong.

### Rancher

- [Dockerfile for Windows](https://github.com/rancher/kubernetes-windows-dockerfiles)
- [hyperkube.ps1 - Rancher launcher for k8s](https://github.com/rancher/rancher/blob/master/package/windows/hyperkube.ps1)
  - [rancher-hyperkube.md](./rancher-kyperkube.md)

This seems to work alright but flannel again did not work correctly tracking down the files seemed to not work very well but at least it was fruitful and gave some new files.

**WARNING**: These IP Addresses where just to get valid configs out they are not accurate.

```powershell
.\hyperkube.ps1 `
  -KubeClusterCIDR 10.244.0.0/16 `
  -KubeClusterDomain cluster.local `
  -KubeServiceCIDR 127.0.0.0 `
  -KubeDnsServiceIP 172.20.0.10 `
  -KubeCNIComponent flannel `
  -KubeCNIMode win-bridge `
  -KubeletCloudProviderName aws `
  -KubeletOptions $(@"
--v=4;
--pod-infra-container-image=kubeletwin/pause;
--allow-privileged=true;
--cloud-provider=aws;
--cluster-dns=172.20.0.10;
--cluster-domain=cluster.local;
--register-node=true;
--anonymous-auth=false;
--kubeconfig='C:\etc\kubernetes\kubelet.conf';
--pod-manifest-path='C:\etc\kubernetes\manifests';
--authentication-token-webhook;
--authorization-mode=Webhook;
--client-ca-file='C:\etc\kubernetes\pki\ca.crt';
--image-pull-progress-deadline=20m;
--resolv-conf='';
--enable-debugging-handlers;
--feature-gates=RotateKubeletServerCertificate=true;
"@ -replace "`t|`n|`r","") `
  -NodeIP 10.14.34.71 `
  -NodeName "ip-10-14-34-71.us-west-2.compute.internal" `
  -KubeproxyOptions $(@"
--v=4;
--proxy-mode=userspace;
--kubeconfig='C:\etc\kubernetes\kubelet.conf'
"@ -replace "`t|`n|`r","")

```

net-cni.conf

```
-KubeClusterCIDR 10.244.0.0/16 `
-KubeClusterDomain cluster.local `
-KubeServiceCIDR 127.0.0.0 `
-KubeDnsServiceIP 172.20.0.10 `
-NodeIP 10.14.34.71 `
# -NetworkRange 10.14.34.0/25 # Note this value is not defined and auto computed by hyperkube.ps1
```

flannel conf

```
{  
   "capabilities":{  
      "dns":true
   },
   "delegate":{  
      "dns":{  
         "search":[  
            "svc.cluster.local"
         ],
         "nameservers":[  
            "172.20.0.10"
         ]
      },
      "policies":[  
         {  
            "value":{  
               "ExceptionList":[  
                  "10.244.0.0/16",
                  "127.0.0.0",
                  "10.14.34.0/25"
               ],
               "Type":"OutBoundNAT"
            },
            "name":"EndpointPolicy"
         },
         {  
            "value":{  
               "DestinationPrefix":"127.0.0.0",
               "NeedEncap":true,
               "Type":"ROUTE"
            },
            "name":"EndpointPolicy"
         },
         {  
            "value":{  
               "DestinationPrefix":"10.14.34.71/32",
               "NeedEncap":true,
               "Type":"ROUTE"
            },
            "name":"EndpointPolicy"
         }
      ],
      "type":"win-l2bridge"
   },
   "name":"cbr0",
   "type":"flannel",
   "cniVersion":"0.2.0"
}
```

##### Compiled files - thxCode

[flannel.exe, win-cni.exe etc.](https://github.com/thxCode/containernetworking-plugins/releases/tag/v0.1.0-rancher)

- Add dns capabilities for Windows CNI plugins - Merged [[67435](https://github.com/kubernetes/kubernetes/pull/67435)]

[flanneld.exe](https://github.com/thxCode/coreos-flannel/releases/tag/v0.1.0-rancher)

- Windows "host-gw" & "vxlan" support - Merged [[1042](https://github.com/coreos/flannel/pull/1042)]

Currently releases have not been cut with the merged code. Until then it sounds like custom builds of master might work.

### PJH

- [k8s-the-hard-way](https://github.com/pjh/kubernetes-the-hard-way)
  - [Node Setup psm1](https://github.com/pjh/kubernetes/blob/windows-up/cluster/gce/win1803/k8s-node-setup.psm1)

### Glenns West

- [hybrid](https://github.com/glennswest/hybrid/blob/master/3.9/bin/archive/start.ps1)

### David  Jahn

[deploy-kube-windows](https://github.com/davidjahn/deploy-kube-windows)



