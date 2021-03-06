Administration Guide
=========

### Post-Install First Steps

Post-installation first steps

1. Change the password for the islet user (default: demo)
```
passwd demo
```

2. Create a Docker image for your training environment (see Adding Training Environments)
```
cat <<EOF > Dockerfile
# Build image for C programming
FROM      ubuntu
MAINTAINER Jon Schipp <jonschipp@gmail.com>

RUN adduser --disabled-password --gecos "" demo
RUN apt-get update -qq
RUN apt-get install -y build-essential
RUN apt-get install -y git vim emacs nano tcpdump gawk rsyslog
RUN apt-get install -y --no-install-recommends man-db
EOF

docker build -t gcc-training - < Dockerfile
```
3. Create an ISLET configuration file for the Docker image (see Adding Training Environments)
```
make template > /etc/islet/gcc.conf
vim /etc/islet/islet/gcc.conf
# Set ENVIRONMENT variable to name of training environment (e.g. gcc-training)
# Set VIRTUSER variable to name of user in docker image that the student will become (e.g. demo)
```

### Configuration

In order of variable precedence
* Global configuration file: */etc/islet/islet.conf*
* Module configuration files: */etc/islet/modules/$MODULE.conf*
* Environments configuration files: */etc/islet/environments/$ENVIRONMENT.conf*
* Plugin configuration files: */etc/islet/plugins/$PLUGIN.conf*

Settings in the global configuration file are overridden by those in module
config file, and finally those in the environment config file. This means that
you set global defaults and can get more granular with specific environments.

For each training environment you want available for use by ISLET, create a
a environment configuration file with the necessary settings and place it in the
*/etc/islet/environments* directory. These images will be selectable from the ISLET
menu after authentication via SSH. Plugin configurations are also selectable from the menu
and are intended to add functionality.

Common Tasks:

* Add another training account for ISLET (used to remotely access e.g. ssh)

```
useradd --create-home --shell /opt/islet/bin/islet_shell training
echo "training:training" | chpasswd
gpasswd -a training docker # User needs to be in the docker grouip
gpasswd -a training islet  # User needs to be in the islet group
```

* Change the password of a container user (Not a system account).

```
    $ PASS=$(echo "newpassword" | sha1sum | sed 's/ .*//)
	$ sqlite3 /var/tmp/islet.db "UPDATE accounts SET password='$PASS' WHERE user='jon';"
	$ sqlite3 /var/tmp/islet.db "SELECT password FROM accounts WHERE user='jon';"
	aaaaaaa2a4817e5c9a56db45d41ed876e823fcf|1413533585

```

* Configure container and user lifetime (e.g. conference duration)

  1. Specify the number of days for user account and container lifetime in:

```
        $ grep ^DAYS /etc/islet/islet.conf
        DAYS=3 # Length of the event in days
```

  Removal scripts are cron jobs that are scheduled in /etc/cron.d/islet

* Allocate more or less resources for containers, and control other container settings.
  These changes will take effect for each newly created container.
  - System and use case dependent

```
    $ grep -A 5 "Container config /etc/islet/environments/brolive.conf
	# Container Configuration
	VIRTUSER="demo"                                         # Account used when container is entered (Must exist in container!)
	CPUSHARES="1024"                                        # Proportion of cpu share allocation per container
	MEMORY="256m"                                           # Amount of memory allocated to each container
	SWAP="100m"                                             # Amount of swap memory allocated to each container
	HOSTNAME="bro"	                                      	# Set hostname in container. PS1 will end up as $VIRTUSER@$HOSTNAME:~$ in shell
	NETWORK="none"                                          # Disable networking by default: none; Enable networking: bridge
	DNS="127.0.0.1"                                         # Use loopback when networking is disabled to prevent error messages from resolver
	MOUNT="-v /exercises:/exercises:ro"			# Mount point(s), sep. by -v: /src:/dst:attributes, ro = readonly (avoid rw)
	OPTIONS="--cap-add=NET_RAW --cap-add=NET_ADMIN"		# Apply any other options you want passed to Docker run here
	MOTD="Training materials are in /exercises"             # Message of the day is displayed before container launch and reattachment
```

* Adding, removing, or modifying exercises

  1. Make changes in /exercises on the host's filesystem

  *  Changes are immediately available for new and existing containers

# Case Study

The precursor to ISLET was used to aid the instructers in teaching the Bro platform at BroCon14.

Workflow:
1. Install ISLET and dependencies
2. Build Docker image containing Bro (docker pull broplatform/brolive)
3. Write a ISLET config file for the Bro image
4. Set a banner in the ISLET config file for light branding (logo)
5. Hand out the demo account credentials to your students so they can SSH in
6. Instruct them on the software

Here's a brief demonstration:

```
        $ ssh demo@live.bro.org

        Welcome to Bro Live!
        ====================

            -----------
          /             \
         |  (   (0)   )  |
         |            // |
          \     <====// /
            -----------

        A place to try out Bro.

        Are you a new or existing user? [new/existing]: new

        A temporary account will be created so that you can resume your session. Account is valid for the length of the event.

        Choose a username [a-zA-Z0-9]: jon
        Your username is jon
        Choose a password:
        Verify your password:
        Your account will expire on Fri 29 Aug 2014 07:40:11 PM UTC

        Enjoy yourself!
        Training materials are located in /exercises.
        e.g. $ bro -r /exercises/beginner/http.pcap

        demo@bro:~$ pwd
        /home/demo
        demo@bro:~$ which bro
        /usr/local/bro/bin/bro
```

