# Define a container with Dockerfile
Dockerfile defines what goes on in the environment inside your container. Access to resources like networking interfaces and disk drives is virtualized inside this environment, which is isolated from the rest of your system, so you need to map ports to the outside world, and be specific about what files you want to “copy in” to that environment. However, after doing that, you can expect that the build of your app defined in this Dockerfile behaves exactly the same wherever it runs.

## Dockerfile
Create an empty directory. Change directories (cd) into the new directory, create a file called Dockerfile, copy-and-paste the following content into that file, and save it. Take note of the comments that explain each statement in your new Dockerfile.
```
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"] 
```
This Dockerfile refers to a couple of files we haven’t created yet, namely app.py and requirements.txt. Let’s create those next.

## The app itself
Create two more files, requirements.txt and app.py, and put them in the same folder with the Dockerfile. This completes our app, which as you can see is quite simple. When the above Dockerfile is built into an image, app.py and requirements.txt is present because of that Dockerfile’s ADD command, and the output from app.py is accessible over HTTP thanks to the EXPOSE command.

### requirements.txt
```
Flask
Redis
```

### app.py
```
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```
Now we see that pip install -r requirements.txt installs the Flask and Redis libraries for Python, and the app prints the environment variable NAME, as well as the output of a call to socket.gethostname(). Finally, because Redis isn’t running (as we’ve only installed the Python library, and not Redis itself), we should expect that the attempt to use it here fails and produces the error message.
**Note: Accessing the name of the host when inside a container retrieves the container ID, which is like the process ID for a running executable. **
That’s it! You don’t need Python or anything in requirements.txt on your system, nor does building or running this image install them on your system. It doesn’t seem like you’ve really set up an environment with Python and Flask, but you have.

## Build the app
We are ready to build the app. Make sure you are still at the top level of your new directory. Here’s what ls should show:
```
$ ls
Dockerfile 	  app.py    requiresment.txt
```
Now run the build command. This creates a Docker image, which we’re going to tag using -t so it has a friendly name.
```
$ docker build -t frienthello .
```
Where is your built image? It’s in your machine’s local Docker image registry:
```
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
friendlyhello       latest              bd3b2bb8e6fd        11 minutes ago      151MB
frienthello         latest              e7a50e816285        34 minutes ago      151MB
<none>              <none>              68def2cc0966        35 minutes ago      140MB
python              2.7-slim            46ba956c5967        29 hours ago        140MB
hello-world         latest              e38bc07ac18e        3 weeks ago         1.85kB
```

## Run the app
Run the app, mapping your machine’s port 4000 to the container’s published port 80 using -p:
```
docker run -p 4000:80 friendlyhello
```
use the curl command in a shell to view the same content.

```
$ curl http://localhost:4000
<h3>Hello World!</h3><b>Hostname:</b> 8fc990912a14<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
```

## Share your image
To demonstrate the portability of what we just created, let’s upload our built image and run it somewhere else. After all, you need to know how to push to registries when you want to deploy containers to production.
``` 
$ docker login
```
For example:
```
docker tag friendlyhello jeffzhouhhh/get-started:parted2 
```
Run docker image ls to see your newly tagged image.
``` 
$ docker image ls
REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
friendlyhello             latest              bd3b2bb8e6fd        32 minutes ago      151MB
jeffzhouhhh/get-started   part2               bd3b2bb8e6fd        32 minutes ago      151MB
python                    2.7-slim            46ba956c5967        30 hours ago        140MB
hello-world               latest              e38bc07ac18e        3 weeks ago         1.85kB
```
### Publish the image
Upload your tagged image to the repository:
```
docker push username/repository:tag
```
Once complete, the results of this upload are publicly available. If you log in to Docker Hub, you see the new image there, with its pull command.
