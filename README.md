# Consul Agent in Docker

This project is a Docker container for [Consul](http://www.consul.io/). It's a slightly opinionated, pre-configured Consul Agent made specifically to work in the Docker ecosystem.

## Getting the container

The container is very small (28MB virtual, based on busybox) and available on the Docker Index:

	$ docker pull progrium/consul

## Using the container

#### Just trying out Consul

If you just want to run a single instance of Consul Agent to try out its functionality:

	$ docker run -p 8400:8400 -p 8500:8500 -p 8600:53/udp -h node1 progrium/consul -server -bootstrap

We publish 8400 (RPC), 8500 (HTTP), and 8600 (DNS) so you can try all three interfaces. We also give it a hostname of `node1`. Setting the container hostname is the intended way to name the Consul Agent node. 

Our recommended interface is HTTP using curl:

	$ curl localhost:8500/v1/catalog/nodes

We can also use dig to interact with the DNS interface:

	$ dig @0.0.0.0 -p 8600 node1.node.cluster

However, if you install Consul on your host, you can use the CLI interact with the containerized Consul Agent:

	$ consul members

#### Testing a Consul cluster on a single host

If you want to start a Consul cluster on a single host to experiment with clustering dynamics (replication, leader election), here is the recommended way to start a 3 node cluster. 

We're **not** going to start the first node in bootstrap mode because we want it as a stable IP for the others to join the cluster. 

	$ docker run -d --name node1 -h node1 progrium/consul -server

We can get the container's internal IP by inspecting the container. We'll put it in the env var `JOIN_IP`.

	$ JOIN_IP="$(docker inspect -f '{{ .NetworkSettings.IPAddress }}' node1)"

Then we'll start `node2`, which we'll run as the bootstrap (forced leader) node, and tell it to join `node1` using `$JOIN_IP`:

	$ docker run -d --name node2 -h node2 progrium/consul -server -bootstrap -join $JOIN_IP

Now we can start `node3`. Very simple:

	$ docker run -d --name node3 -h node3 progrium/consul -server -join $JOIN_IP

That's a three node cluster. Notice we've also named the containers after their internal hostnames / node names. At this point, we can kill and restart `node2` without bootstrap mode since otherwise it will always be the leader.

	$ docker rm -f node2
	$ docker run -d --name node2 -h node2 progrium/consul -server -join $JOIN_IP

We now have a real cluster running on a single host. We haven't published any ports to access the cluster, but we can use that as an excuse to run a fourth agent node in "client" mode. This means it doesn't participate in the consensus quorum, but can still be used to interact with the cluster.

	$ docker run -d -p 8400:8400 -p 8500:8500 -p 8600:53/udp -h node4 progrium/consul -join $JOIN_IP

Now we can interact with the cluster on those published ports and, if you want, play with killing, adding, and restarting  nodes to see how the cluster handles it.

#### Running real Consul cluster in a production environment

Setting up a real cluster on separate hosts is very similar to our single host cluster setup process. However, we're going to pass the `-advertise` flag with the machine's external IP. This should be on a private network otherwise more advanced encryption is needed. You'll also need to publish all the ports to this interface, including the internal Consul ports (8300, 8301, 8302).

Assuming we're on a host with a private IP of 10.0.1.1, we can start the first host agent:

	$ docker run -d -h node1 \
		-p 10.0.1.1:8300:8300 \
		-p 10.0.1.1:8301:8301 \
		-p 10.0.1.1:8302:8302 \
		-p 10.0.1.1:8400:8400 \
		-p 10.0.1.1:8500:8500 \
		-p 10.0.1.1:8600:53/udp \
		progrium/consul -server -advertise 10.0.1.1

On the second host, we'd run the same thing, but passing `-bootstrap` and a `-join` to the first node's IP. Let's say the private IP for this host is 10.0.1.2:

	$ docker run -d -h node2 \
		-p 10.0.1.2:8300:8300 \
		-p 10.0.1.2:8301:8301 \
		-p 10.0.1.2:8302:8302 \
		-p 10.0.1.2:8400:8400 \
		-p 10.0.1.2:8500:8500 \
		-p 10.0.1.2:8600:53/udp \
		progrium/consul -server -advertise 10.0.1.2 -bootstrap -join 10.0.1.1

And the third host with an IP of 10.0.1.3:

	$ docker run -d -h node3 \
		-p 10.0.1.3:8300:8300 \
		-p 10.0.1.3:8301:8301 \
		-p 10.0.1.3:8302:8302 \
		-p 10.0.1.3:8400:8400 \
		-p 10.0.1.3:8500:8500 \
		-p 10.0.1.3:8600:53/udp \
		progrium/consul -server -advertise 10.0.1.3 -join 10.0.1.1

Once the third host is running, you want to go back to the second host, kill the container, and run it again just as before but without the `-bootstrap` flag. You'd then have a full cluster running in production on a private network.

## Opinionated Configuration

#### DNS

This container was designed assuming you'll be using it for DNS on your other contianers. So it listens on port 53 inside the container to be more compatible and accessible via linking. It also has DNS recursive queries enabled, using the Google nameservers.

It also assumes DNS is a primary means of service discovery for your entire system. It uses `cluster` as the top level domain instead of `consul` just as a more general/accurate naming ontology.

#### Runtime Configuration

Although you can extend this image to add configuration files to define services and checks, this container was designed for environments where services and checks can be configured at runtime via the HTTP API. 

It's recommended you keep your check logic simple, such as using inline `curl` or `ping` commands. Otherwise, keep in mind the default shell is `ash`. 

## Sponsor

This project was made possible thanks to [DigitalOcean](http://digitalocean.com).

## License

BSD