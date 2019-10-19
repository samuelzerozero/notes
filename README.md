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
1. tmp directory with no persistance using tmpfs
'''
docker run --tmpfs /tmp -d java-img
'''
2. Create a thin layer with only the creation of that empty directory (?)
3. Mount a durable volume 
'''
docker run -v tmp-files:/tmp -d java-img
'''
