This file contains information for maintainers.

## How to update this release with a newer version of Minio

The latest version of the Minio binary for amd64 Linux is available at
https://dl.minio.io/server/minio/release/linux-amd64/

Pick the file name that has the version suffix such as:

``` shell
MINIO_BIN_NAME=minio.RELEASE.2017-03-16T21-50-32Z
```

Download it locally and verify the shasum.


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
bosh add blob <local-path> <pkg-name>
```

### Update `minio-server` job

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

### Try it out

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
