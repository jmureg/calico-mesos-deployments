<!--- master only -->
> ![warning](images/warning.png) This document applies to the HEAD of the calico-mesos-deployments source tree.
>
> View the calico-mesos-deployments documentation for the latest release [here](https://github.com/projectcalico/calico-mesos-deployments/blob/0.26.0%2B1/README.md).
<!--- else
> You are viewing the calico-mesos-deployments documentation for release **release**.
<!--- end of master only -->

# Calico with Marathon and the Mesos Docker Containerizer

The following instructions outline the steps required to enable and manage 
Calico networked containers launched from Marathon using the Mesos Docker
containerizer.  This guide covers:
-  Setting up Docker multi-host networking
-  Installing and configuring Calico
-  Creating a Docker network and managing network policy
-  Launching a container

## Set up Docker multi-host networking

To utilize the Docker libnetwork plugin feature of Docker, it is necessary to 
run with a Docker version 1.9 or higher, and to configure Docker to use a
cluster store.  We'll set up an etcd datastore which will be used by Docker and
Calico.

It is important to install the following steps for etcd configuration, calico/mesos installation and address pool configuration on each host involved in the multi-host cluster. 

#### Install etcd on each host machine

Set up an etcd cluster.  For initial testing we’d recommend starting with a
single-node cluster on one of your master nodes.

For simplicity there is a Docker image you can use to spin up a single etcd
instance.  On the selected server, run the following

```
IP=<IP Address>
sudo docker pull quay.io/coreos/etcd:v2.2.0
sudo mkdir -p /var/etcd
sudo docker run --detach --name etcd --net host -v /var/etcd:/data quay.io/coreos/etcd:v2.2.0 \
     --advertise-client-urls "http://${IP}:2379,http://${IP}:4001" \
     --listen-client-urls "http://0.0.0.0:2379,http://0.0.0.0:4001" \
     --data-dir /data
```

replacing `<IP Address>` with the server IP address.

#### Set up Docker daemon to use etcd as a cluster store on each host machine

On each agent:
-  Stop your docker service.
-  If you don’t already have at least Docker 1.9 then upgrade/install Docker
   1.9 or higher.
-  Modify the docker daemon parameters to include:
   `--cluster-store=etcd://<ip address>:4001` e.g.
   `echo "DOCKER_OPTS=--cluster-store=etcd://<etcd ip address>:2379" | tee -a /etc/default/docker`
-  Start your docker service.

## Install and set up Calico on each host machine

#### Install Calico on each agent

To install Calico on each agent, run the following commands on each agent:

```
wget https://github.com/projectcalico/calico-containers/releases/download/v0.16.1/calicoctl
chmod a+x calicoctl 
sudo ETCD_AUTHORITY=<IP Address>:4001 ./calicoctl node --libnetwork
```

substituting `<IP Address>` with the IP address of your etcd server.  

> Note: The ETCD_AUTHORITY environment variable needs to be set when running
> any calicoctl commands.  We recommend setting the environment variable in 
> your login scripts (or similar), and placing the calicoctl binary in your
> system path.  The remaining instructions assume you have done this.

> Note: Docker and Calico allow you to control network policy for your entire 
> cluster by issuing commands from anywhere in the cluster.  For example, if 
> you want to manage network policy by issuing commands on a master node rather
> than on an agent, you will also need to follow the above steps to "Set up 
> Docker to use etcd as a cluster store" and "Install Calico on each Agent" on
> that master node.

#### Configure your IP pools

By default, Calico creates an IP pool with CIDR 192.168.0.0/16.  The Calico 
IPAM module will assign IP addresses to your containers from that range of IP 
addresses.

If you want to allocate from different CIDRs you can use the 
`calicoctl pool remove` and `calicoctl pool add` commands to modify the 
available pools.  For example, to change the pool from 192.168.0.0/16 to be 
172.168.0.0/16 run the following to remove the original pool and create a new
one: 

```
calicoctl pool remove 192.168.0.0/16
calicoctl pool add 172.168.0.0/16
```

You can view the configured pools using the `calicoctl pool show` command.  e.g.

```
$ ./calicoctl pool show
+----------------+---------+
|   IPv4 CIDR    | Options |
+----------------+---------+
| 172.168.0.0/16 |         |
+----------------+---------+
+--------------------------+---------+
|        IPv6 CIDR         | Options |
+--------------------------+---------+
| fd80:24e2:f998:72d6::/64 |         |
+--------------------------+---------+
```

## Creating a Docker network and managing network policy

With Calico, a Docker network represents a logical set of rules that define the 
allowed traffic in and out of containers assigned to that network. The rules
are encapsulated in a Calico "profile". Each Docker network is assigned its 
own Calico profile.

#### Create a Docker network on one machine

To create a Docker network using Calico, run the `docker network create`
command specifying "calico" as both the network and IPAM driver.  For example,
to create a network called `data-tier` run the following command:

```
docker network create --driver calico --ipam-driver calico data-tier 
```

This command returns a "network ID".  You can also run 
`docker network inspect <network name>` to manually obtain the network ID from
the network name.
  
For example, here we create a network and then inspect it.  You can see the 
ID returned by the creation is also returned by network inspection.

```
$ docker network create --driver calico --ipam-driver calico data-tier
c36ddf3b
$ docker network inspect -f '{{.ID}}' data-tier
c36ddf3b
```

>  This example uses shortened network IDs for clarity.

#### Verify a Docker network on another machine
The network installed on one machine is transferred on other machines in the multi-host environment via Calico. 

For example,
to verify a new instantiated network called `data-tier` on other hosts in the Calico network run the following command:

```
docker network ls 
```
The resulting list should contain the new Calico network next to the three standard docker networks plus other networks eventually.

```
# docker network ls
NETWORK ID          NAME                DRIVER
<new network id>    data-tier           calico
...
<network id>        bridge              bridge
<network id>        none                null
<network id>        host                host
```

#### Networks and Calico Profiles

By default, when a network is created the corresponding Calico profile contains
rules which allow full communication between containers on the same network.

We can see this by using `calicoctl profile <profile> rule show` to display the
rules in the profile associated with the network.  The name of the profile is 
the same as the network ID.

In the example above:

```
$ calicoctl profile c36ddf3b rule show
Inbound rules:
   1 allow from tag c36ddf3b
Outbound rules:
   1 allow
```

Each profile has an identically named tag.  You can think of the tag as a
way to identify the containers using the particular profile, or within the 
corresponding network.

The default rules state that the container allows all outbound traffic, but
only accepts inbound traffic from containers with the same tag.  In other words,
only containers within the same network can communicate with each other.

> Note that we are planning a future release which will allow network names to
> be used rather than network IDs.  This will simplify policy management as it
> will not be necessary to map between network names and IDs.

#### Configuring network policy

Calico has a rich set of policy rules that can be leveraged.  Rules can be
created to allow and disallow packets based on a variety of parameters such
as source and destination CIDR, port and tag.

The `calicoctl profile <profile> rule add` and `calicoctl profile <profile> rule remove`
commands can be used to add and remove rules in the profile.
  
As an example, suppose the data-tier network represents a tier of MySQL
databases which should allow TCP traffic to port 3306 from containers
in the application tier.

For example:

```
$ docker network create --driver calico --ipam-driver calico app-tier
005bc278
$ ./calicoctl profile c36ddf3b rule add inbound allow tcp from tag 005bc278 to ports 3306
```

These commands do the following:
-  create a second network called "app-tier" which represents the containers
   in the application tier.  This has a network ID of "005bc278" which directly 
   corresponds to the name of the Calico profile assigned to the container and
   the "tag" that identifies all containers in this network.
-  add a new rule to the data-tier that we created earlier that allows inbound
   TCP traffic from the "005bc278" tag (i.e. from containers in the app-tier)
   to port 3306.
   
For more details on the syntax of the rules, run `calicoctl profile --help` to
display the valid profile commands.

For more details take a look at our 
[Advanced Network Policy guide](https://github.com/projectcalico/calico-containers/blob/master/docs/AdvancedNetworkPolicy.md).


## Launching a container

With your networks configured, it is trivial to launch a Docker container 
through Mesos using the standard Marathon UI and API.

#### Launching a container through the UI

Through the UI, launch a Docker task as normal.  Select an arbitrary network
(Bridge or Host), and then provide the following additional parameter 
(under the Docker options)

```
Key = net
Value = <network name>
```

Where `<network name>` is the name of the network, for example "data-tier".


#### Launching a container with a JSON blob

To launch a container using the Marathon API with a JSON blob, simply include
the net parameter in the request.  For example:


```
{
    "id":"/calico-apps",
    "apps": [
        {
            "id": "unified-docker-task",
            "cmd": "ip addr && sleep 300",
            "cpus": 0.1,
            "mem": 64.0,
            "container": {
                "type": "DOCKER",
                "docker": {
                    "image": "busybox",
                    "parameters": [
                        {"key": "net", "value": "data-tier"}
                    ]
                }
            }
        }
    ]
}
```
Use the following command to send the example.json above to the Marathon REST API.
```
curl -i -H 'Content-Type: application/json' -d@example.json <marathon ip>:8080/v2/apps
```
Finally verify the ip address and network assigned to the container via the command
```
docker inspect unified-docker-task  
```

[![Analytics](https://ga-beacon.appspot.com/UA-52125893-3/calico-mesos-deployments/docs/CalicoWithTheDockerContainerizer.md?pixel)](https://github.com/igrigorik/ga-beacon)