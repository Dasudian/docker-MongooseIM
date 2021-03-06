# docker-MongooseIM:1.5
Build MongooseIM image based on [dasudian/docker-erlang:17.5](https://hub.docker.com/r/dasudian/docker-erlang/) image.

## VERSION
1.5.1

## Exposed ports

* 4369 - epmd - erlang port mapper daemon
* 5222 - xmpp port
* 5280 - rest endpoint
* 5269 - port for the s2s communication
* 9100 - port for distributed erlang

## Volumes

* `/data/log` - mongooseim logs directory (logs are also available via `docker logs CONTAINER_NAME`
* `/data/mnesia` - mnesia directory. It contains Mnesia.mongoose@hostname directory

## Usage

### Start single node

To start an interactive session and map the XMPP port to the host:

`$ docker run -i -t -p 5222:5222  dasudian/docker-mongooseim:1.5 live`

To start MongooseIM in the background (logs are available via `docker logs CONTAINER_NAME`).

`$ docker run -d -t  -p 5222:5222 dasudian/docker-mongooseim:1.5`

Note the -t option also for the background case, without it
the `mongooseim debug` shell won't be able to attach to running MongooseIM.

To attach the menstioned debug shell to already running node use:

`$ docker exec -it CONTAINER_NAME mongooseimctl debug`

where CONTAINER_NAME can be obtained via `docker ps` or by explicitly specifing
it using the `--name` option during `docker run`.

Actually with the `docker exec` command you can do any mongoosectl command for example
we can  register a new user:

`$ docker exec -it CONTAINER_NAME mongooseimctl mongooseimctl register pawel localhost test`

### Create cluster of the MongooseIM nodes using docker links

To be able to create a MongooseIM cluster you need to set the hostname for
the containers. Based on the hostname the `start.sh` script will start a node with
the following sname `mongooseim@HOSTNAME`.

Then, to create a cluster, you first need to start an initial node:

`$ docker run -d -t -h mim1 --name mim1 dasudian/docker-mongooseim:1.5 `

sname of initial node will be `mongooseim@mim1`. To start the next node and make
it part of the existing cluster you need to add the `--link` option and set the
`CLUSTER_WITH` env variable:

`$ docker run -d -t -h mim2 --name mim2 --link mim1:mim1 -e CLUSTER_WITH=mim1 dasudian/docker-mongooseim:1.5 `

In this example we started a new node with sname: `mognooseim@mim2`
which will join the cluster to which mim1 belongs.  Note that link name
and CLUSTER_WITH *must* match the hostname of initial node.
It wouldn't work if set in the following way:

`--link mim1:my_initial_node` and `-e CLUSTER_WITH=my_initial_node_name`.

In this case the IP address will be resolved correctly but it won't match
sname of the CLUSTER_WITH host - as a result nodes won't be able to cluster.

### Create cluster of the MongooseIM nodes - multihost setup

Clustering setup presented in the previous section only makes sense if all containers are on the same on the same 
node. Otherwise, we won't be able to use links mechanism without extra work(ambassador pattern etc.). The situation is not as bad as it seems to be, the only thing we need to change is switch from `--link` to proper `--add-host` options which adds required entries to the `/etc/hosts` file,  Additionally we need to make sure, that we have exposed and forwarded all required ports, which are:
* 4369 - for erlang port mapper daemon
* 9100 - for actual cluster connections

It was not an issue in the previous case, because by default all ports in the "docker network" are open, so containers are able to talk to each other using them. 
See the Mongoose documentation for more details about clustering: http://mongooseim.readthedocs.org/en/latest/operation-and-maintenance/Cluster-configuration-and-node-management/.

## Change configuration

The best way to replace the default config file is to create a new image that uses
`dasudian/docker-mongooseim:1.5` as a base image and just replace the config file
with the ADD command. Moreover this approach allows you to update any MongooseIM file.

For example new Dockerfile might look like this (assuming that ejabberd.cfg is present in the
current directory:

```
FROM dasudian/docker-mongooseim:1.5

ADD ./ejabberd.cfg  /opt/mongooseim/rel/mongooseim/etc/ejabberd.cfg
```

There is also a way that doesn't require creating, so  we should be
able to mount the given file with `-v option`:

```
docker run -v `pwd`/ejabberd.cfg:/opt/mongooseim/rel/mongooseim/etc/ejabberd.cfg dasudian/docker-mongooseim:1.5`
```

To get the default config you need to run a container and then use the `docker cp`
command to copy it from container or just take it from https://github.com/esl/MongooseIM/blob/master/rel/files/ejabberd.cfg (remember to select correct branch!)

## Persistent data

There are two volumes that one might want to persist or share between image upgrades:

* `/data/log`
* `/data/mnesia`

To bind a volume you can use -v option:

```
docker run -td -h mim1 -v `pwd`/mnesia:/data/mnesia -v `pwd`/log:/data/log --name mim1 dasudian/docker-mongooseim:1.5
```

It will start a MongooseIM instance and it will bind the mnesia dir and log directory
to the `mnesia` and `log` dir in the current working dir.

The mnesia directory will look like this:

```
ls -l mnesia
drwxr-xr-x  16 pawel.pikula  staff  544 May 25 16:05 Mnesia.mongooseim@mim1
drwxr-xr-x  16 pawel.pikula  staff  544 May 25 16:05 Mnesia.mongooseim@mim2
```

As a result it is possible to use one volume for all instances, but in case of
the log directory the log files are saved directly in that directory:

```
ls -la log
-rw-r--r--   1 pawel.pikula  staff    0 May 25 16:02 crash.log
-rw-r--r--   1 pawel.pikula  staff  472 May 25 16:02 ejabberd.log
```

Of course we can use data containers instead of our local filesystem. Look at
https://docs.docker.com/userguide/dockervolumes/ for more details.
