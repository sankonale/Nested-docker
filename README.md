# Docker-in-Docker

This recipe lets you run Docker within Docker.

There is only one requirement: your Docker version should support the
`--privileged` flag.


## A word of warning

If you came here because you would like to run a testing system like
Jenkins in a container, and want that container to spin up more containers,


Docker is Docker : bad
One is about LSM (Linux Security Modules) like AppArmor and SELinux: when starting a container, the “inner Docker” might try to apply security profiles that will conflict or confuse the “outer Docker.” This was actually the hardest problem to solve when trying to merge the original implementation of the -privileged flag. My changes worked (and all tests would pass) on my Debian machine and Ubuntu test VMs, but it would crash and burn on Michael Crosby’s machine (which was Fedora if I remember well). I can’t remember the exact cause of the issue, but it might have been because Mike is a wise person who runs with SELINUX=enforce (I was using AppArmor) and my changes didn’t take SELinux profiles into account.

The second issue is linked to storage drivers. When you run Docker in Docker, the outer Docker runs on top of a normal filesystem (EXT4, BTRFS, what have you) but the inner Docker runs on top of a copy-on-write system (AUFS, BTRFS, Device Mapper, etc., depending on what the outer Docker is setup to use). There are many combinations that won’t work. For instance, you cannot run AUFS on top of AUFS. If you run BTRFS on top of BTRFS, it should work at first, but once you have nested subvolumes, removing the parent subvolume will fail. Device Mapper is not namespaced, so if multiple instances of Docker use it on the same machine, they will all be able to see (and affect) each other’s image and container backing devices. 


The Docker daemon was explicitly designed to have exclusive access to /var/lib/docker. Nothing else should touch, poke, or tickle any of the Docker files hidden there.

Why is that? It’s one of the hard learned lessons from the dotCloud days. The dotCloud container engine worked by having multiple processes accessing /var/lib/dotcloud simultaneously. Clever tricks like atomic file replacement (instead of in-place editing), peppering the code with advisory and mandatory locking, and other experiments with safe-ish systems like SQLite and BDB only got us so far; and when we refactored our container engine (which eventually became Docker) one of the big design decisions was to gather all the container operations under a single daemon and be done with all that concurrent access nonsense.

(Don’t get me wrong: it’s totally possible to do something nice and reliable and fast involving multiple processes and state-of-the-art concurrency management; but we think that it’s simpler, as well as easier to write and to maintain, to go with the single actor model of Docker.)

This means that if you share your /var/lib/docker directory between multiple Docker instances, you’re gonna have a bad time. Of course, it might work, especially during early testing. “Look ma, I can docker run ubuntu!” But try to do something more involved (pull the same image from two different instances…) and watch the world burn.

This means that if your CI system does builds and rebuilds, each time you’ll restart your Docker-in-Docker container, you might be nuking its cache. That’s really not cool.




Docker-in-Docker: the good

The goal was to help the core team to work faster on Docker development. Before Docker-in-Docker, the typical development cycle was:

hackity hack
build
stop the currently running Docker daemon
run the new Docker daemon
test
repeat
And if you wanted to a nice, reproducible build (i.e. in a container), it was a bit more convoluted:

hackity hack
make sure that a workable version of Docker is running
build new Docker with the old Docker
stop Docker daemon
run the new Docker daemon
test
stop the new Docker daemon
repeat
With the advent of Docker-in-Docker, this was simplified to:

hackity hack
build+run in one step
repeat




The socket solution
Let’s take a step back here. Do you really want Docker-in-Docker? Or do you just want to be able to run Docker (specifically: build, run, sometimes push containers and images) from your CI system, while this CI system itself is in a container?

I’m going to bet that most people want the latter. All you want is a solution so that your CI system like Jenkins can start containers.

And the simplest way is to just expose the Docker socket to your CI container, by bind-mounting it with the -v flag.

Simply put, when you start your CI container (Jenkins or other), instead of hacking something together with Docker-in-Docker, start it with:

docker run -v /var/run/docker.sock:/var/run/docker.sock ...
Now this container will have access to the Docker socket, and will therefore be able to start containers. Except that instead of starting “child” containers, it will start “sibling” containers.

Try it out, using the docker official image (which contains the Docker binary):

docker run -v /var/run/docker.sock:/var/run/docker.sock \
           -ti docker
This looks like Docker-in-Docker, feels like Docker-in-Docker, but it’s not Docker-in-Docker: when this container will create more containers, those containers will be created in the top-level Docker. You will not experience nesting side effects, and the build cache will be shared across multiple invocations.

⚠️ Former versions of this post advised to bind-mount the docker binary from the host to the container. This is not reliable anymore, because the Docker Engine is no longer distributed as (almost) static libraries.

