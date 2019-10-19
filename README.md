# Docker Notes
This is a list of docker notes collected while try to figure out stuff about the subject

#### Is docker for desktop for OSx a native docker?
No, it’s running on hypervisor vm. It’s a thin layer, bus still a VM.

#### What should be the user that runs docker deamon?
Docker daemon runs as a root user, since he’s controlling network, cgroups, filesystem, apparmor security profiles. There are experiments on ubuntu-based distros to run rootless, but that has limitations on the list above.

#### If I stop a running container and then start again, do we loose the changes on the fs?
No. Those will be lost only if the container is removed (i.e. -rm)
