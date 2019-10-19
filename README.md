# Docker Notes
This is a list of docker notes collected while try to figure out stuff about the subject

#### Is docker for desktop for OSx a native docker?
No, it’s running on hypervisor vm. It’s a thin layer, bus still a VM.

#### What should be the user that runs docker deamon?
Docker daemon runs as a root user, since he’s controlling network, cgroups, filesystem, apparmor security profiles. There are experiments on ubuntu-based distros to run rootless, but that has limitations on the list above.

#### If I stop a running container and then start again, do we loose the changes on the fs?
No. Those will be lost only if the container is removed (i.e. -rm)

#### How much size is used when creating a container from an image?
It depends. For each layer uses a copy-on-write is used instead of regular copy. If the container tryies to write into some layer, a new one will be copied and the write will end up there. The most efficient container is the one that minimise that.

#### How to minimise the copy-on-write?
The default driver is aufs, that is copying **only** the file that gets modified. The file is searched from the lastest layer going backward and copied into the writing layer (hence doesn't need to be searched again). If modifying big amount of data:
1. tmp directory with no persistance using tmpfs
```
docker run --tmpfs /tmp -d java-img
```
2. Mount a durable volume 
```
docker run -v tmp-files:/tmp -d java-img
```
3. Start the container with the read-only option **(that is great for security also)**
```
docker run -d --read-only image
```
To inspect what a container is doing at running time is useful to use the command diff
```
docker diff <yourContainerId>
```
Overall it seems like it's going to be an issue only where big files are modified.

#### Why to use JSON format when defining RUN steps?
Not Using JSON syntax in the Dockerfile will create command wrapped into /bin/sh -c "<CMD>". That means that sh doesn't need to be installed to run the command also.
