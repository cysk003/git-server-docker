Git Server Docker
=================
[![Test Build Status][b1]][bl]
[![Docker Image Size][b2]][bl]
[![Docker Pulls][b3]][bl]

Simple Docker image containing a Git server accessible via SSH.

- [Basic Usage](#basic-usage)
- [Security Enhancements](#security-enhancements)
  * [Use a Custom Password](#use-a-custom-password)
  * [Use SSH public keys](#use-ssh-public-keys)
  * [Custom SSH Host Keys](#custom-ssh-host-keys)
  * [Disable Git User Interactive Login](#disable-git-user-interactive-login)
- [Advanced Configuration](#advanced-configuration)
  * [Enable Git URLs Without Absolute Path](#enable-git-urls-without-absolute-path)
  * [Set Git User UID / GID](#set-git-user-uid--gid)
  * [Setup logging](#setup-logging)
  * [Visualization and HTTP support](#visualization-and-http-support)
- [Variants](#variants)
- [Tagging Scheme](#tagging-scheme)
- [License](#license)
- [Credits](#credits)


Basic Usage
-----------
Simply run:

```
docker run -v git-repositories:/srv/git -p 2222:22 rockstorm/git-server
```

Your server should be accessible on port 2222 via:

```
git clone ssh://git@localhost:2222/srv/git/your-repo.git
```

The default password for the git user is `12345`.

### Create a New Repository

Log into the server through SSH. Note the git user is constrained to
only a handful of commands, enough to list, create, delete, or rename
repositories, or change repository descriptions:

```
ssh git@localhost -p 2222
```

Create and initialise a repository at `/srv/git`:
```
mkdir /srv/git/your-repo.git
git-init --bare /srv/git/your-repo.git
```

A [sample `docker-compose.yml`](examples/docker-compose.yml) is provided with
all available options to use with `docker-compose`.


Security Enhancements
---------------------
### Use a Custom Password

Every container generated by this image has the same default password
set for the git user. You can set your own password using the
`GIT_PASSWORD` variable.

```shell
docker run --detach \
  --name git-server \
  --volume git-repositories:/srv/git \
  --env GIT_PASSWORD=your-password \
  --publish 2222:22 \
  rockstorm/git-server
```

To avoid specifying your password on the command line or on your compose file,
you can load it from a file using the `GIT_PASSWORD_FILE` variable. This
variable must be set to the file within the container to load the password
from.

```shell
docker run --detach \
  --name git-server \
  --volume git-repositories:/srv/git \
  --volume /path/to/password/file:/run/secrets/git_password:ro \
  --env GIT_PASSWORD_FILE=/run/secrets/git_password \
  --publish 2222:22 \
  rockstorm/git-server
```

Or making use of a `docker-compose.yml` file:

```yaml
services:
  git-server:
    ...
    environment:
      GIT_PASSWORD_FILE: /run/secrets/git_password
    volumes:
    - /path/to/password/file:/run/secrets/git_password:ro
```


### Use SSH public keys

More secure than using passwords is using SSH public key authentication
instead[^1]. Simply mount the file with the SSH authentication keys for the
users that will be allowed to interact with the server. These are set in the
`docker-compose.yml` file as:

```yaml
services:
  git-server:
    ...
    volumes:
      - /path/to/authorized_keys:/home/git/.ssh/authorized_keys
```


#### Create an authorized keys file

SSH key generation for your client machine to connect to the server is
detailed in depth on [Git's book Chapter 4.3][1].

Then, simply copy the contents of all allowed clients' `id_*.pub` into a file
and mount it as detailed above.

[1]: https://git-scm.com/book/en/v2/Git-on-the-Server-Generating-Your-SSH-Public-Key


#### Use SSH public keys stored online

You can use a set of keys stored somewhere online using the
`SSH_AUTHORIZED_KEYS_URL` variable like:

```shell
docker run --detach \
  --name git-server \
  --volume git-repositories:/srv/git \
  --env SSH_AUTHORIZED_KEYS_URL=https://github.com/username.keys \
  --publish 2222:22 \
  rockstorm/git-server
```


#### Disable password log in

By default, the git user is allowed to log in using either SSH public key
_or_ a password. To disable clear text passwords completely and only allow
connections via SSH public keys, set to 'publickey' the `SSH_AUTH_METHODS`
variable:

```yaml
services:
  git-server:
    ...
    environment:
      SSH_AUTH_METHODS: "publickey"
```

The `SSH_AUTH_METHODS` variable effectively sets the 'AuthenticationMethods'
variable within the SSH server configuration file. Therefore, it can be set to
any value allowed by it. See [OpenSSH server documentation][2] for more
information. Example values for `SSH_AUTH_METHODS`:

| Value                | Authentication method(s) allowed        |
|----------------------|-----------------------------------------|
| 'publickey'          | SSH public key only                     |
| 'publickey password' | SSH public key _or_ password            |
| 'publickey,password' | SSH public key _followed by_ a password |

Of course, you can also mount your custom configuration file for the SSH
server at `/etc/ssh/sshd_config` for better fine tuning. The default
configuration is provided at [`examples/sshd_config`](examples/sshd_config).

```yaml
services:
  git-server:
    ...
    volumes:
      - examples/sshd_config:/etc/ssh/sshd_config:ro
```

> **NOTE**: If you mount a custom SSH server configuration file
> (e.g. `/etc/ssh/sshd_config`), using `SSH_AUTH_METHODS` is discouraged since
> it would try to rewrite such file.

[2]: https://man.openbsd.org/sshd_config#AuthenticationMethods


### Custom SSH Host Keys

The default host keys are generated during image build and are the
same for every container which uses this image. This is a security
risk and therefore the use of a custom set of keys is highly
recommended. This will also ensure keys are persistent if the image is
upgraded.

To enable custom SSH host keys set the `SSH_HOST_KEYS_PATH` variable
to a location such as `/tmp/host-keys` and mount a folder with your
custom keys on the server. The setup process will replace the default
keys with these ones. This would look like the following on your
docker-compose.yml file:

```yaml
services:
  git-server:
    ...
    environment:
      SSH_HOST_KEYS_PATH: /tmp/host-keys
    volumes:
      - /path/to/host-keys:/tmp/host-keys:ro
```


### Disable Git User Interactive Login

To disable the interactive SSH login for the git user and limit it to
only git clone, push and pull actions, mount a file onto
`/home/git/git-shell-commands/no-interactive-login`. This file must be
executable. When the git user attempts to login, this file is run and
the interactive shell is aborted. This is set in the
docker-compose.yml file as:

```yaml
services:
  git-server:
    ...
    volumes:
      - /executable/file:/home/git/git-shell-commands/no-interactive-login
```


Advanced Configuration
----------------------
### Enable Git URLs Without Absolute Path

By default, git URLs to your repositories will be in the form of:

```
git clone ssh://git@example.com:2222/srv/git/project/repository.git
```

By setting the environment variable `REPOSITORIES_HOME_LINK` to
e.g. `/srv/git/project` a link will be created into the git user home
directory so that your git URLs don't require the repository absolute
path[^2]:

```
git clone ssh://git@example.com:2222/project/repository.git
```

To configure this on your docker-compose.yml file:

```yaml
services:
  git-server:
    ...
    environment:
      REPOSITORIES_HOME_LINK: /srv/git/project
```

To avoid specifying ports on git URLs you can configure your client
machine by adding the following to your `~/.ssh/config` file:

```
Host my-server
    HostName example.com
    User git
    Port 2222
```

This way your git URLs would look like:

```
git clone my-server:project/repository.git
```


### Set Git User UID / GID

The variables `GIT_USER_UID` and `GIT_USER_GID` allow you to customise
the UID and GID of the git user inside the container. This could be
useful if the host is administered by a non-root user and you would
like the git user to have the same UID (This would allow not having to
restart the container to reset file permissions on files created by a
host user). If `GIT_USER_UID` is defined but `GIT_USER_GID` isn't, the
latter is assumed to be equal to the first. To configure this on your
docker-compose.yml file:

```yaml
services:
  git-server:
    ...
    environment:
      GIT_USER_UID: 1001
```


### Setup logging

This image will produce no logs by default. To output logging to
stderr configure your docker-compose.yml like:

```yaml
services:
  git-server:
    ...
    command: ["/usr/sbin/sshd", "-D", "-e"]
```

If you add a custom command, be sure to include `/usr/sbin/sshd -D`
for the git server to stay in the foreground, otherwise your container
will stop immediately after starting.


### Visualization and HTTP support

To have unauthenticated read access to your repositories through HTTP
and visualize them you can run a webserver along this image. One
example of such a webserver is [this GitWeb image][3]. You just need
to mount the folder/volume with your repositories on both containers
at the relevant locations.

```yaml
services:
  git-server:
    image: rockstorm/git-server
    ...
    volumes:
      - ./path/to/repos:/srv/git

  gitweb:
    image: rockstorm/gitweb
    ...
    volumes:
      - ./path/to/repos:/srv/git:ro
```

[3]: https://github.com/rockstorm101/gitweb-docker


Variants
--------
All images are based on the latest stable image of [Alpine Linux][4].

### `git-server:<git-version>`

Default image. It contains just git and SSH.

### `git-server:<git-version>-docker`

This image used to include the Docker CLI. This variant is now
deprecated in favor of running a CI/CD service separate from this
image. For example, see [Bash CI Server][5].

[4]: https://hub.docker.com/_/alpine
[5]: https://github.com/rockstorm101/bash-ci-server


Tagging Scheme
--------------
 - **'X.Y-bZ'**: Immutable tag. Points to a specific image build and will
   not be reused.

 - **'X.Y'**: Stable tag for specific Git major and minor versions. It
   follows the latest build for Git version X.Y and therefore changes
   on every patch change (i.e. 1.2.3 to 1.2.4), on every change on
   OpenSSH and every change on the base Alpine image.

 - **'latest'**: This tag follows the very latest build regardless any
   major/minor versions.


License
-------
View [license information][6] for the software contained in this
image.

As with all Docker images, these likely also contain other software
which may be under other licenses (such as Bash, etc from the base
distribution, along with any direct or indirect dependencies of the
primary software being contained).

As for any pre-built image usage, it is the image user's
responsibility to ensure that any use of this image complies with any
relevant licenses for all software contained within.

[6]: https://github.com/rockstorm101/git-server-docker/blob/master/LICENSE

Credits
-------
Re-implementation heavily based on [jkarlosb's][7] but coded from
scratch.

Table of contents on this README was generated with [markdown-toc][8].

[7]: https://github.com/jkarlosb/git-server-docker
[8]: http://ecotrust-canada.github.io/markdown-toc


[^1]: Detailed reasoning behind this claim at [SO][9].
[^2]: How it works and more information are discussed at [SO][10].

[9]: https://superuser.com/a/303388
[10]: https://stackoverflow.com/a/39841058


[b1]: https://img.shields.io/github/actions/workflow/status/rockstorm101/git-server-docker/test-build.yml?branch=master
[b2]: https://img.shields.io/docker/image-size/rockstorm/git-server/latest?logo=docker
[b3]: https://img.shields.io/docker/pulls/rockstorm/git-server
[bl]: https://hub.docker.com/r/rockstorm/git-server
