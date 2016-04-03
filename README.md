Inspired from https://github.com/docker/swarm-microservice-demo-v1

=========================
  AWS Setup
=========================

#
# Pre-requisites
#

0.  Create a VPC and a subnet within it
  VPC Name:  SwarmCluster
  VPC Network:  192.168.0.0/16
  Subnet:  "PublicSubnet", 192.168.33.0/24  (after create, make sure to enable "Auto-Assign Public IP")
  
1.  Run CloudFormation template in this repo.  

	- Use defaults for IPs
	- Select the VPC and subnet created in step 0 from the drop downs
	- Use a keypair you have the private key for, in case you need to ssh into a machine directly.

2.  You will end up with these machines:

  manager: t2.micro / 192.168.33.11
  node01: t2.micro / 192.168.33.20
  node02: t2.micro / 192.168.33.21
  node03: t2.micro / 192.168.33.200

  AMI for all:  ami-dc8772bc in region US-WEST-2 only
  SG for all:  SG-WideOpen
  All have public IPs.

3.  Now ssh into master using its public IP.  We will do all cluster setup from this machine.  

  Note:  to ssh into any machines:
	- find the machine's public IP in EC2 dashboard
	ssh -i ~/.ssh/id_rsa_aws.pem ubuntu@<public IP>
Replace ~/.ssh/id_rsa_aws.pem with the private key corresponding to the public key you entered in the CloudFormation setup.


--- ALL STEPS AFTER THIS POINT DONE FROM MASTER / manager ---


4. Build the Discovery Service:

  docker -H=tcp://192.168.33.11:2375 run --restart=unless-stopped -d -h consul --name consul -v /mnt:/data \
  -p 192.168.33.11:8300:8300 \
  -p 192.168.33.11:8301:8301 \
  -p 192.168.33.11:8301:8301/udp \
  -p 192.168.33.11:8302:8302 \
  -p 192.168.33.11:8302:8302/udp \
  -p 192.168.33.11:8400:8400 \
  -p 192.168.33.11:8500:8500 \
  -p 172.17.0.1:53:53/udp \
  progrium/consul -server -advertise 192.168.33.11 -bootstrap
  
  docker -H=tcp://192.168.33.11:2375 run -d --name registrator -h registrator -v /var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://192.168.33.11:8500/
  
  
  High-availability:
  #docker -H=tcp://192.168.33.11:2375 run --restart=unless-stopped -d <PlaceAllPorts> -h consul1 --name consul1 progrium/consul -server -advertise 192.168.33.11 -bootstrap-expect 3
  #docker -H=tcp://192.168.33.12:2375 run --restart=unless-stopped -d <PlaceAllPorts> -h consul2 --name consul2 progrium/consul -server -advertise 192.168.33.12 -join 192.168.33.11
  #docker -H=tcp://192.168.33.13:2375 run --restart=unless-stopped -d <PlaceAllPorts> -h consul3 --name consul3 progrium/consul -server -advertise 192.168.33.13 -join 192.168.33.11

  Verify:
  docker exec -it consul<1/2/3> bash
	consul members

5. Build Swarm Managers:

  docker -H=tcp://192.168.33.11:2375 run --restart=unless-stopped -d -p 3375:2375 --name swarmgr swarm manage consul://192.168.33.11:8500/
  
  High-availability:
  #docker -H=tcp://192.168.33.11:2375 run --restart=unless-stopped -h mgr1 --name swarmgr1 -d -p 3375:2375 swarm manage --replication --advertise 192.168.33.11:3375 consul://192.168.33.11:8500/
  #docker -H=tcp://192.168.33.12:2375 run --restart=unless-stopped -h mgr1 --name swarmgr2 -d -p 3375:2375 swarm manage --replication --advertise 192.168.33.12:3375 consul://192.168.33.11:8500/
  #docker -H=tcp://192.168.33.13:2375 run --restart=unless-stopped -h mgr1 --name swarmgr3 -d -p 3375:2375 swarm manage --replication --advertise 192.168.33.13:3375 consul://192.168.33.11:8500/
  
6. Add nodes to the cluster:

  **Node01:**
  docker -H=tcp://192.168.33.20:2375 run --restart=unless-stopped -d -h consul-agt1 --name consul-agt1 -v /mnt:/data \
  -p 8300:8300 \
	-p 8301:8301 -p 8301:8301/udp \
	-p 8302:8302 -p 8302:8302/udp \
	-p 8400:8400 \
	-p 8500:8500 \
	-p 8600:8600/udp \
  progrium/consul -rejoin -advertise 192.168.33.20 -join 192.168.33.11
  
  docker -H=tcp://192.168.33.20:2375 run -d swarm join --advertise=192.168.33.20:2375 consul://192.168.33.20:8500/
  
  docker -H=tcp://192.168.33.20:2375 run -d --name registrator -h registrator -v /var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://192.168.33.11:8500/
  
  **Node02:**
  docker -H=tcp://192.168.33.21:2375 run --restart=unless-stopped -d -h consul-agt2 --name consul-agt2 -v /mnt:/data \
  -p 8300:8300 \
	-p 8301:8301 -p 8301:8301/udp \
	-p 8302:8302 -p 8302:8302/udp \
	-p 8400:8400 \
	-p 8500:8500 \
	-p 8600:8600/udp \
  progrium/consul -rejoin -advertise 192.168.33.21 -join 192.168.33.11
  
  docker -H=tcp://192.168.33.21:2375 run -d swarm join --advertise=192.168.33.21:2375 consul://192.168.33.21:8500/
  
  docker -H=tcp://192.168.33.21:2375 run -d --name registrator -h registrator -v /var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://192.168.33.11:8500/
  
  **Node03:**
  docker -H=tcp://192.168.33.200:2375 run --restart=unless-stopped -d -h consul-agt3 --name consul-agt3 -v /mnt:/data \
  -p 8300:8300 \
	-p 8301:8301 -p 8301:8301/udp \
	-p 8302:8302 -p 8302:8302/udp \
	-p 8400:8400 \
	-p 8500:8500 \
	-p 8600:8600/udp \
  progrium/consul -rejoin -advertise 192.168.33.200 -join 192.168.33.11
  
  docker -H=tcp://192.168.33.200:2375 run -d swarm join --advertise=192.168.33.200:2375 consul://192.168.33.200:8500/
  
  docker -H=tcp://192.168.33.200:2375 run -d --name registrator -h registrator -v /var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://192.168.33.11:8500/

7. Validating / Checking
   
   export DOCKER_HOST="tcp://192.168.33.11:3375"
   
   docker exec -it consul bash
   consul members
   
   curl http://192.168.33.11:8500/v1/catalog/nodes | python -m json.tool
   
   docker logs consul<1/2/3>
   docker logs swarmgr<1/2/3>

   curl http://192.168.33.11:8500/v1/catalog/services | python -m json.tool

8. Create overlay network

   export DOCKER_HOST="tcp://192.168.33.11:3375"
   docker network create --driver overlay mynetwork

9. Set up application layer containers:

  git clone https://github.com/docker/example-voting-app.git     # (this repo)
  cd example-voting-app

  docker-compose up