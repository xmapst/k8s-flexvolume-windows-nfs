# Kubernetes nfs Volume Driver for windows
A simple volume driver based on [Kubernetes' Flexvolume](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md) that allows Kubernetes hosts to mount nfs volumes (nfs shares) into pods and containers.

It has been tested under Kubernetes versions:

* 1.8.x
* 1.9.x
* 1.10.x
* 1.11.x
* 1.12.x
* 1.13.x
* 1.14.x
* 1.15.x
* 1.16.x
* 1.17.x
* 1.18.x

## DaemonSet Installation
When dealing with a large cluster, manually copying the driver to all hosts becomes inhuman. As proposed in Flexvolume’s documentation, the recommended driver deployment method is to have a DaemonSet install the driver cluster-wide automatically.

A Docker image `mcr.microsoft.com/powershell:7.1.0-preview.5-nanoserver-1809` is available for this purpose, which can be deployed into a Kubernetes cluster using the `01-flexvolume-nfs-windows.yaml` from this repository. The image is built `FROM nanoserver`, so the it’s essentially very small.

```bash
git clone https://github.com/xmapst/k8s-flexvolume-windows-nfs.git
cd k8s-flexvolume-windows-nfs
kubectl create -f 00-q1-win~nfs.cmd.yaml
kubectl create -f 01-flexvolume-nfs-windows.yaml
```

> NOTE: Note: This deployment automatically installs host dependencies and does not need to be completed manually on all hosts.

If you need to tweak or customize the installation, you can modify the `01-flexvolume-nfs-windows.yaml` directly.

Installing is a one time job. So, once you have verified that it’s completed, the DaemonSet can be safely removed.

```bash
kubectl delete -f 01-flexvolume-nfs-windows.yaml
kubectl delete -f 00-q1-win~nfs.cmd.yaml
```

## The Volume Plugin Directory
As of today with Kubernetes v1.16, the kubelet’s default directory for volume plugins is `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/`. This could be different if your installation changed this directory using the `--volume-plugin-dir` parameter.

Please, review the kubelet command line parameters (namely `--volume-plugin-dir`) and make sure it matches the directory where the driver will be installed.

You can modify `01-flexvolume-nfs-windows.yaml` and change the field `spec.template.spec.volumes.hostPath.path` to the path used by your Kubernetes installation.

## Customizing the Vendor/Driver name
By default, the driver installation path is `$KUBELET_PLUGIN_DIRECTORY/q1autoops-win/nfs.cmd`.

For some installations, you may need to change the vendor + driver name. You can define the installed vendor name/directory by adjusting the 01-flexvolume-nfs-windows.yaml plugin directory.

```yaml

## snippet ##

      containers:
        - image: mcr.microsoft.com/powershell:7.1.0-preview.5-nanoserver-1809
          env:
          - name: DRIVER
            value: q1autoops-win
          command: 
          - pwsh.exe
          args:
          - /Command
          - mkdir -Force c:\host\${DRIVER}~nfs.cmd;
          - cp -Force c:\kubelet-plugins\flexvolume.ps1 c:\host\${DRIVER}~nfs.cmd;
          - cp -Force c:\kubelet-plugins\nfs.cmd c:\host\${DRIVER}~nfs.cmd;
          - cp -Force c:\kubelet-plugins\nfs.ps1 c:\host\${DRIVER}~nfs.cmd;
          - cat c:\kubelet-plugins\Readme.md;
          - while ($true) {start-sleep -s 3600}

## snippet ##

```

The example above will install the driver in the path `$KUBELET_PLUGIN_DIRECTORY/q1autoops-win~nfs.cmd/nfs.cmd`. For the most part, changig the `DRIVER` variable should be enough to make your installation unique to your needs.

## Example of Deployment
The following is an example of Deployment that uses the volume driver.
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs
  namespace: nfs
  labels:
    app: nfs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs
  template:
    metadata:
      labels:
        app: nfs
    spec:
      nodeSelector:
        beta.kubernetes.io/os: windows
      tolerations:
      - key: "os"
        operator: "Equal"
        value: "windows"
        effect: "NoSchedule"
      containers:
      - name: nfs
        image: mcr.microsoft.com/powershell:7.1.0-preview.5-nanoserver-1809
        imagePullPolicy: IfNotPresent
        command: ["ppwsh.exe"]
        args: ["/Command", "ping", "-t", "127.0.0.1", ">>", "/d/test.txt"]
        volumeMounts:
        - name: nfs-volume
          mountPath: /d
      imagePullSecrets:
      - name: shenzhen-regcred
      volumes:
      - name: nfs-volume
        flexVolume:
          driver: "q1-win/nfs.cmd"
          options:
            # source can be in any of the following formats
            # \\servername\share\path  (\'s will need to be escaped)
            # nfs://servername/share/path
            # //servername/share/path
            source: "nfs://xxx-xxx.cn-shenzhen.nas.aliyuncs.com/!"
```