If you want to use e.g. Docker from your Jenkins CI system, you have multiple options:

installing the Docker CLI using your base image’s packaging system (i.e. if your image is based on Debian, use .deb packages),
using the Docker API.






## Another word of warning

This work is now obsolete, thanks to the [combined](
https://github.com/docker/docker/pull/15596) [efforts](
https://github.com/docker-library/official-images/blob/master/library/docker)


If you want to run Docker-in-Docker today, all you need to do is:

```bash
docker run --privileged -d docker:dind
```

... And that's it; you get Docker running in Docker, thanks to
the official Docker image, in its "Docker-in-Docker" flavor.
You can then connect to this Docker instance by starting
another Docker container linking to the first one (which is
a pretty amazing thing to do).

For more details about the `docker:dind` official image,
explanations about how to use it, customize it to use
specific storage drivers, and other tidbits of useful
knowledge, check [its documentation on the Docker Hub](
https://hub.docker.com/_/docker/).


## If you read past this paragraph ...

... Then you're probably an archaeologist, a masochist, or both.

Seriously, though: the information below is here mostly
for historical value, or if you want to understand how those
things work under the hood.

You've been warned!


## Quickstart

Build the image:
```bash
docker build -t dind .
```

Run Docker-in-Docker and get a shell where you can play, and docker daemon logs
to stdout:
```bash
docker run --privileged -t -i dind
```

Run Docker-in-Docker and get a shell where you can play, but docker daemon logs
into `/var/log/docker.log`:
```bash
docker run --privileged -t -i -e LOG=file dind
```

Run Docker-in-Docker and expose the inside Docker to the outside world:
```bash
docker run --privileged -d -p 4444 -e PORT=4444 dind
```

Note: when started with the `PORT` environment variable, the image will just
the Docker daemon and expose it over said port. When started *without* the
`PORT` environment variable, the image will run the Docker daemon in the
background and execute a shell for you to play.

### Daemon configuration

You can use the `DOCKER_DAEMON_ARGS` environment variable to configure the
docker daemon with any extra options:
```bash
docker run --privileged -d -e DOCKER_DAEMON_ARGS="-D" dind
```

## It didn't work!

If you get a weird permission message, check the output of `dmesg`: it could
be caused by AppArmor. In that case, try again, adding an extra flag to
kick AppArmor out of the equation:

```bash
docker run --privileged --lxc-conf="lxc.aa_profile=unconfined" -t -i dind
```

If you get the warning:

````
WARNING: the 'devices' cgroup should be in its own hierarchy.
````

When starting up dind, you can get around this by shutting down docker and running:

````
# /etc/init.d/lxc stop
# umount /sys/fs/cgroup/
# mount -t cgroup devices 1 /sys/fs/cgroup
````

If the unmount fails, you can find out the proper mount-point with:

````
$ cat /proc/mounts | grep cgroup
````

## How It Works

The main trick is to have the `--privileged` flag. Then, there are a few things
to care about:

- cgroups pseudo-filesystems have to be mounted, and they have to be mounted
  with the same hierarchies than the parent environment; this is done by a
  wrapper script, which is setup to run by default;
- `/var/lib/docker` cannot be on AUFS, so we make it a volume.

That's it.


## Important Warning About Disk Usage

Since AUFS cannot use an AUFS mount as a branch, it means that we have to
use a volume. Therefore, all inner Docker data (images, containers, etc.)
will be in the volume. Remember: volumes are not cleaned up when you
`docker rm`, so if you wonder where did your disk space go after nesting
10 Dockers within each other, look no further :-)


## Which Version Of Docker Does It Run?

Outside: it will use your installed version.

Inside: the Dockerfile will retrieve the latest `docker` binary from
https://get.docker.io/; so if you want to include *your* own `docker`
build, you will have to edit it. If you want to always use your local
version, you could change the `ADD` line to be e.g.:

    ADD /usr/bin/docker /usr/local/bin/docker


## Can I Run Docker-in-Docker-in-Docker?

Yes. Note, however, that there seems to be a weird FD leakage issue.
To work around it, the `wrapdocker` script carefully closes all the
file descriptors inherited from the parent Docker and `lxc-start`
(except stdio). I'm mentioning this in case you were relying on
those inherited file descriptors, or if you're trying to repeat
the experiment at home.


a wrapper script that uses dind to nest Docker to arbitrary depth.

Also, when you will be exiting a nested Docker, this will happen:

```bash
root@975423921ac5:/# exit
root@6b2ae8bf2f10:/# exit
root@419a67dfdf27:/# exit
root@bc9f450caf22:/# exit
jpetazzo@tarrasque:~/Work/DOTCLOUD/dind$
```

