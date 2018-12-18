# Disection of Rancher Hyperkube configuration.

This PowerShell script can be [found here](https://github.com/rancher/rancher/blob/master/package/windows/hyperkube.ps1)



### Config-CNI-Flannel

  - Drop JSON File 'c:\etc\kube-flannel\net-conf.json'

### Restart-Proxy

  - Sleep 5s
  - docker restart nginx-proxy
  - Sleep 5s

### Repair-Cloud-Routes

  - aws Add-Routes -IPAddrs ("169.254.169.254/32", "169.254.169.250/32", "169.254.169.251/32")
  - azure Add-Routes -IPAddrs ("169.254.169.254/32")

### Stop-Flannel

  - Get-Process 'flanneld*" | Stop-Process -Force

### Start-Flannel

  - Check that Files exists
  - Retry = 6
    - At 3 Restart-Proxy
    - At 1 Try one last time with Redirect Logging
    - At 0 Throw Error
  - Delay 20s and Remove -1 from Retry
  - Retry = 6
    - Check HnsNetwork for Flannl Backend
    - At 0 Throw Error
  - Delay 5s and Remove -1 from Retry
  - Restart-Proxy
  - Repair-Cloud-Routes

### Stop-Kubelet

  - Get-Process 'kubelet*" | Stop-Process -Force

### Start-Kubelet

  - Check that Files exist
  - Config-CNI-Flannel -Restart ?
  - Create-File 'c:\var\lib\kubelet\config.json' if $KubeletDockerConfig defined
  - Collect Kubelet Arguments
  - Retry = 6
    - At 1 Try one last time with Redirect Logging
  - Delay 15s and Remove -1 from Retry
  - Start-Flannel -Restart ?

### Stop-Kube-Proxy

  - Get-Process 'kube-proxy*" | Stop-Process -Force

### Start-Kube-Proxy

  - Stop-Kube-Proxy
  - Delay 10-30s
  - Check that Files exist
  - Get-HnsPolicyList | Remove-HnsPolicyList
  - Collect KubeProxy Args
  - Retry = 6 
    - At 1 Try one last time with Redirect Logging
  - Delay 10s and Remove -1 from Retry
  - Restart-Proxy

### Init

  - if cleanup kube.tip
    - Stop-Kube-Proxy
    - Stop-Kubelet
    - Remove Binary Files
  - if cleanup cni.tip
    - Stop-Flanneld
    - Remove Binary Files
  - Repair-cloud-routes
  - Set Nodename to meta-data/hostname

### Main

  - Get-Processes
    - Should be running 'kubelet*'
    - Should be running 'kube-proxy*'
    - Should be running 'flannel*'
  - Recover
    - if kubelet != running
      - Start-Kubelet -Restart
    - if kube-proxy != running
      - Start-Kube-Proxy -Restart
    - if flanneld != running
      - Start-Flanneld -Restart
   - else
     - Start-Kubelet
     - Start-Kube-Proxy