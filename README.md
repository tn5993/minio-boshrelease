# .bosh-minio-release
Temporary repo for developing a Bosh Minio release.

# How to

## 0. Install Bosh CLI and BOSH Lite

Install bosh-lite
from [here](https://github.com/cloudfoundry/bosh-lite) along with
bosh_cli (also described in the link).

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

## 4. Create manifest from Example

Copy a manifest in the `manifest/` directory and replace the
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
access the minio server at your `http://10.244.0.2:9001`, with the mc
tool:

``` shell

###### Using settings in the example manifest ######
mc config host add boshminio http://10.244.0.2:9001 minio minio123
# Test it out:
mc ls boshminio
mc mb boshminio/bucket
mc cp /etc/issue boshminio/bucket/
```
