# CFMapserver

CFmapserver is a wrapper utility for deploying [MapServer](https://mapserver.org/about.html#aboutMapServer) on Kubernetes, and enabling automated updates of its mapfile sources.

## How it works

CFmapserver is available as a [Helm](https://helm.sh/) chart, which enables deploying MapServer instances on Kubernetes with minimum configuration.
MapServer instances get deployed as a containers and serve map files from `/etc/mapserver` folder. The contents of this folder are populated based on the contents of your S3 bucket. 
If the map files in the buckets change (e.g. are added, deleted or updated), a Kubernetes cronjob will periodically update the `/etc/mapserver` contents as well.

## Prerequisites
* A running Kubernetes cluster (e.g. on CloudFerro WAW3-1 cloud, with integrated EO data repository) -> [CloudFerro Kubernetes on Magnum installation guide](https://creodias.docs.cloudferro.com/en/latest/kubernetes/How-to-Create-a-Kubernetes-Cluster-Using-Creodias-OpenStack-Magnum.html)
* kubectl installed -> [Install kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
* Helm installed -> [Installing Helm](https://helm.sh/docs/intro/install/)
* An S3 bucket with credentials, where a sample mapfile is stored -> [CloudFerro guideline, EC2 keys generation](https://creodias.docs.cloudferro.com/en/latest/general/How-to-generate-ec2-credentials-on-Creodias.html?highlight=generate%20s3)

## How to use this chart?
Add repository locally:

```
helm repo add cfmapserver https://PoulTur.github.io/cfmapserver/charts
```


Verify the repo was added:

```
helm search repo
```

Create a customization file `my-values.yaml` by pulling the default file, then edit this file. At minimum, fill in the S3 coordinates (name of your bucket and S3 credentials). You can override other default values, in line with the configuration guidelines.

```
wget https://raw.githubusercontent.com/PoulTur/cfmapserver/main/charts/values.yaml -O my-values.yaml
```


Install the chart. We are adding here the custom flags `--namespace` and `--create-namespace` in order to have all artifacts contained in a dedicated namespace. You could skip these flags to install into the default namespace.

```
helm install cfmapserver cfmapserver/CFmapserver -f custom-values.yaml --namespace cfmapserver --create-namespace
```

Check that the service is running:

```
kubectl get services -n cfmapserver
```

After a couple minutes you should see a public IP assigned to service. Verify that mapserver is running e.g. by typing:

```
http://<your-public-ip>/?map=/etc/mapserver/<your-mapfile.map>&service=WMS&request=GetCapabilities
```


## Configuration guidelines

The following table lists the configurable parameters of the template Helm chart and their default values.

| Parameter                  | Description                                     | Default                                                    |
| -----------------------    | ---------------------------------------------   | ---------------------------------------------------------- |
| `s3Maps.host`              | S3 endpoint URL for storing map files                                 | `https://s3.waw3-1.cloudferro.com`                         |
| `s3Maps.bucket`            | S3 bucket for map files (please store mapfiles directly in this S3 bucket, use of folders is currently not supported with the default setup)        | ``     |
| `s3Maps.accessKey`         | S3 access key for map files                                 | ``                                                         |
| `s3Maps.secretKey`         | S3 secret key for map files                                  | ``                                                         |
| `cronJobSchedule`          | Synchronization schedule between S3 bucket and map files path, in cron format (default is daily at midnight)  | `0 0 * * *`  |
| `image.repository`         | Repository of MapServer Docker image            | `camptocamp\mapserver`                                     | 
| `image.tag`                | Version/Tag of MapServer image                  | `7.6-20-04`                                                |
| `image.mapFilesPath`       | Path to serve map files ((do not change if using the default MapServer image)    | `/etc/mapserver` |
| `initImage.repository`     | Repository with Docker image of the initContainer used for syncing mapfiles | `paultur\mapserver-init`  |
| `initImage.tag`            | Tag of the initContainer image                  | `0.1.0`
| `service.type`             | Kubernetes service type exposing port           | `LoadBalancer`                                             |
| `service.port`             | TCP Port for this service                       | 80                                                         |
| `resources.limits.memory`  | Pod memory resource limits in MiB (1Mi = 1 MiB)       | `1024Mi`                                |
| `resources.limits.cpu`     | Pod CPU resource limits (1000m = 1 (v)CPU core)       | `1000m`                                 |
| `autoscaling.enabled`      | Whether to apply Horizontal Pod Autoscaling (HPA) | false |
| `replicas`                 | How many instances of MapServer to deploy (if autoscaling is disabled) | 1  |
| `autoscaling.minReplicas`  | Min # of MapServer instances (if autoscaling is enabled)                                             | 1 |
| `autoscaling.maxReplicas`  | Max # of MapServer instances (if autoscaling is enabled)                                              | 10 |


## Customizing the MapServer initialization script

Prior to deploying each MapServer container by CFmapserver, an initialization script runs in a separate [Kubernetes initContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/). The [default script](https://raw.githubusercontent.com/PoulTur/cfmapserver/main/init/mapserver-init.py) checks for the contents of the S3 bucket defined in `values.yaml`, pulls the files with `.map` extension and copies them to the mounted `/etc/mapserver` folder. This folder is then mounted to the MapServer container as well, thus the mapfiles from S3 get synced to MapServer.

The sequence described above takes place the first time when CFmapserver is deployed. Then, a cronjob running on the cluster, reinitiates this sequence periodically on a specified time interval. 

The init container script can be substituted by any other script, performing custom logic e.g. applying custom changes to mapfiles prior to loading them to MapServer, pulling from different source, setting other environment variables etc. To apply such customization, you can create a custom Docker image containing a modified script and place it in a Docker repository. Then point to this image in the `values.yaml` (`initImage.repository` and `initImage.tag` values).

The caveats about the custom init script:
* The script will have access to the environment variables provided in `values.yaml` (S3_MAPS_HOST, S3_MAPS_BUCKET, S3_MAPS_ACCESS_KEY, S3_MAPS_SECRET_KEY)
* The script upon running should re it should upload mapfiles to this location

## Accessing protected data in S3 buckets

Mapfiles could point to protected geospatial data in S3 buckets (e.g. tiff images, .vrt files), where authentication using EC2 keypair is required (access key, secret key).
The keys provided in the S3Maps bucket, do not directly cater for this use case.

To access geo data in private buckets, the initialization script should be ammended to use [VSIS3](https://gdal.org/user/virtual_file_systems.html#vsis3-aws-s3-files). The S3 keys should then be separately provided to VSIS3 driver either via configuration file, config section in map files or via required environment variables.