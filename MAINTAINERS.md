This file contains information for maintainers.

## How to update this release with a newer version of Minio using Bosh V2

In this section, the `bosh-cli` command is bound to Bosh V2 binary.

### Fetch and verify new binary

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
$ bosh-cli generate-package ${MINIO_BIN_NAME}

$ tree packages/${MINIO_BIN_NAME}/
packages/minio.RELEASE.2017-03-16T21-50-32Z/
├── packaging
└── spec

0 directories, 2 files

```

### Update `packaging` and `spec` files in the generated package directory

The main content for these two files may be taken from the existing
(older released) package directory.

Then just replace the name of the package with the correct RELEASE
name string used in the `packaging` script and in the `spec` files
(e.g. minio.RELEASE.2017-03-16T21-50-32Z).

After editing the files makes sure to remove any files ending in `~`
or any other editor created backup files.

### Update blob for new package

``` shell
bosh-cli add-blob /tmp/minio ${MINIO_BIN_NAME}/minio
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

### Remove old package(s) and remove reference to older blobs

We remove old package directory as it is not needed for the new release.

``` shell
# Run for each older package in the packages dir (there will be usually be just one)
git rm -f packages/minio.RELEASE.xxxxxxx

# Run for each older blob referenced in config/blobs.yml:
bosh-cli remove-blob minio.RELEASE.2017-03-16T21-50-32Z/minio
```


### Try it out

#### Setting up bosh-lite with boshv2

https://bosh.io/docs/bosh-lite

Follow install steps to get the Bosh director running locally on
VirtualBox.

After performing those steps also run:

``` shell
sudo route add -net 10.244.0.0/16 gw 192.168.50.6
```

to be able to ssh/access Minio inside the VM.

#### Trying/Testing release

Commit changes to git before testing the candidate release:

``` shell
git commit -a -m 'Add files for new release'

# Create release
bosh-cli create-release --force

# Upload above created release to VM:
bosh-cli -e vbox upload-release # upload release

# Upload stem-cell step:
bosh-cli -e vbox upload-stemcell \
    https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent?v=3421.9 \
    --sha1 1396d7877204e630b9e77ae680f492d26607461d

# Deploy to VM step:
bosh-cli -e vbox --deployment minio-singlenode-deploy deploy manifests/manifest-fs.yml

```

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
bosh-cli upload-blobs # This requires you to configure s3 creds in config/private.yml
git commit -a -m 'Add blobs'
```

And create the final release, and commit files it generates. Example
commands:

``` shell
bosh-cli create-release --final

git add .final_builds/jobs/minio-server/index.yml
git add releases/minio/index.yml
git add .final_builds/packages/minio.RELEASE.2017-06-13T19-01-01Z/
git releases/minio/minio-4.yml
git commit -m 'Publish release minio.RELEASE.2017-06-13T19-01-01Z'
```

All done!

## How to sanity check a BOSH release

The aim here is to try and create a release tarball like how a user
may do. This is done by starting from a vanilla ubuntu docker image,
and installing each required component:

1. Create a Dockerfile with (both) bosh cli's installed:

``` dockerfile
FROM ubuntu

RUN apt update && \
    apt-get install -y build-essential ruby ruby-dev libxml2-dev \
    libsqlite3-dev libxslt1-dev libpq-dev libmysqlclient-dev zlib1g-dev git wget

RUN gem install bosh_cli --no-ri --no-rdoc

RUN wget -q -O /usr/local/bin/bosh2 \
    https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-2.0.28-linux-amd64

RUN chmod +x /usr/local/bin/bosh2

```
2. Build docker image, launch container by running a shell, and clone
   the release:

``` shell
docker build -t donatello/minio-bosh-test .
docker run --rm -it donatello/minio-bosh-test /bin/bash
# Inside the container:
git clone https://github.com/minio/minio-boshrelease.git
cd minio-boshrelease
```

3. Try making a tarball from the release:

``` shell
# Replace '3' with whatever is the latest below
bosh create release releases/minio/minio-3.yml --with-tarball
# or with bosh v2:
bosh2 create-release --tarball=/tmp/release.tgz releases/minio/minio-3.yml
```

In either case, the command should work and a tarball should be created.

## OLDER INFO: How to update this release with a newer version of Minio with Bosh V1 (DEPRECATED)

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

The main content for these two files may be taken from the existing
(released) package directory.

Then just replace the name of the package with the correct RELEASE
name string used in the `packaging` script and in the `spec` files
(e.g. minio.RELEASE.2017-03-16T21-50-32Z).

After editing the files makes sure to remove any files ending in `~`
or any other editor created backup files.

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
