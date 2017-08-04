This file contains information for maintainers.

## How to update this release with a newer version of Minio

The latest version of the Minio binary for amd64 Linux is available at
https://dl.minio.io/server/minio/release/linux-amd64/

Download it locally and verify the shasum. Rename the binary file to
just `minio`. Example commands:

``` shell
MINIO_BIN_NAME=minio.RELEASE.2017-03-16T21-50-32Z
# For a current release:
wget -O /tmp/minio https://dl.minio.io/server/minio/release/linux-amd64/${MINIO_BIN_NAME}
# OR, if adding an older release, the URL differs slightly:
wget -O /tmp/minio https://dl.minio.io/server/minio/release/linux-amd64/archive/${MINIO_BIN_NAME}

# Then verify the sha256sum, like below (e.g. commands and output are below):
$ sha256sum /tmp/minio
f85839e391ec616e2a0d28740dba1d3f662f624cc13c3683a6cad8cd019ba17d  /tmp/minio
$ curl https://dl.minio.io/server/minio/release/linux-amd64/${MINIO_BIN_NAME}.sha256sum
f85839e391ec616e2a0d28740dba1d3f662f624cc13c3683a6cad8cd019ba17d minio.RELEASE.2017-03-16T21-50-32Z
```

Though the local file is named `/tmp/minio`, we will use the full name
(`$MINIO_BIN_NAME`) as the name of the package in the BOSH release.

### Create new package

``` shell
$ bosh generate package ${MINIO_BIN_NAME}

$ tree packages/${MINIO_BIN_NAME}/
packages/minio.RELEASE.2017-03-16T21-50-32Z/
├── packaging
├── pre_packaging
└── spec

0 directories, 3 files

```

### Update `packaging` and `spec` files in the generated package directory

The update just replaces the name of the package used in the
`packaging` script and in the `spec` files.

### Update blob for new package

``` shell
bosh add blob /tmp/minio ${MINIO_BIN_NAME}
```

### Update `minio-server` job

In the job directory, update the dependencies listed in the
`jobs/minio-server/spec` file, similar to:

``` shell
packages:
  - minio.RELEASE.2017-03-16T21-50-32Z

```

Also, in the template script `jobs/minio-server/templates/ctl.erb` update the
BINPATH variable with the new version name of the minio package. Example:

``` shell
BINPATH=/var/vcap/packages/minio.RELEASE.2017-03-16T21-50-32Z

```

### Remove old package(s)

We remove old package directory as it is not needed for the new release.

``` shell
# Run for each older package in the packages dir (there will be usually be just one)
git rm -f packages/minio.RELEASE.xxxxxxx
```

### Try it out

Commit changes to git before testing the candidate release:

``` shell
git commit -a -m 'Add files for new release'
bosh create release --force
bosh upload release # also upload stemcell if needed
bosh deployment manifests/manifest-fs.yml # Set manifest
bosh deploy
```

To test if the deployment worked - we need to connect to the deployed
Minio instance inside the bosh-lite Vagrant VM. Run the following
script from the bosh-lite repo directory to enable network access:

``` shell
bin/add-route
```

It will ask for sudo permission to add a network route.

Now, we can attempt to access the instance via mc:

``` shell
mc config host add b-miniofs http://10.244.0.2:9001 minio minio123
mc mb b-miniofs/newbucket
```

If the second command above worked, we have accessed the deployed
Minio instance. Carry out any further tests to be sure the server is
working as expected.

If everything is working as expected, upload blobs for final release
and commit:

``` shell
bosh upload blobs # This requires you to configure s3 creds in config/private.yml
git commit -a -m 'Add blobs'
```

And create the final release, and commit files it generates. Example
commands:

``` shell
bosh create release --final
git add .final_builds/jobs/minio-server/index.yml
git add releases/minio/index.yml
git add .final_builds/packages/minio.RELEASE.2017-06-13T19-01-01Z/
git releases/minio/minio-4.yml
git commit -m 'Publish release minio.RELEASE.2017-06-13T19-01-01Z'
```
