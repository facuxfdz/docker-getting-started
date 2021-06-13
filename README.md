# Docker Getting Started Tutorial

This tutorial has been written with the intent of helping folks get up and running
with containers and is designed to work with Docker Desktop. While not going too much 
into depth, it covers the following topics:

- Running your first container
- Building containers
- Learning what containers are running and removing them
- Using volumes to persist data
- Using bind mounts to support development
- Using container networking to support multi-container applications
- Using Docker Compose to simplify the definition and sharing of applications
- Using image layer caching to speed up builds and reduce push/pull size
- Using multi-stage builds to separate build-time and runtime dependencies


# Part 7: Multi-Container apps  

## Container networking  
Remember that containers, by default, run in isolation and don't know anything about other processes or containers on the same machine. So, how do we allow one container to talk to another? The answer is **networking**. Now, you don't have to be a network engineer. Simply remember this rule...  
> <span style="color: #147ac8"> Note </span>  
If two containers are on the same network, the can talk to each other. If they aren't, the can't

## Start MySQL  
There are two ways to put a container on a network:  
1.   Assign it at start.  
2.  Connect an existing container.  

For now, we will create the network first and attach the MySQL container at startup.  

1.  Create the network.  
```docker
    docker network create todo-app  
```  
2.  Start a MySQL container and attach it to the network. We're also going to define a few environment variables that the database will use to initialize the database (see the "Environment Variables" section in the [MySQL Docker Hub listing](https://hub.docker.com/_/mysql/))  

```dockerfile
    docker run -d \
    --network todo-app --network-alias mysql \
    -v todo-mysql-data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=todos \
    mysql:5.7
```

You'll also see we specified the ```--network-alias``` flag. We'll come back to that in just a moment. 

> <span style="color: #147ac8"> Tip </span>  
You'll notice we're using a volume named ```todo-mysql-data``` here and mounting it at ```var/lib/mysql```, which is where MySQL stores its data. However, we never ran a ```docker volume create``` command. Docker recognizes we want to use a named volume and creates one automatically for us.  

