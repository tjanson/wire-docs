.. _offline-charts-update:

Updating helm charts in an offline environment
======================================================

How to update parts of the wire-server deployment when your servers cannot reach the internet.

Overview
-----------------

From an internet-connected device:

1. Find the new version
2. Download the new version of an image with ``docker pull <image>:<version>``
3. Save the image with ``docker save <image>:<version> > image.tar``
4. Copy the image to your offline environment

From your offline device/server

5. Load the image with ``docker load < image.tar``
6. (possibly re-tag the image for your local docker registry)
7. Push it into your registry: ``docker push <image>:<version>``
8. Update your charts to use the new version
9. Upgrade your releases with ``helm upgrade``

See the next sections for details.

Prerequisites
---------------

Ensure you have the `docker` command line available::

    apt-get update && apt-get install -y docker.io

How to update wire-server backend components
---------------------------------------------

On an internet-connected device, first find the newest release docker image versions:

On https://github.com/wireapp/wire-server/tags look for the latest tag ``image/2.XYZ.0`` (versions with a ``0`` at the end denote the last production release)

Example, on 2019-08-16, the latest production version is ``2.59.0``.

Next, download the docker images you need, setting that version::

    BACKEND_VERSION=2.59.0
    images=( brig galley gundeck cannon proxy spar cargohold nginz )
    for image in "${images[@]}"; do
        docker pull "quay.io/wire/$image:$BACKEND_VERSION"
        docker save "quay.io/wire/$image:$BACKEND_VERSION" > "$image-$BACKEND_VERSION.tar"
    done

On the registry machine, place all the docker images to a new folder. Then, from that folder::

    for image in *.tar; do
        docker load < "$image"
    done

Ensure "quay.io" points to localhost (or add an entry to /etc/hosts). Then::

    BACKEND_VERSION=2.59.0
    images=( brig galley gundeck cannon proxy spar cargohold nginz )
    for image in "${images[@]}"; do
        docker push "quay.io/wire/$image:$BACKEND_VERSION"
    done

If that succeeds, you may remove the local files::

    rm *.tar

Next, update 



