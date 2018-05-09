## About services
In a distributed application, different pieces of the app are called “services.” For example, if you imagine a video sharing site, it probably includes a service for storing application data in a database, a service for video transcoding in the background after a user uploads something, a service for the front-end, and so on.

Services are really just “containers in production.” A service only runs one image, but it codifies the way that image runs—what ports it should use, how many replicas of the container should run so the service has the capacity it needs, and so on. Scaling a service changes the number of container instances running that piece of software, assigning more computing resources to the service in the process.

Luckily it’s very easy to define, run, and scale services with the Docker platform -- just write a docker-compose.yml file.

## first docker-compose.yml file
A docker-compose.yml file is a YAML file that defines how Docker containers should behave in production.

### docker-compose.yml
Save this file as docker-compose.yml wherever you want. Be sure you have pushed the image you created in Part 2 to a registry, and update this .yml by replacing username/repo:tag with your image details.
``` 
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: username/repo:tag
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "80:80"
    networks:
      - webnet
networks:
  webnet:
```
This docker-compose.yml file tells Docker to do the following:
- Pull the image we uploaded in step 2 from the registry.
- Run 5 instances of that image as a service called web, limiting each one to use, at most, 10% of the CPU (across all cores), and 50MB of RAM.
- Immediately restart containers if one fails.
- Map port 80 on the host to web’s port 80.
- Instruct web’s containers to share port 80 via a load-balanced network called webnet. (Internally, the containers themselves publish to web’s port 80 at an ephemeral port.)
- Define the webnet network with the default settings (which is a load-balanced overlay network).
## Run your new load-balanced app
```
docker swarm init
```
** Note: We get into the meaning of that command in part 4. If you don’t run docker swarm init you get an error that “this node is not a swarm manager.”**
Now let’s run it. You need to give your app a name. Here, it is set to getstartedlab:
```
docker stack deploy -c docker-compose.yml getstartedlab 
```
Our single service stack is running 5 container instances of our deployed image on one host. Let’s investigate.

Get the service ID for the one service in our application:
```
docker service ls
ID                  NAME                 MODE                REPLICAS            IMAGE                          PORTS
tix0fcdnrgg8        getstartedable_web   replicated          0/5                 jeffzhouhhh/get-parted:part2   *:80->80/tcp 
```

Look for output for the web service, prepended with your app name. If you named it the same as shown in this example, the name is getstartedlab_web. The service ID is listed as well, along with the number of replicas, image name, and exposed ports.

A single container running in a service is called a task. Tasks are given unique IDs that numerically increment, up to the number of replicas you defined in docker-compose.yml. List the tasks for your service:
```
docker service ps getstartedable_web
ID                  NAME                       IMAGE                           NODE                DESIRED STATE       CURRENT STATE             ERROR                              PORTS
s5iu3ue603k7        getstartedable_web.1       jeffzhouhhh/get-started:part2   C80                 Running             Running 2 seconds ago
zj816gnn81z7         \_ getstartedable_web.1   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 5 minutes ago    "No such image: jeffzhouhhh/ge…"
zn4s7w83r7xf         \_ getstartedable_web.1   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 8 minutes ago    "No such image: jeffzhouhhh/ge…"
zwst6p2ys4pi         \_ getstartedable_web.1   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 10 minutes ago   "No such image: jeffzhouhhh/ge…"
ysrs1nse5u2x         \_ getstartedable_web.1   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 11 minutes ago   "No such image: jeffzhouhhh/ge…"
23h8st9torjh        getstartedable_web.2       jeffzhouhhh/get-started:part2   C80                 Ready               Ready 2 seconds ago
yqfom3vjnc44         \_ getstartedable_web.2   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 3 minutes ago    "No such image: jeffzhouhhh/ge…"
yh1jgwr0ecmz         \_ getstartedable_web.2   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 5 minutes ago    "No such image: jeffzhouhhh/ge…"
y6wnlvnhrof4         \_ getstartedable_web.2   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 6 minutes ago    "No such image: jeffzhouhhh/ge…"
yxdar8vjgi61         \_ getstartedable_web.2   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 11 minutes ago   "No such image: jeffzhouhhh/ge…"
la7cyj4u1nqu        getstartedable_web.3       jeffzhouhhh/get-started:part2   C80                 Running             Running 3 seconds ago
zf2ds8eog69m         \_ getstartedable_web.3   jeffzhouhhh/get-started:part2   C80                 Shutdown            Failed 2 minutes ago      "task: non-zero exit (1)"
z3tqmwmee27r         \_ getstartedable_web.3   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 10 minutes ago   "No such image: jeffzhouhhh/ge…"
z5ss050tld05         \_ getstartedable_web.3   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 11 minutes ago   "No such image: jeffzhouhhh/ge…"
znvz8shusvg0         \_ getstartedable_web.3   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 12 minutes ago   "No such image: jeffzhouhhh/ge…"
cpm1tgjp970x        getstartedable_web.4       jeffzhouhhh/get-started:part2   C80                 Ready               Ready 4 seconds ago
zwycxjbzjq0e         \_ getstartedable_web.4   jeffzhouhhh/get-started:part2   C80                 Shutdown            Failed 4 seconds ago      "task: non-zero exit (1)"
z1tib2ruwebn         \_ getstartedable_web.4   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 8 minutes ago    "No such image: jeffzhouhhh/ge…"
yvj124u1r26h         \_ getstartedable_web.4   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 10 minutes ago   "No such image: jeffzhouhhh/ge…"
ypkgxhw4oz6h         \_ getstartedable_web.4   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 11 minutes ago   "No such image: jeffzhouhhh/ge…"
0g8wisclat0c        getstartedable_web.5       jeffzhouhhh/get-started:part2   C80                 Running             Starting 1 second ago
y7fwhuijxvzt         \_ getstartedable_web.5   jeffzhouhhh/get-started:part2   C80                 Shutdown            Failed 2 minutes ago      "task: non-zero exit (1)"
zr28m5tgqase         \_ getstartedable_web.5   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 4 minutes ago    "No such image: jeffzhouhhh/ge…"
znl9ftgrve9i         \_ getstartedable_web.5   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 6 minutes ago    "No such image: jeffzhouhhh/ge…"
za415scwbv6q         \_ getstartedable_web.5   jeffzhouhhh/get-parted:part2    C80                 Shutdown            Rejected 9 minutes ago    "No such image: jeffzhouhhh/ge…" 
```
Tasks also show up if you just list all the containers on your system, though that is not filtered by service:
```
docker container ls -q 
8170daa1babc
9d09c864bf66
f8fbc680bf7c
cb74fb7e5db7
ef6c91619b87
```
result:<br> ![avatar](https://github.com/HaHaJeff/note/blob/master/docker/document/serivces/hello.png)<br/>

#Scale the app
You can scale the app by changing the replicas value in docker-compose.yml, saving the change, and re-running the docker stack deploy command:
``` 
docker stack deploy -c docker-compose.yml getstartedlab
```
Docker performs an in-place update, no need to tear the stack down first or kill any containers.

Now, re-run docker container ls -q to see the deployed instances reconfigured. If you scaled up the replicas, more tasks, and hence, more containers, are started.
## Take down the app and the swarm
- Take the app down with docker stack rm: <br> ``` docker stack rm getstartedlab ```  <br/>
- Take down the swarm. <br> ``` docker swarm leave --force ```  <br/>
