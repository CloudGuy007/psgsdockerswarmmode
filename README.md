# psgsdockerswarmmode

## 2. Why Multiple Container Hosts?
### 4 What if a Single Container isn't Enough?
```
docker run -d -p 3000:3000 swarmgs/nodewebstress
```
stress test
```
ab -n 100 http://127.0.0.1:3000/customer/1
```

parallel test
```
echo "http://127.0.0.1:3000/customer/1\http://127.0.0.1:3001/customer/2" | parallel -j 2 "ab -n 100 {.}"
```
## 3.Creating a Swarm and Running a a Service
### 3 Enabling Swarm Mode by Initializing a new Swarm
```
docker swarm -h
```
### 4 Listing and Inspecting Nodes
```
docker node ls
docker node inspect self
```

### 5 Creating an Nginx Service
```
docker service create --name web --publish 8080:80 nginx
```

### 7 Services Lead to Tasks
```
docker service ls
docker service ps web
```

### 8 Removing a Service
```
docker service rm web
```
### 9 Updating a Service to Scale the Number of Containers
make 2 containers(not make 2 more containers)
```
docker service ps web
docker service update --replicas=2 web
```
scale
```
docker service scale web=4
```
### 10 Swarm Managers Ensure the Desired State Is Maintained
```
docker stop 123
```

## 4. Adding Nodes
### 2 Destroying the Single Node Swarm
```
docker swarm leave --force
```

### 7  docker swarm init --advertise-addr
```
 docker swarm init --advertise-addr ip
 
 ```
### 8 Joining Worker Nodes to the Swarm Mode

```
docker swarm join-token worker
docker swarm join-token manager
```
### 9 Creating a Service to Visualize Our Cluster State
```
docker service create --name viz --publish 8090:8080 --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock --constraint=node.role==manager manomarks/visualizer
```

```
docker service ls
docker service ps viz
```
## 10 What Happens When a Node Is Shut Down
```
docker node rm -f c
```
## 16 Promoting a Worker to a Manager
```
docker node promote worker1
docker node demote manager1
```

