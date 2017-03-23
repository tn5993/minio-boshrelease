# BOSH Minio Release

A BOSH release of Minio Object Storage Server. For more information
about Minio visit the repo [here](https://github.com/minio/minio).

This release can be used to deploy Minio in standalone, single-node
mode as well as in distributed mode on multiple nodes.

# How to try out this release

The following describes trying out the release on your local system
using BOSH Lite.

## 0. Install Bosh CLI and BOSH Lite

Install bosh-lite and bosh_cli as
described [here](https://github.com/cloudfoundry/bosh-lite).

After setting up the VM with bosh-lite, run the `bin/add-route` script
to add routes to access your VMs. The script is present in the
bosh-lite repo.

## 1. Clone and cd into this repo

## 2. Download and upload stemcell to BOSH

The BOSH-lite Warden stemcell is used here and is
available
[here](https://s3.amazonaws.com/bosh-core-stemcells/warden/bosh-stemcell-3363.12-warden-boshlite-ubuntu-trusty-go_agent.tgz).

After downloading it, use a command like below to upload it to your
VM:

``` shell
bosh upload stemcell ~/Downloads/bosh-stemcell-3363.12-warden-boshlite-ubuntu-trusty-go_agent.tgz --skip-if-exists

```

## 3. Create and upload the BOSH release

``` shell
bosh create release --force
    Please enter development release name: minio
bosh upload release
```

## 4. Create manifest from examples files in `manifests`

Copy a manifest in the `manifests/` directory and replace the
`director_uuid` field with the one from your BOSH director.

The UUID to put in the manifest can be found with:

``` shell
bosh status
```

## 5. Deploy

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

# How to update this release with a newer version of Minio

The latest version of the Minio binary for amd64 Linux is available at
https://dl.minio.io/server/minio/release/linux-amd64/

Pick the file name that has the version suffix such as:

``` shell
MINIO_BIN_NAME=minio.RELEASE.2017-03-16T21-50-32Z
```

Download it locally and verify the shasum.


## 1. Create new package

``` shell
$ bosh generate package ${MINIO_BIN_NAME}

$ tree packages/${MINIO_BIN_NAME}/
packages/minio.RELEASE.2017-03-16T21-50-32Z/
├── packaging
├── pre_packaging
└── spec

0 directories, 3 files

```

## 2. Update `packaging` and `spec` files in the generated package directory

The update just replaces the name of the package used in the
`packaging` script and in the `spec` files.

## 3. Update blob for new package

``` shell
bosh add blob <local-path> <pkg-name>
```

## 4. Update `minio-server` job

In the job directory, update the dependencies listed in the
`jobs/minio-server/spec` file, similar to:

``` shell
packages:
  - minio.RELEASE.2017-03-16T21-50-32Z

```

Also, in the template script `jobs/minio-server/templates/ctl.erb` update the
BINPATH variable with the new version name of the minio package.

``` shell
BINPATH=/var/vcap/packages/minio.RELEASE.2017-03-16T21-50-32Z

```

## 5. Try it out

``` shell
bosh create release --force
bosh upload release # also upload stemcell if needed
bosh deployment manifests/manifest-fs/yml # Set manifest
bosh deploy
```

Test if the deployment worked and finally, if everything is working as
expected, create a final release with:

``` shell
bosh upload blobs # This requires you to configure s3 creds in config/private.yml
bosh create release --final
```
