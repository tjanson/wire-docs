Dependencies on operator's machine
----------------------------------

We provide a container containing a compatible software stack (binaries)
required to run the ansible playbooks, helm charts etc.

If you don't intend to develop *on the tooling itself*, you should use this.


Use the provided Docker container
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

On your machine, you need to have `docker` installed (or any other compatible
runtime really, even though instructions may differ).

See `how to install docker <https://docker.com>`__. Then:

::

   docker pull quay.io/wire/wire-server-deploy-deps:latest

   # cd to a fresh, empty directory and create some sub directories
   cd ...  # you pick a good location!
   mkdir ./admin_work_dir ./dot_kube ./dot_ssh && cd ./admin_work_dir
   # copy ssh key (the easy way, if you want to use your main ssh key pair)
   cp ~/.ssh/id_rsa ../dot_ssh/
   # alternatively: create a key pair exclusively for this installation
   ssh-keygen -t ed25519 -a 100 -f ../dot_ssh/id_ed25519
   ssh-add ../dot_ssh/id_ed25519
   # make sure the server accepts your ssh key for user root
   ssh-copy-id -i ../dot_ssh/id_ed25519.pub root@<server>

   docker run -it --network=host -v $(pwd):/mnt -v $(pwd)/../dot_ssh:/root/.ssh -v $(pwd)/../dot_kube:/root/.kube quay.io/wire/wire-server-deploy-deps:latest
   # inside the container, copy everything to the mounted host file system:
   cp -a /src/* /mnt
   # and make sure the git repos are up to date:
   cd /mnt/wire-server && git pull
   cd /mnt/wire-server-deploy && git pull
   cd /mnt/wire-server-deploy-networkless && git pull

Now exit the docker container.  On subsequent times:

::

   cd admin_work_dir
   docker run -it --network=host -v $(pwd):/mnt -v $(pwd)/../dot_ssh:/root/.ssh -v $(pwd)/../dot_kube:/root/.kube quay.io/wire/wire-server-deploy-deps:latest
   cd wire-server-deploy/ansible
   # do work.

Any changes inside the container under the mount-points listed in the
above command will persist (albeit as user ``root``), everything else
will not, so be careful when creating other files.

To connect to a running container for a second shell:

::

   docker exec -it `docker ps -q --filter="ancestor=quay.io/wire/wire-server-deploy-deps:latest"` /bin/bash


Use Nix and Direnv to provide dependencies
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We use a custom version of `ansible` (with some additional python
dependencies), a specific version of `terraform`, `helm`, `kubectl`, `gnumake`
and some more tooling.

These are all built via `Nix <https://nixos.org>_`, and added to `$PATH` by
`Direnv <https://direnv.net/>_`, so you would need these two things installed
and setup to make use of it - but if you just want to use the tools, see for
the instructions above.


Download external Ansible Roles
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We use git submodules to manage external ansible roles.

We recommend setting `git config --global submodule.recurse true` and `git
config --global fetch.parallel 10`, so they're automatically fetched for
convenience.

If you've already cloned this repo before having enabled this, or don't want to
enable this, run `git submodule update --init --recursive` once (and every time
in the future you update the repo).
