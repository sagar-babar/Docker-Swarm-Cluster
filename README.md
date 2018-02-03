##High Available Docker Swarm cluster with ETCD  cluster using Docker machine and Docker Compose 

  ####Tested with Mac OS 10.11.5 (EI Capitan)

###Dependency

 * [Docker-machine](https://docs.docker.com/engine/installation/mac/) >= 0.7.0
 * [Docker-client](https://docs.docker.com/engine/installation/mac/)  >= 1.11.1
 * [Docker-compose](https://docs.docker.com/compose/install/)         >= 1.7.0
 * Digital Ocean credentials 

### Provision Two docker machine 

Here we are provisioning two virtual machines into digital ocean cloud and then manually we will set up etcd and docker swarm.

* Provison a VM on cloud with name **backend-swarm-master** execute the follwing cmd from your laptop.

<pre>

docker-machine create --driver digitalocean --digitalocean-access-token=&lt;Your_Token&gt; --digitalocean-region fra1 --digitalocean-size 1gb  --digitalocean-private-networking --digitalocean-userdata userdata.sh backend-swarm-master

</pre>

* Provison a VM on cloud with name **backend-swarm-node** execute the follwing cmd from your laptop.

<pre>

docker-machine create --driver digitalocean --digitalocean-access-token=&lt;Your_Token&gt; --digitalocean-region fra1 --digitalocean-size 1gb  --digitalocean-private-networking --digitalocean-userdata userdata.sh backend-swarm-node

</pre>


###A  key-value store 

Docker compose version 2  requires a overlay network to communicate among dokcer container on different host.
An overlay network requires a key-value store. The key-value store holds information about the network state which includes discovery, networks, endpoints, IP addresses, and more.

Here we are going to use [ETCD](https://github.com/coreos/etcd/) as key-value store.

**Variables:**

etcd_node1_IP:46.101.233.204
etcd_node2_IP:46.101.201.90

**Bootstrap etcd cluster**

We will create 2 nodes in **backend-swarm-master** and 1 node in **backend-swarm-node**

**Prerequisite:**

* switch to sudo user(e.g root) download etcd, unzip it.
* move binaries to /usr/local/bin using below command
* we will create cluster using private network, change etcd_node_ip value to private IP.
* check etcd is working or not

<pre>
curl -L  https://github.com/coreos/etcd/releases/download/v2.3.3/etcd-v2.3.3-linux-amd64.tar.gz -o etcd-v2.3.3-linux-amd64.tar.gz
tar xzvf etcd-v2.3.3-linux-amd64.tar.gz

mv etcd-v2.3.3-linux-amd64/etcd etcd-v2.3.3-linux-amd64/etcdctl  /usr/local/bin/

etcdctl --version
</pre>

Once it is working fine go ahead and create etcd cluster
create a directory /etc/lib/etcd/ for stroing etcd data. first time we need to keep cluster-state=new later modify it to existing

Node1:(Execute on backend-swarm-master)
<pre>
nohup etcd --name etcd1 --data-dir /var/lib/etcd/etcd1.etcd --listen-client-urls http://0.0.0.0:5001 --advertise-client-urls http://${etcd_node1_IP}:5001 --listen-peer-urls http://0.0.0.0:8001 --initial-advertise-peer-urls http://${etcd_node1_IP}:8001 --initial-cluster-token etcd-cluster-1  --initial-cluster etcd1=http://${etcd_node1_IP}:8001,etcd2=http://${etcd_node1_IP}:8002,etcd3=http://${etcd_node2_IP}:8001 --initial-cluster-state=existing >>  /dev/null 2>&1 & 
</pre>

Node2:(Execute on backend-swarm-master)
<pre>
 nohup etcd --name etcd2 --data-dir /var/lib/etcd/etcd2.etcd --listen-client-urls http://0.0.0.0:5002 --advertise-client-urls http://${etcd_node1_IP}:5002 --listen-peer-urls http://0.0.0.0:8002 --initial-advertise-peer-urls http://${etcd_node1_IP}:8002 --initial-cluster-token etcd-cluster-1  --initial-cluster etcd1=http://${etcd_node1_IP}:8001,etcd2=http://${etcd_node1_IP}:8002,etcd3=http://${etcd_node2_IP}:8001 --initial-cluster-state=existing >>  /dev/null 2>&1 &
</pre>

Node3:(Execute on backend-swarm-node)
<pre>
nohup etcd --name etcd3 --data-dir /var/lib/etcd/etcd3.etcd --listen-client-urls http://0.0.0.0:5001 --advertise-client-urls http://${etcd_node2_IP}:5001 --listen-peer-urls http://0.0.0.0:8001 --initial-advertise-peer-urls http://${etcd_node2_IP}:8001 --initial-cluster-token etcd-cluster-1  --initial-cluster etcd1=http://${etcd_node1_IP}:8001,etcd2=http://${etcd_node1_IP}:8002,etcd3=http://${etcd_node2_IP}:8001 --initial-cluster-state=existing >>  /dev/null 2>&1 &
</pre>

check etcd bootstrapped correctly with below reference

<pre>
	curl -L http://ETCD_ENDPOINT/v2/keys/ 

	If you see response something like below then your ETCD cluster is correctly bootstrapped.

	{"action":"get","node":{"dir":true}}

</pre>

###Create a HA Swarm cluster

**Prerequisite:**

Add cluster store details into docker.service (/etc/systemd/system/docker.service) on both machine.

stop docker service and make changes in above file

service docker start

<pre>
e.g:
--cluster-store=etcd://${etcd_node1_IP}:5001,${etcd_node1_IP}:5002,${etcd_node2_IP}:5001 --cluster-advertise=eth0:2376

Reference Details:
[Service]
ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2376 -H unix:///var/run/docker.sock --storage-driver aufs --tlsverify --tlscacert /etc/docker/ca.pem --tlscert /etc/docker/server.pem --tlskey /etc/docker/server-key.pem --label provider=digitalocean --cluster-store=etcd://46.101.233.204:5001,46.101.233.204:5002,46.101.201.90:5001 --cluster-advertise=eth0:2376
MountFlags=slave
</pre>

After modifing run below commands
systemctl daemon-reload
service docker start

* To create the Swarm **backend-swarm-master** execute the follwing cmd from your laptop into swarm-master machine

<pre>
docker pull swarm:latest

docker run -d -p 3376:3376 -v /etc/docker:/etc/docker swarm:latest  manage --tlsverify --tlscacert=/etc/docker/ca.pem --tlscert=/etc/docker/server.pem --tlskey=/etc/docker/server-key.pem --tlskey=/etc/docker/server-key.pem -H tcp://0.0.0.0:3376 --strategy spread --advertise ${etcd_node1_IP}:3376 "etcd://${etcd_node1_IP}:5001,${etcd_node1_IP}:5002,${etcd_node2_IP}:5001"
</pre>

<pre>
docker run -d swarm:latest join --advertise ${etcd_node1_IP}:2376 "etcd://${etcd_node1_IP}:5001,${etcd_node1_IP}:5002,${etcd_node2_IP}:5001"
</pre>

* To create the Swarm **backend-swarm-node** execute the follwing cmd from your laptop into swarm-node machine

<pre>
docker pull swarm:latest

docker run -d swarm:latest join --advertise ${etcd_node2_IP}:2376 "etcd://${etcd_node1_IP}:5001,${etcd_node1_IP}:5002,${etcd_node2_IP}:5001"
</pre>

  Now the Swarm cluster has been created. To view the cluster status and get the current primary of cluster.

<pre>
eval `docker-machine env backend-swarm-master` && export $(env | grep 'DOCKER_HOST' | sed "s/\(.*\):\(.*\)/\1:3376/g")
docker info 
</pre>

It should show something like this. ( Note the node names are different here in the screenshot )

For this case your nodes will be 

  * backend-swarm-master
  * backend-swarm-node 

<img src="images/swarm-info.png" />


###Bring up your stack on swarm cluster using Blue-Green deployment

To deploy your application using blue green stratagy, 

 * Update files in your web, sidekiq or scheduler folder if required ( If your deploying fo the first time skip this step ).
 * Run the script with your infra paramerts like "swarm primary name" , etcd cluster endpoint and and swarm cluster name 

<pre> 
 	./deploy.sh -a 'APPLICATIONS_TO_DEPLOY' -e 'YOUR_ETCD_ENDPOINT' -p 'YOUR_SWARM_PRIMARY' 

 E.g 

 	./deploy.sh -e '139.59.140.125:5001' -a 'web sidekiq scheduler'  -p 'backend-swarm-master'

  

  You can directly set those values in scripts if you dont want to pass it throught command line.

	primary="${primary:-YOUR_SWARM_PRIMARY}"
	ETCD="${ETCD:-YOUR_ETCD_ENDPOINT}"

 E.g.

	primary="${primary:-backend-swarm-master}"
	ETCD="${ETCD:-139.59.140.125:5001}"

 Then run the "deploy.sh"

	./deploy.sh -a 'web sidekiq'
	

 To deploy a single application either api or administration afterwards, just run the 

   ./deploy.sh -a 'web'

 OR 

   ./deploy.sh -a 'sidekiq'

</pre>

This will create the application stack in blue zone if this is the first time or it will create the stack in the zone which is not currently serving the actual traffic.

Once the script execution is completed, you will have your app running in blue zone.





	











### Running Migration files

access the bash of the web app container and run :
```
rake namespace:taskname
```
example :
  If we need to run the import teams task in the import_parse_data.rake where the namespace is : parse_import
  the command will be ``` rake parseimport:import_teams ```
