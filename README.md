# BOSH Minio Release

A BOSH release of Minio Object Storage Server. For more information
about Minio visit the repo [here](https://github.com/minio/minio).

This release can be used to deploy Minio in standalone, single-node
mode as well as in distributed mode on multiple nodes.

# How to try out this release

The following describes trying out the release on your local system
using BOSH Lite.

## Install Bosh CLI and BOSH Lite

Install bosh-lite and bosh_cli as
described [here](https://github.com/cloudfoundry/bosh-lite).

After setting up the VM with bosh-lite, run the `bin/add-route` script
to add routes to access your VMs. The script is present in the
bosh-lite repo.

## Clone and cd into this repo

## Download and upload stemcell to BOSH

The BOSH-lite Warden stemcell is used here and is
available
[here](https://s3.amazonaws.com/bosh-core-stemcells/warden/bosh-stemcell-3363.12-warden-boshlite-ubuntu-trusty-go_agent.tgz).

After downloading it, use a command like below to upload it to your
VM:

``` shell
bosh upload stemcell ~/Downloads/bosh-stemcell-3363.12-warden-boshlite-ubuntu-trusty-go_agent.tgz --skip-if-exists

```

## Create and upload the BOSH release

``` shell
bosh create release --force
    Please enter development release name: minio
bosh upload release
```

## Create manifest from examples files in `manifests`

Copy a manifest in the `manifests/` directory and replace the
`director_uuid` field with the one from your BOSH director.

The UUID to put in the manifest can be found with:

``` shell
bosh status
```

## Deploy

Set the deployment manifest by providing the name of your manifest,
for example:

``` shell
bosh deployment manifest/manifest-fs.yml
```

Then run the deploy command:

``` shell
bosh deploy
```

With the settings in the example manifest, you should be able to
access the minio server at `http://10.244.0.2:9001`, with the mc
tool:

``` shell

###### Using settings in the example manifest ######
mc config host add boshminio http://10.244.0.2:9001 minio minio123
# Test it out:
mc ls boshminio
mc mb boshminio/bucket
mc cp /etc/issue boshminio/bucket/
```

For deploying a distributed version, just set the number of desired
instances in the manifest file along with that many `static_ips` in
the `jobs` section of the manifest, as shown in the example manifest
at `manifests/manifest-dist-4node.yml`.

With the settings in that example manifest, you can test the
deployment with mc as follows:

``` shell
mc config host add boshminio1 http://10.244.0.2:9001 minio minio123
mc config host add boshminio2 http://10.244.0.3:9001 minio minio123
mc config host add boshminio3 http://10.244.0.4:9001 minio minio123
mc config host add boshminio4 http://10.244.0.5:9001 minio minio123

mc mb boshminio1/bucket # Create a bucket
mc ls boshminio{1..4} # List all the 4 minio endpoints. Should see the
                      # bucket printed four times.
```