## 17 Draining a Node to Perform Maintenance
```
docker node update --availablity=drain worker2
```
this will move all container from worker2 to other workers
restore
```
docker node update --availablity=active worker2
```
trick: scaleup then scale down
## 18 One Container per Node with Global Services
[see here](https://blog.codeship.com/monitoring-docker-containers-with-elasticsearch-and-cadvisor/)
```
docker service create --mode=global --name cadvisor --mount type=bind,source=/,target=/rootfs,readonly=true \
  --mount type=bind,source=/var/run,target=/var/run,readonly=false \
  --mount type=bind,source=/sys,target=/sys,readonly=true \
  --mount type=bind,source=/var/lib/docker/,target=/var/lib/docker,readonly=true \
  google/cadvisor
```

## 5. Ingress Routing and Publishing Ports
### 3 The Ingress Overlay Network
```
docker network ls
docker network inspect ingress //we can get published port
```
### 7 Removing a Published Port on an Existing Service
```
docker service update --publish-rm 8080 cadvisor
```
### 8 Adding a Host Mode Published Port
```
docker service update --publish-add mode=host,published=8080,target=8080 cadvisor
```

### 9 Publishing a Random Port
```
docker service create --name random -p target=80 nginx
```
using inspect to get port number
```
docker service inspect random --pretty
```

## 6. Reconciling a Desired State


## 7. Rolling Updates
### 5 Specifying an Image Tag When Creating a Service
```
docker service create --name pay -p 3000:3000 swarmgs/payroll:1
```
```
docker service scale pay=5
```
### 6 Adding Delay Between Task Updates
```
docker service update --image swarmgs/payroll:3 --update-delay=1s pay
```
### 7 Updating Multiple Tasks Concurrently with --update-parallelism
```
docker service update --image swarmgs/payroll:3 --update-delay=1s --update-parallelism=2 pay
```
### 8 Cleaning up Task History When Learning

### 16 Rolling Back to the Previous Service Definition
```
docker service update --rollback pay
```

## 17 Use --force to Test Changes to Update Policies
```
docker service update --force pay
```
## 18 Watching UpdateStatus During a Service Update
```
watch -d -n 0.5 "docker service inspect pay | jq .[].UpdateStatus"
```
## 8. Container to Container Networking 
### 4 Creating an Overlay Network
```
docker network create -d overlay --subnet=10.0.9.0/24 backend
```
### 6 Attaching a New Service to Our Overlay Network
```
docker service create --name balance -p 5000:3000 --network backend swarmgs/balance
```

## 9. Deploying with Stacks
### 1 Enough with All the Flags Already
```
docker service create --name viz --publish 8090:8080 --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock --constraint=node.role==manager manomarks/visualizer
```
convert to compose
```
version: '3.1'
services:
  viz:
    image: manomarks/visualizer
    volumes:
      - "/var/run/docker.sock:/var/ru/docker.sock"
    deploy:
      placement:
        constraints:
          - node.role==manager
    
```
### 5 Deploying a Stack with a Compose File
```
docker stack deploy -c viz.yml viz
docker service ps viz_viz
```
stackname: viz
```
docker stack ps viz
```

### 6 Updating a Service with a Stack Is as Easy as Creating It
```
version: '3.1'
services:
  viz:
    image: manomarks/visualizer
    volumes:
      - "/var/run/docker.sock:/var/ru/docker.sock"
    ports:
      - "8090:8080"
    deploy:
      placement:
        constraints:
          - node.role==manager
```
### 7 Removing a Stack
remove
```
docker stack rm viz
```

### 8 Creating and Deploying a Multi Service Stack
```
version: '3.1'
services:
  customer:
    image: swarmgs/customer
  balance:
    image: swarmgs/balance
    ports:
      - "5000:3000"
    environment:
      MYWEB_CUSTOMER_API: "customer:3000"
```
```
docker stack deploy -c apis.yml apis
docker stack services apis
docker stack ps apis
docker services ps apis_customer
```

### 9 Specifying Replicas in Compose File
```
version: '3.1'
services:
  customer:
    image: swarmgs/customer
    deploy:
      replicas: 5
  balance:
    image: swarmgs/balance
    ports:
      - "5000:3000"
    environment:
      MYWEB_CUSTOMER_API: "customer:3000"
    deploy:
      replicas: 2
```

## 10. Health Checking
### 2 Deploying a Cowsay Stack
```
version: '3.1'
services:
  cowsay:
    image: swarmgs/cowsay
    ports:
      - "7000:80"
    deploy:
      placement:
        constraints:
          - node.role==manager
```
```
docker stack deploy -c cowsay.yml cow
```


### 3 What Happens if We Break a Container?
```
docker exec -it $(docker ps -f name=cow -q) bash
mv /usr/games/fortune /usr/games/fortune2
```
watch
```
watch -n 0.5 docker stack ps cow
```



### 4 Automatic Service Recovery with Health Checks
```
version: '3.1'
services:
  cowsay:
    image: swarmgs/cowsayhealth
    ports:
      - "7001:80"
    deploy:
      placement:
        constraints:
          - node.role==manager
```
```
docker stack deploy -c cowsayhealth.yml cowh
```


### 5 Manually Forcing a Corrupted Service to Restart
```
version: '3.1'
services:
  calc:
    image: swarmgs/calc
    ports:
      - "7000:80"
    deploy:
      placement:
        constraints:
          - node.role==manager
```
```
docker stack deploy -c calc.yml calc
docker service update --force calc_calc
```


### 7 Adding a Health Check to a Stack Compose File
healthcheck command.
- 0 success
- 1 unhealthy
- 2 reserved  

### 8 Configuring Interval and Timeout and Retries
```
--interval
--timeout
--retries
```
```
version: '3.1'
services:
  calc:
    image: swarmgs/calc
    ports:
      - "7000:80"
    healthcheck:
      test: curl -f -s -S htt[://localhost/calc/iseverythingok|| exit 1
      interval: 5s
      timeout: 5s
      retries: 3
    deploy:
      placement:
        constraints:
          - node.role==manager
```
### 9 Deploying Health Checks and Inspecting Container Health



## 11. Protecting Secrets
### 1 Environment Variables Can Leak Passwords

```
vi mysql.yml
```
edit
```
version: '3.1'
services:
  mysql:
    image: mysql
    environment:
      MYSQL_USER: wordpress
      MYSQL_DATABASE: wordpress
      MYSQL_ROOT_PASSWORD: root
    deploy:
      placement:
        constraints:
          - node.role==manager

```
```
docker stack deploy -c mysql.yml mysql
docker ps
docker exec -it 12345678 bash
```

```
mysql -uroot -p
show databases;
```

### 2 Creating a Secret for a MySQL Password
```
echo pass |docker secret create mysql_root_pass -
```
ls
```
docker secret ls
```
### 3 Granting a Service Access to a Secret
```
version: '3.1'
services:
  mysql:
    image: mysql
    environment:
      MYSQL_USER: wordpress
      MYSQL_DATABASE: wordpress
      #MYSQL_ROOT_PASSWORD: root
    secrets:
      - root_pass   
      # not mysql_root_pass
    deploy:
      placement:
        constraints:
          - node.role==manager

```
### 4 Troubleshooting a Failing Service
```
docker stack ps mysql
docker service
docker service logs mysql_mysql
```
we must specify MYSQL_ROOT_PASSWORD MYSQL_ALLOW_EMPTY_PASSWORD MYSQL_RANDOM_ROOT_PASSWORD.
```
version: '3.1'
services:
  mysql:
    image: mysql
    environment:
      MYSQL_USER: wordpress
      MYSQL_DATABASE: wordpress
      #add
      MYSQL_ALLOW_EMPTY_PASSWORD: "yes"
      #MYSQL_ROOT_PASSWORD: root
    secrets:
      - root_pass   
      # not mysql_root_pass
    deploy:
      placement:
        constraints:
          - node.role==manager
```

### 5 Accessing Secrets in a Container via the Filesystem
```
docker stack ps mysql
docker exec -it $(docker ps | head -2 | tail -1 | cut -d ' ' -f 1) bash
```
### 6 Using a Secret to Provide a MySQL Root Password
```
cd run/secrets
cat root_pass
```
```
version: '3.1'
services:
  mysql:
    image: mysql
    environment:
      MYSQL_USER: wordpress
      MYSQL_DATABASE: wordpress
      MYSQL_ROOT_PASSWORD_FILE: "/run/secrets/root_pass"
      #MYSQL_ROOT_PASSWORD: root
    secrets:
      - root_pass   
      # not mysql_root_pass
    deploy:
      placement:
        constraints:
          - node.role==manager
```
### 7  Steps to Use Secrets
If use command line
```
docker service create -secret X
```
another way is to use cdocker-compose


### 8 _FILE Image Secrets Convention

### 9 Removing Secrets
```
docker secret rm mysql_root_pass
```
If the secret is in use,remove stack first
```
docker stack rm mysql
```

### 10 A Convention for Updating a Secret
create password:
```
echo pass1 |docker secret create mysql_root_pass_v1 -
echo pass2 |docker secret create mysql_root_pass_v2 -
```
```
version: '3.1'
services:
  mysql:
    image: mysql
    environment:
      MYSQL_USER: wordpress
      MYSQL_DATABASE: wordpress
      MYSQL_ROOT_PASSWORD_FILE: "/run/secrets/root_pass"
      #MYSQL_ROOT_PASSWORD: root
    secrets:
      # add source
      - source: root_pass_v1
        target: root_pass
      # not mysql_root_pass
    deploy:
      placement:
        constraints:
          - node.role==manager
```
