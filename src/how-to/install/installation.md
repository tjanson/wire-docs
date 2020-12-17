# How to install wire

We have a pipeline in  `wire-server-deploy` producing container images, static
binaries, ansible playbooks, debian package sources and everything required to
install Wire.

On your machine (we call this the "admin host"), you need to have `docker`
installed (or any other compatible container runtime really, even though
instructions may need to be modified). See [how to install
docker](https://docker.com) for instructions.

Create a fresh workspace to download the artifacts:

```
cd ...  # you pick a good location!
```

Obtain the latest artifacts from wire-server-deploy. Once
https://github.com/wireapp/wire-server-deploy/pull/363 is finished, these will
be in the "Releases" tab. Until then, go to the PRs "Checks" tab, and look for
the URL of the "assets" tarball linked from CI.

Extract the above listed artifacts into your workspace:

```
wget https://s3-eu-west-1.amazonaws.com/public.wire.com/artifacts/wire-server-deploy-static-<HASH>.tgz
tar xvzf wire-server-deploy-static-<HASH>.tgz
```
Where the HASH above is the hash of your deployment artifact, given to you by Wire, or acquired by looking at the above build job.

You should now have a directory named `assets` containing the offline deploy artifacts.


Copy over ssh keys, or generate a key pair exclusively for this installation:

```
mkdir -p ./assets/dot_ssh
cp ~/.ssh/id_rsa ./assets/dot_ssh/
```

vs.

```
ssh-add ./assets/dot_ssh/id_ed25519
```

Ensure the nodes you are deploying to, as well as any bastion host between you and them accept this ssh key for the user you are performing the install as:

```
# make sure the server accepts your ssh key for user root
# TODO: replace with ansible oneliner making use of the inventory
ssh-copy-id -i ./assets/dot_ssh/id_ed25519.pub <username>@<server>
```

Now `cd ./assets`. All the following operations should happen from the `assets` directory.


There's also a docker image containing the tooling inside this repo.

If you don't intend to develop *on wire-server-deploy itself*, you should load
this and register an alias:
```
docker load -i ./containers-other/pxllp20fzzbafc179v4yjqijx9zw0254-docker-image-wire-server-deploy.tar.gz
alias d="docker run -it --network=host -v $PWD:/wire-server-deploy quay.io/wire/wire-server-deploy:pxllp20fzzbafc179v4yjqijx9zw0254"
alias dapi="d ansible-playbook -i ansible/inventory/offline/hosts.ini --private-key ./dot_ssh/id_ed25519"
```

The following artifacts are provided:

 - `containers-other/wire-server-deploy-*.tar`
   A container image containing ansible, helm, and other tools and their
   dependencies in versions verified to be compatible with the current wire
   stack. Published to `quay.io/wire/wire-server-deploy` as well, but shipped
   in the artifacts tarball for convenience.
 - `ansible`
   These contain all the ansible playbooks the rest of the guide refers to, as
   well as an example inventory, which should be configured according to the
   environment this is installed into.
 - `binaries`
   This contains static binaries, both used during the kubespray-based
   kubernetes bootstrapping, as well as to provide some binaries that are
   installed during other ansible playbook runs.
 - `charts`
   The charts themselves, as tarballs. We don't use an external helm
   repository, every helm chart dependency is resolved already.
 - `containers-system`
   These are the container images needed to bootstrap kubernetes itself
   (currently using kubespray)
 - `containers-helm`
   These are the container images our charts (and charts we depend on) refer to.
   Also come as tarballs, and are seeded like the system containers.
 - *containers-other*
   These are other container images, not deployed inside k8s. Currently, only
   contains restund.
 - `debs`
   This acts as a self-contained dump of all packages required to install
   kubespray, as well as all other packages that are installed by ansible
   playbooks on nodes that don't run kubernetes.
   There's an ansible playbook copying these assets to an "assethost", starting
   a little webserver there serving it, and configuring all nodes to use it as
   a package repo.
 - `values`
   Contains helm chart values and secrets. Needs to be tweaked to the
   environment.

Provide a `ansible/inventory/offline/hosts.ini` configured to your environment.
Copy over `hosts.ini.example`  to `hosts.ini`, and edit it, following the instructions in that file.

In case that is part of your inventory, don't forget to set a value for `restund_zrest_secret`:
```
tr -dc 'a-zA-Z0-9' < /dev/urandom | fold -w 42 | head -n 1
```

Also, take a look at `ansible/inventory/offline/group_vars/all/offline.yml` to
see if it matches your expectations.


Copy over binaries and debs, serves assets from the asset host, and configure
other hosts to fetch debs from it:


```
dapi ansible/setup-offline-sources.yml
```

Run kubespray until docker is installed and runs:

```
dapi ansible/kubernetes.yml --tags bastion,bootstrap-os,preinstall,container-engine
```

Now; run the restund playbook until docker is installed:
```
dapi ansible/restund.yml --tags docker
```

With docker being installed on all nodes that need it, seed all container images:

```
dapi ansible/seed-offline-docker.yml
```

Run the rest of kubespray:

```
dapi ansible/kubernetes.yml --skip-tags bootstrap-os,preinstall,container-engine
```

Copy over the kubeconfig from one master node:
```
d ansible -i ./ansible/inventory/offline/hosts.ini "kube-master[0]" -m fetch -a  "src=/root/.kube/config dest=ansible/kubeconfig flat=yes"
```

Ensure the cluster comes up healthy. The container also contains kubectl, so check the node status:

```
d kubectl get nodes -owide
```
They should all report ready.

Now, deploy all other services which don't run in kubernetes:

```
dapi ansible/restund.yml
dapi ansible/cassandra.yml
dapi ansible/elasticsearch.yml
dapi ansible/minio.yml
```
TODO: add commands to verify things are good

Afterwards, run the following playbook to create helm values
```
dapi ansible/helm_external.yml
```

Write other values by copying over TODO

Use Helm to apply the various charts in order to install the Wire platform

TODO: update https://docs.wire.com/how-to/install/helm-prod.html to not add the iptables hack

```
d helm install cassandra-external ./charts/cassandra-external --values ./values/cassandra-external/values.yaml
d helm install elasticsearch-external ./charts/elasticsearch-external --values ./values/elasticsearch-external/values.yaml
d helm install minio-external ./charts/minio-external --values ./values/minio-external/values.yaml
d helm install fake-aws ./charts/fake-aws --values ./values/fake-aws/values.yaml
d helm install demo-smtp ./charts/demo-smtp --values ./values/demo-smtp/values.yaml
d helm install redis-ephemeral ./charts/nginx-ingress-controller
d helm install reaper ./charts/reaper

d helm install wire-server ./charts/wire-server --timeout=15m0s --values ./values/wire-server/values.yaml --values ./values/wire-server/secrets.yaml

d helm install nginx-ingress-controller ./charts/nginx-ingress-controller
d helm install nginx-ingress-services ./charts/nginx-ingress-services --values ./values/nginx-ingress-services/values.yaml  --values ./values/nginx-ingress-services/secrets.yaml
```

### Ephemeral Helm deployment

Before applying the HElm charts, adjust your own copy of `wire-server/values.yaml` so that every `replicaCount` is set to `1`. Futhermore, change every occurrence of `[cassandra,elasticsearch]-external` to `[cassandra,elasticsearch]-ehemeral` and `minio-external` to `fake-aws-s3`.

```
d helm install databases-ephemeral ./charts/databases-ephemeral
d helm install fake-aws ./charts/fake-aws
d helm install reaper ./charts/reaper
d helm install demo-smtp ./charts/demo-smtp

d helm install wire-server ./charts/wire-server --timeout=15m0s --values ./values/wire-server/values.yaml --values ./values/wire-server/secrets.yaml

d helm install nginx-ingress-controller ./charts/nginx-ingress-controller
d helm install nginx-ingress-services ./charts/nginx-ingress-services --values ./values/nginx-ingress-services/values.yaml  --values ./values/nginx-ingress-services/secrets.yaml
```

## Deploying legalhold
Legalhold is an optional component that can be installed on-prem for
compliance. The following steps explain how to enable it.

Legalhold is enabled per team in the team settings. While enabling it, you need
to provide the public key of the legalhold service, and a service token.

The Legalhold ingress endpoint needs to be reachable from the backend service.

Legalhold consists of a service and a PostgreSQL database. We provide a helm
chart that installs it in a cluster (which does not need to be the same as the
one running wire):

### Ensure you have a storage provisioner installed
Our helm charts provide a PostgreSQL database. For this to work, a storage
provisioner needs to be installed in your cluster, and a default storage class
needs to be configured.

Please pick a storage provisioner matching to your environment. See
https://kubernetes.io/docs/concepts/storage/ for details. If you are in a cloud
environment there is a high chance a storage provisioner is already
preconfigured. You can check this with `kubectl get storageclass`.

We also provide a off-the-shelf storage provider that uses the local disks of
the kubernetes nodes.  Keep in mind that this has several downsides. There is
no redundancy to protect against disk failures, no automated backups, and your
workloads will be bound to the node that keeps the data volume and can not be
rescheduled.  Disaster recovery requires manual intervention.

If you have no more suitable storage provisioner configured, and feel
comfortable with the local path provisioner, install it with the following
command:
```
d helm install local-path-provisioner ./charts/local-path-provisioner --set storageClass.defaultClass=true
```

### Install the legalhold helm chart
In `values/wire-server/values.yaml` make sure that `tags.legalhold` is set to `true`.
Also, make sure that `FEATURE_ENABLE_LEGAL_HOLD: "true"` is set.  Then set
```
legalhold:
  host: "legalhold.example.com"
  wireApiHost: "https://nginz-https.example.com"
```
to the domain name of your legalhold service and to the endpoint of the `nginz`
component you configured in the previous steps.

Note that for legalhold to function, it is important that both `host` and
`wireApiHost` are reachable from _within_ the cluster. Make sure you have
configured your loadbalancer to support this.


Now, get a certificate for the domain you set in `legalhold.host` and add this
to `values/wire-server/secrets.yaml`.  These are the `legalhold.tlsKey` and
`legalhold.tlsCert` options.

Finally, in `values/wire-server/secrets.yaml`, `legalhold.serviceToken`  needs to
be set to a random alpha-numerical string.


Now the legalhold helm chart can be installed with the following command:

```
d helm upgrade wire-server ./charts/wire-server -f ./values/wire-server/values.yaml -f ./values/wire-server/secrets.yaml
```


### Configuring legalhold in Team-settings

Extract the public key out of the legalhold certificate:
```
openssl x509 -pubkey -noout -in cert.pem  > pubkey.pem
```

Go to your team-settings page at `https://teams.<your-domain>`. In the Settings page you can configure
the legalhold service. Paste in the public key and the service token (that you configured under `legalhold.serviceToken`) to configure the legalhold service.


TODO:

 - run other playbooks for other pets.
 - add zauth tool to our docker container

### Installing sftd

For full docs with details and explanations please see https://github.com/wireapp/wire-server-deploy/blob/d7a089c1563089d9842aa0e6be4a99f6340985f2/charts/sftd/README.md

First, make sure you have a certificate for `sftd.<yourdomain>`. This could be the same wildcard or SAN certificate
you used at previous steps.

Next, make sure that in your `hosts.ini` you have annotated all the nodes that are able to run sftd workloads correctly.
It is important for sftd to be aware of its own external IP.
E.g.:
```
kubenode3 node_labels="wire.com/role=sftd" node_annotations="{'wire.com/external-ip': 'XXXX'}"
```

If these weren't already set; you should rerun :
```
dapi ansible/kubernetes.yml --skip-tags bootstrap-os,preinstall,container-engine
```

Now, deploy the chart. Note the `nodeSelector` matching up with your ansible inventory!!

```
d helm upgrade sftd ./charts/sftd \
  --set 'nodeSelector.wire\.com/role=sftd' \
  --set host=sftd.example.com \
  --set allowOrigin=https://webapp.example.com \
  --set-file tls.crt=/path/to/tls.crt \
  --set-file tls.key=/path/to/tls.key
```


