---
title: Production installation
taxonomy:
    category: docs
---

Mender backend services can be deployed to production using a skeleton provided
in `template` directory
of [integration](https://github.com/mendersoftware/integration?target=_blank)
repository.

## Deployment steps

This is a step by step guide for deploying mender to production. Prerequisites:

- a Linux machine with public IP address
- allocated DNS names (for purpose of the guide, it is assumed that the user
  owns `mender.acme.org` and `s3.acme.org` domains) that resolve to public
  IP of current host
- ports 443 and 9000 are accessible from the outside
- tools:
  - git
  - [docker-compose](https://docs.docker.com/compose/?target=_blank)
  - [Docker engine](https://www.docker.com/products/docker-engine?target=_blank)

The guide will use a publicly available Mender integration repository. Then
proceed with setting up a local branch, preparing deployment configuration from
a template, committing the changes locally. Keeping deployment configuration in
a git repository ensures that the history of changes is tracked and subsequent
updates can be easily applied.

At the end of this guide you will have:

- production configuration in a single `docker-compose` file
- a running mender backend cluster
- persistent service data in stored in Docker volumes:
  - artifact storage
  - MongoDB data
- SSL certificate for API Gateway
- SSL certificate for storage proxy
- a set of keys for generating and validating access tokens

Consult [certificates and keys](../Certificates-and-keys) section for details on
how these are used in the system.

#### Docker compose naming scheme

`docker-compose` implements a particular naming scheme for containers, volumes
and networks that prefixes each object it creates with project name. By default,
the project is named after the directory the main compose file is located in.
Production configuration template provided in the repository explicitly sets
project name to `menderproduction`.

`docker-compose` automatically assigns a
`com.docker.compose.project=menderproduction` label to created containers. The
label can be used when filtering output of commands such as `docker ps`.


### Basic preparation setup

Start off by cloning mender integration repository to a local directory
named `mender-server`:

```
user@local$ git clone https://github.com/mendersoftware/integration mender-server
Cloning into 'deployment'...
remote: Counting objects: 1117, done.
remote: Compressing objects: 100% (11/11), done.
remote: Total 1117 (delta 1), reused 0 (delta 0), pack-reused 1106
Receiving objects: 100% (1117/1117), 233.85 KiB | 411.00 KiB/s, done.
Resolving deltas: 100% (678/678), done.
Checking connectivity... done.
```

Enter the directory:

```
user@local$ cd mender-server
```

Prepare a branch where all deployment related changes will be kept:

```
user@local$ git checkout -b my-production-setup
```

Copy deployment configuration template to a new directory named `production`:

```
user@local$ cp -a template production
```

Enter the directory:

```
user@local$ cd production
user@local$ ls -l
total 12
-rw-rw-r--. 1 user user 4101 01-26 14:06 prod.yml
-rwxrwxr-x. 1 user user  161 01-26 14:06 run
```

Similarly to demo setup, production deployment template is based on
`docker-compose`. The template includes 2 files:

- `prod.yml` - contains deployment specific configuration, builds on top of
  `docker-compose.yml` (located at the root of integration repository)

- `run` - a convenience helper that invokes `docker-compose` by passing required
  compose files; arguments passed to `run` in command line are forwarded
  directly to `docker-compose`

Update paths that are mounted from the root of integration repository:

```
user@local$ sed -i -e 's#/template/#/production/#g' prod.yml
```

At this point all changes should be committed to the repository:

```
user@local$ git add .
user@local$ git commit -m 'production: initial template'
[my-production-setup 556cc2e] production: initial template
 2 files changed, 110 insertions(+)
 create mode 100644 production/prod.yml
 create mode 100755 production/run
```

Assuming that current working directory is still `production`, download
necessary Docker images:

```
user@local$ ./run pull
Pulling mender-mongo-device-adm (mongo:3.4)...
3.4: Pulling from library/mongo
Digest: sha256:aff0c497cff4f116583b99b21775a8844a17bcf5c69f7f3f6028013bf0d6c00c
Status: Image is up to date for mongo:3.4
Pulling mender-mongo-device-auth (mongo:3.4)...
<snip...>
Pulling mender-gui (mendersoftware/gui:latest)...
latest: Pulling from mendersoftware/gui
b7f33cc0b48e: Already exists
31e101b48355: Pull complete
307a023cd01f: Pull complete
2edcf3035646: Pull complete
aeb59eb28f90: Pull complete
<snip...>
Pulling mender-useradm (mendersoftware/useradm:latest)...
latest: Pulling from mendersoftware/useradm
Digest: sha256:3346985a2679b7edd9243363c4b1c291871481f8c6f557ccdc51af58dc6d3a1a
Status: Image is up to date for mendersoftware/useradm:latest
Pulling mender-device-auth (mendersoftware/deviceauth:latest)...
latest: Pulling from mendersoftware/deviceauth
Digest: sha256:ed47310dc9a86cca70d52c520d565ef9814f39421f2faa02d60bbe8a1dbd63c5
Status: Image is up to date for mendersoftware/deviceauth:latest
Pulling mender-api-gateway (mendersoftware/api-gateway:latest)...
latest: Pulling from mendersoftware/api-gateway
Digest: sha256:97243de3da950e754ada73abf8c123001c0a0c8b20344256dc7db4c89c0ecd82
Status: Image is up to date for mendersoftware/api-gateway:latest
```

!! Using `run` helper script may fail when local user has insufficient permissions to reach a local docker daemon. Make sure that Docker installation was completed successfully and the user has sufficient permissions (typically the user must be a member of `docker` group).

### Certificates and keys

Prepare certificates using a helper script `keygen` (replacing `mender.acme.org`
and `s3.acme.org` with your DNS names):

```
user@local$ CERT_API_CN=mender.acme.org CERT_STORAGE_CN=s3.acme.org ../keygen
Generating a 256 bit EC private key
writing new private key to 'private.key'
-----
Generating a 256 bit EC private key
writing new private key to 'private.key'
-----
.................................................................................................................++
......................++
writing RSA key
.....++
........................................++
writing RSA key
All keys and certificates have been generated in directory keys-generated.
Please include them in your docker compose and device builds.
For more information please see https://docs.mender.io/Administration/Certificates-and-keys.
```

Your local tree should look like this:
```
user@local$ tree .
.
├── keys-generated
│   ├── certs
│   │   ├── api-gateway
│   │   │   ├── cert.crt
│   │   │   └── private.key
│   │   └── storage-proxy
│   │       ├── cert.crt
│   │       └── private.key
│   └── keys
│       ├── deviceauth
│       │   └── private.key
│       └── useradm
│           └── private.key
├── prod.yml
└── run
```

Production template file `prod.yml` is already configured to load keys and
certificates from locations created by `keygen` script. Should you wish to use a
different set of certificates or keys
consult [relevant documentation](../Certificates-and-keys).

Add and commit generated keys and certificates:

```
user@local$ git add keys-generated
user@local$ git commit -m 'production: adding generated keys and certificates'
[my-production-setup fd8a397] production: adding generated keys and certificates
 6 files changed, 108 insertions(+)
 create mode 100644 production/keys-generated/certs/api-gateway/cert.crt
 create mode 100644 production/keys-generated/certs/api-gateway/private.key
 create mode 100644 production/keys-generated/certs/storage-proxy/cert.crt
 create mode 100644 production/keys-generated/certs/storage-proxy/private.key
 create mode 100644 production/keys-generated/keys/deviceauth/private.key
 create mode 100644 production/keys-generated/keys/useradm/private.key
```

!! Instructions presented above are a simplification. Make sure to keep all keys in a safe location. Use encryption if needed, make sure to decrypt the keys before using them in containers, see [encrypting keys](#encrypting-keys) section for examples.

API Gateway and storage proxy certificates generated here need to be made
available to mender client.
Consult [building for production](../../Artifacts/Building-for-production) section
in artifacts documentation.

!! Only certificates need to be made available to devices or end users. Private keys should never be shared.

#### Encrypting keys (optional)

A simplest method of encrypting keys is by
using [GNU Privacy Guard](https://www.gnupg.org/) tool. Instead of committing
plain-text keys to the repository, we will `tar` and encrypt the whole
`keys-generated` directory structure, then commit it to the repository as a
single binary blob.

Encryption of whole directory structure can be performed like this:

```
user@local$ tar c keys-generated |  gpg --output keys-generated.tar.gpg --symmetric
Enter passphrase:
user@local$ rm -rf keys-generated
user@local$ ls -l
total 20
-rw-rw-r--. 1 mborzecki mborzecki 5094 01-30 11:24 keys-generated.tar.gpg
-rw-rw-r--. 1 mborzecki mborzecki 5519 01-30 11:08 prod.yml
-rwxrwxr-x. 1 mborzecki mborzecki  173 01-27 14:16 run
```

Keys need to be decrypted before bringing the whole environment up:

```
user@local$ gpg --decrypt keys-generated.tar.gpg | tar xvf -
gpg: AES encrypted data
Enter passphrase:
```

When using encryption, commit `keys-generated.tar.gpg` instead of the whole
`keys-generated` directory structure to the repository, like this:

```
user@local$ git add keys-generated.tar.gpg
user@local$ git commit -m 'production: adding generated keys and certificates'
[my-production-setup 237af44] production: adding generated keys and certificates
 1 file changed, 111 insertions(+)
 create mode 100644 production/keys-generated.tar.gpg
```

### Persistent storage

Persistent storage of backend services' data is implemented using
named [Docker volumes](https://docs.docker.com/engine/tutorials/dockervolumes/).
The template is configured to mount the following volumes:

- `mender-artifacts` - artifact objects storage
- `mender-deployments-db` - deployments service database data
- `mender-useradm-db` - user administration service database data
- `mender-deviceauth-db` - device authentication service database data
- `mender-deviceadm-db` - device admission service database data
- `mender-inventory-db` - inventory service database data

Each of these volumes need to be created manually with the following commands:

```
user@local$ docker volume create --name=mender-artifacts
mender-artifacts
user@local$ docker volume create --name=mender-deployments-db
mender-deployments-db
user@local$ docker volume create --name=mender-useradm-db
mender-useradm-db
user@local$ docker volume create --name=mender-inventory-db
mender-inventory-db
user@local$ docker volume create --name=mender-deviceadm-db
mender-deviceadm-db
user@local$ docker volume create --name=mender-deviceauth-db
mender-deviceauth-db
```

Since we are using local driver for volumes, each volume is based on a host
local directory. It is possible access files in this directory once a local path
is known. To find the local path for a specific volume run the following
command:

```
user@local$ docker volume inspect --format '{{.Mountpoint}}' mender-artifacts
/var/lib/docker/volumes/mender-artifacts/_data
```

The path depends on local docker configuration and may vary between installations.

### Final configuration

The deployment configuration still needs some final touches. Open `prod.yml` in
favorite editor.

#### Storage proxy

Locate `storage-proxy` service and add `s3.acme.org` (or your DNS name)
under `networks.mender.aliases` key. The entry should look like this:

```
    ...
    storage-proxy:
        networks:
            mender:
                aliases:
                    - s3.acme.org
    ...

```

You can also change values for `DOWNLOAD_SPEED` and `MAX_CONNECTIONS`.
See [bandwidth control](../Bandwidth) section for details on meaning of these
settings.

#### Minio

Locate `minio` service. Keys `MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY` control
credentials for uploading artifacts into object store. Since Minio is a S3 API
compatible service, these settings correspond to Amazon's AWS Access Key ID and
Secret Access Key respectively.

Set `MINIO_ACCESS_KEY` to `mender-deployments`. `MINIO_SECRET_KEY` should be set
to a value that cannot be easily guessed. We recommend using `pwgen` utility
for generating secret. In order to generate a 32 characters long secret run the
following command:

```
user@local$ pwgen 32 1
ahshagheeD1ooPaeT8lut0Shaezeipoo
```

Updated entry should look like this:

```
    ...
    minio:
        environment:
            # access key
            MINIO_ACCESS_KEY: mender-deployments
            # secret
            MINIO_SECRET_KEY: ahshagheeD1ooPaeT8lut0Shaezeipoo
    ...

```

#### Deployments service

Locate `mender-deployments` service. The deployments service will upload
artifact objects to `mionio` storage via `storage-proxy`
(see [administration overview](../Overview) for details). For this reason,
access credentials `DEPLOYMENTS_AWS_AUTH_KEY` and `DEPLOYMENTS_AWS_AUTH_SECRET`
need to be updated and `DEPLOYMENTS_AWS_URI` must point to `s3.acme.org` (or
your DNS name)

!! The address used in `DEPLOYMENTS_AWS_URI` must be exactly the same as one that will be used by devices. Deployments service generats signed URLs for accessing artifact storage. A different host name or port will result in signature verification failure and download attempts will be rejected.

Set `DEPLOYMENTS_AWS_AUTH_KEY` and `DEPLOYMENTS_AWS_AUTH_SECRET` to the values
of `MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY` respectively. Set
`DEPLOYMENTS_AWS_URI` to point to `https://s3.acme.org:9000`.

Updated entry should look like this:

```
    ...
    mender-deployments:
        ...
        environment:
            DEPLOYMENTS_AWS_AUTH_KEY: mender-deployments
            DEPLOYMENTS_AWS_AUTH_SECRET: ahshagheeD1ooPaeT8lut0Shaezeipoo
            DEPLOYMENTS_AWS_URI: https://s3.acme.org:9000
    ...
```

#### Saving final configuration

Once all configuration is complete, commit all changes to repository:

```
user@local$ git add prod.yml
user@local$ git commit -m 'production: final configuration'
```

At this point your commit history should look as follows:

```
user@local$ git log --oneline master..HEAD
7a4de3c production: final configuration
41273f7 production: adding generated keys and certificates
5ad6528 production: initial template
```

### Bring it all up

Bring all services up in detached mode:

```
user@local$ ./run up -d
Creating network "menderproduction_mender" with the default driver
Creating menderproduction_mender-mongo-device-auth_1
Creating menderproduction_mender-mongo-inventory_1
Creating menderproduction_mender-gui_1
Creating menderproduction_minio_1
Creating menderproduction_mender-mongo-deployments_1
Creating menderproduction_mender-mongo-useradm_1
Creating menderproduction_mender-mongo-device-adm_1
Creating menderproduction_mender-device-auth_1
Creating menderproduction_mender-inventory_1
Creating menderproduction_storage-proxy_1
Creating menderproduction_mender-useradm_1
Creating menderproduction_mender-deployments_1
Creating menderproduction_mender-device-adm_1
Creating menderproduction_mender-api-gateway_1
```

!!! Services, networks and volumes have `menderproduction` prefix, see a note about [docker-compose naming scheme](#docker-compose-naming-scheme) for details. When using `docker ..` commands, complete container name must be provided (ex. `menderproduction_mender-deployments_1`).

Verify that services are working:

```
user@local$ ./run ps
                   Name                                  Command               State           Ports
-------------------------------------------------------------------------------------------------------------
menderproduction_mender-api-gateway_1         /entrypoint.sh                   Up      0.0.0.0:443->443/tcp
menderproduction_mender-deployments_1         /entrypoint.sh                   Up      8080/tcp
menderproduction_mender-device-adm_1          /usr/bin/deviceadm -config ...   Up      8080/tcp
menderproduction_mender-device-auth_1         /usr/bin/deviceauth -confi ...   Up      8080/tcp
menderproduction_mender-gui_1                 /entrypoint.sh                   Up
menderproduction_mender-inventory_1           /usr/bin/inventory -config ...   Up      8080/tcp
menderproduction_mender-mongo-deployments_1   /entrypoint.sh mongod            Up      27017/tcp
menderproduction_mender-mongo-device-adm_1    /entrypoint.sh mongod            Up      27017/tcp
menderproduction_mender-mongo-device-auth_1   /entrypoint.sh mongod            Up      27017/tcp
menderproduction_mender-mongo-inventory_1     /entrypoint.sh mongod            Up      27017/tcp
menderproduction_mender-mongo-useradm_1       /entrypoint.sh mongod            Up      27017/tcp
menderproduction_mender-useradm_1             /usr/bin/useradm -config / ...   Up      8080/tcp
menderproduction_minio_1                      minio server /export             Up      9000/tcp
menderproduction_storage-proxy_1              /usr/local/openresty/bin/o ...   Up      0.0.0.0:9000->9000/tcp
```

#### Verification

Since this is a brand new installation it should be possible to request initial
login token through the API:

```
user@local$ curl -X POST  -D - --cacert keys-generated/certs/api-gateway/cert.crt https://mender.acme.org:443/api/management/0.1/useradm/auth/login
HTTP/2.0 200
server:openresty/1.11.2.2
date:Fri, 27 Jan 2017 15:44:53 GMT
content-type:application/json; charset=utf-8
content-length:734
vary:Accept-Encoding
x-men-requestid:9a50755b-a246-4128-bfac-e8d20c81a7dc
strict-transport-security:max-age=63072000; includeSubdomains; preload
x-frame-options:DENY
x-content-type-options:nosniff

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJle....
```

!!! Note: if your DNS name does not resolve the public IP address of current host, you may need to add appropriate entries to `/etc/hosts`

At this point you should be able to access [https://mender.acme.org](https://mender.acme.org) using your
browser.