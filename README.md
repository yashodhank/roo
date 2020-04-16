# Roo (Beta)

This aims to be a free replacement of Amazon ECS, EKS, CertificateManager, Load-Balancer and CloudWatch. It IS a complete replacement for nginx, traefik, haproxy, and a lot of kubernetes. The idea is to give developers back the power and take it back from ridiculous self-complicating dev-ops tools that get more complicated and less useful (for example Traefik 2 just removed support for clustered Letsencrypt from their open source version to spruik their enterprise version. Nginx and HAProxy do the same). I wasted a lot of time on their software before writing this in a weekend with a friend. I truly hope it benefits others too.

If you are unfamiliar with swarm/kubernetes and are a developer and want a quick intro into how powerful and easy swarm can be, [check out my command notes](https://github.com/sfproductlabs/haswarm/blob/master/README.md). In a day I was scaling clusters up and down on my own infrastructure with single commands.

The power Roo gives you is to add HTTPS://example1.com and HTTPS://example2.com to your clustered services with zero configuration. Let's encrypt allocates your service's certificates. It works across every machine, docker node, service, in your cluster.

Roo itself is clustered. Every machine it runs on shares the load to your services. It's distributed store shares certificates from Letsencrypt used across all your nodes. Now apple is denying certificates older than a year, I feel as a dev, that lets encrypt is almost mandatory as it creates a lot of admin.

## Getting Started (on swarm)
* Create the default network, for example:
```
docker network create -d overlay --attachable forenet --subnet 192.168.9.0/24 
```
* Add label to the nodes you want to run it on (load_balancer describes the label for roo to attach to):
```
docker node update --label-add load_balancer=true docker1-prod
```
* You'll need to run roo on at least one manager node if you want docker services to auto-sync with Let's Encrypt. Don't worry though as this is not a security issue like in other platforms (like Traefik) as the manager node need not be publicly accessible.
* Run [the docker-comopose file](https://github.com/sfproductlabs/roo/blob/master/roo-docker-compose.yml) on swarm:
```
# docker stack deploy -c roo-docker-compose.yml roo
```
* Then in your own docker-compose file do something like (note the label used in the zeroconfig):
```yaml
version: "3.7"
services:
  test:
    image: test
    deploy:
      replicas: 1
    networks:
      - forenet
    labels:
      OriginHosts: test.sfpl.io
      OriginScheme: https
      OriginPort: 443
      DestinationHost: test_test
      DestinationScheme: http
      DestinationPort: 80
    
networks:
  forenet:
    external: true  
```
JOB DONE!
### Zeroconfig of docker swarm services
You need to tell roo what the incoming hostname etc is and where to route it to in the docker-compose file (if you want to go fully automoatic)

### Going Manual
Roo comes with a clustered Distributed Key-Value (KV) Store (See the API below for access). You can use this to manually configure roo. To add the route and do the zeroconfig example above manually, do this instead:
* You'll need to first deploy the test stack
```
docker stack deploy -c test-docker-compose.yml test
```
* Then let roo know the route (do this inside any machine in the forenet network):
```
curl -X PUT roo_roo:6299/roo/v1/kv/com.roo.host:test.sfpl.io:https -d 'http://test_test:80'
```

## Building from source
* You need to get the dependencies. Check the roo-docker-compose.yml file for a current list.
* Run ```make``` in the root directory (not the roo directory).

## Schema Definitions

### Adding a route to the routing table
* Use the Put API below
```
com.roo.host:<requesting_host<:port (optional)>:scheme>  <destination_url>
```
[See an example](https://github.com/sfproductlabs/roo/blob/master/README.md#a-real-example-route)

## API 
### Post a single key to the distributed store
```
curl -X PUT -d'test data' http://localhost:6299/roo/v1/kv/test
```
* Returns a json object { "ok" : true} if succeded
#### A Real Example Route
**Put in a test route**. For our example we are serving goole.com at our public endpoint, and routing to a swarm service with an endpoint inside our network on port 9001. The swarm stack is _cool_, the swarm service name is _game_. In docker swarm world this equates to a mesh network load balanced DNS record of tasks.cool_game. (you'll need to find your stack, service, and replace that and port 9001 with your details to get it working). The overall result to add this to our routing tables is:

```curl -X PUT -d'http://tasks.cool_game:9001' http://localhost:6299/roo/v1/kv/com.roo.host:google.com:https```

 * Note: as we don't specify a port in our requests we remove the optional port :443

To test you can run something like this (this just makes your localhost pretend like its responding to a request to google.com):
```curl -H "Host: google.com" https://localhost/```

So to summarize, google.com:443 is the incoming route to roo from the internet, and tasks.cool_game:9001 is your service and port to your internal service (in this case its an internal intranet _docker swarm_ service).

### Get a SINGLE key from the store
```
curl -X GET http://localhost:6299/roo/v1/kv/test
```
* Returns the raw bytes (you'll see this as a string if you stored it like that)
### List multiple keys from the store (Using a SCAN/Prefix filter query)
```
curl -X GET http://localhost:6299/roo/v1/kvs/te #Searches the prefix _te_
curl -X GET http://localhost:6299/roo/v1/kvs/tes #Searches the prefix _tes_
curl -X GET http://localhost:6299/roo/v1/kvs/test #Searches the prefix _test_
curl -X GET http://localhost:6299/roo/v1/kvs #Gets everything in the _entire_ kv store (filter is on nothing)
```
* Returns the rows searched using the SCAN query (in KV land its a prefix filter) in JSON
* The resulting values are encoded in base64, so you may need to convert them (unlike the GET single query above which returns raw bytes)
* Use ```window.atob("dGVzdCBkYXRh")``` in javascript. Use json.Unmarshall or string([]byte) in Golang Go if you want a string.

## Help!

### Inspecting the roo containers
* Inspect the logs
```
docker service logs roo_roo -f
```
* Inspect the containers
```
docker run -it --net=forenet alpine ash
nslookup tasks.roo_roo.
curl -X GET http://<result_of_nslookup>:6299/roo/v1/kvs
```

## TODO
* [ ] Add support for Elastic's APM https://www.elastic.co/guide/en/apm/get-started/current/quick-start-overview.html
* [ ] Add an option to whitelist hostnames only in the store (this will prevent dodgy requests)
* [ ] Add a synchronized scheduler so that only one docker manager runs the auto-update script (it currently depends on 1 manager node notifying the slaves indirectly via the kv store)
* [ ] Memory api checker needs to be cached in hourly, replace kvcache, docker update, add node to hosts during join so if it fails it can be deleted, cache host whitelist
* [ ] Downscale swarm cleaner (removing a container should remove the raft address and nodehost, could run a leader process like in dcrontab) @psytron or link with docker connector
* [ ] Autoscale Docker
* [ ] Autoscale Physical Infratructure
* [ ] Move flaoting IPs (Load balance, service down)
* [ ] SSL in API
* [ ] HTTP for Proxying (Only SSL Supported atm)
* [ ] Auto downgrade 
* [ ] Add end to end encryption of kv-store and distributed raft api and api:6299

## Credits
* [DragonBoat](https://github.com/lni/dragonboat)
* [DragonGate](https://github.com/dioptre/DragonGate)
* [SF Product Labs](https://sfproductlabs.com)
