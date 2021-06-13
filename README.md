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

## Index  
*   [Container networking](#Container-networking)  
*   [Start MySQL](#Start-MySQL)  
*   [Connect to MySQL](#Connect-to-MySQL)  
*   [Run your app with MySQL](#Run-your-app-with-MySQL)  
*   [Recap](#Recap)  


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
```console
    docker network create todo-app  
```  
2.  Start a MySQL container and attach it to the network. We're also going to define a few environment variables that the database will use to initialize the database (see the "Environment Variables" section in the [MySQL Docker Hub listing](https://hub.docker.com/_/mysql/))  

```console
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

3.  To confirm we have the database up and running, connect to the database and verify it connects.  
```console
    docker exec -it <mysql-container-id> mysql -u root -p
```  
When the password prompt comes up, type in **secret**. In the MySQL shell, list the databases and verify you see the ```todos``` database.  

```mysql
    mysql> SHOW DATABASES;
```  
You should see output that looks like this:  

```mysql
 +--------------------+
 | Database           |
 +--------------------+
 | information_schema |
 | mysql              |
 | performance_schema |
 | sys                |
 | todos              |
 +--------------------+
 5 rows in set (0.00 sec)
```  
Now we have our ```todos``` database and it's ready for us to use!  

## Connect to MySQL  
Now that we know MySQL is up and running, let's use it! But, the question is... how? If we run another container on the same network, how do we find the container (remember each container has its own IP adress)?  

To figure it out, we're going to make use of the nicolaka/netshoot container, which ships with a lot of tools that are useful for troubleshooting or debugging networking issues.  

1.  Start a new container using the nicolaka/netshoot image. Make sure to connect it to the same network.  
```console
    docker run -it --network todo-app nicolaka/netshoot
```  

2.  Inside the container, we're going to use the ```dig``` command, which is a useful DNS tool. We're going to look up the IP adress for the hostname ```mysql```.  
```console
    dig mysql
```  
And you'll get an output like this...  
```console
 ; <<>> DiG 9.14.1 <<>> mysql
 ;; global options: +cmd
 ;; Got answer:
 ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 32162
 ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

 ;; QUESTION SECTION:
 ;mysql.				IN	A

 ;; ANSWER SECTION:
 mysql.			600	IN	A	172.23.0.2

 ;; Query time: 0 msec
 ;; SERVER: 127.0.0.11#53(127.0.0.11)
 ;; WHEN: Tue Oct 01 23:47:24 UTC 2019
 ;; MSG SIZE  rcvd: 44
```  
In the "ANSWER SECTION", you will see an ```A``` record for ```mysql``` that resolves to ```172.23.0.2```(your IP adress will most likely have a different value). While ```mysql``` isn't normally a valid hostname, Docker was able to resolve it to the IP adress of the container that had that network alias (remember the ```--network-alias``` flag we used earlier?).  

What this means is... our app only simply needs to connect to a host named ```mysql``` and it'll talk to the database! It doesn't get much simpler than that.  


## Run your app with Mysql  
The todo app supports the setting of a few environment variables to specify MySQL connection settings. The are:  
*   ```MYSQL_HOST``` - the hostname for the running MySQL server  
*   ```MYSQL_USER``` - the username to use for the connection  
*   ```MYSQL_PASSWORD``` - the password to use for the connection  
*   ```MYSQL_DB``` - the database to use once connected  

> <span style="color:#147ac8">Setting connection Settings via Env Vars</span>  
While using env vars to set connection settings is generally ok for development, it is **HIGHLY DISCOURAGED** when running applications in production. Diogo Monica, the former lead of security at Docker, [wrote a fantastic blog post](https://diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/) explining why.  
A more secure mechanism is to use the secret support provided by your container orchestration framework. In most cases, these secrets are mounted as files in the running container. You'll see many apps (including the MySQL image and the todo app) also support env vars with ```_FILE``` suffix to point to a file containing the variable.  
As an example, setting the ```MYSQL_PASSWORD_FILE``` var will cause the app to use the contents of the referenced file as the connection password. Docker doesn't do anything to support these env vars. Your app will need to know to look for the variable and get the file contents.  

With all of that explained, let's start our dev-ready container!  

1.  We'll specify each of the environment variables above, as well as connect the container to our app network.  

```console
    docker run -dp 3000:3000 \
    -w /app -v "$PWD:/app" \
    --network todo-app \
    -e MYSQL_HOST=mysql \
    -e MYSQL_USER=root \
    -e MYSQL_PASSWORD=admin \
    -e MYSQL_DB=todos \
    node:12-alpine \
    sh -c "yarn install && yarn run dev"
```  

2.  If we look at the logs for the container (```docker logs <container-id>```), we should see a message indicating it's using the mysql database.  

```console
 # Previous log messages omitted
 $ nodemon src/index.js
 [nodemon] 1.19.2
 [nodemon] to restart at any time, enter `rs`
 [nodemon] watching dir(s): *.*
 [nodemon] starting `node src/index.js`
 Connected to mysql db at host mysql
 Listening on port 3000
```  

3.  Open the app in your browser and add a few items to your todo list.  

4.  Connect to the mysql database and prove that the item are being written to the database. Remember, the password is **admin**.  
```console
    docker exec -it <mysql-container-id> mysql -p todos
```
And if you check the ```todos``` database, it has now the ```todo_items``` table, which stores all the items we've added, run the following:  
```mysql
     mysql> select * from todo_items;
    +--------------------------------------+--------------------+-----------+
    | id                                   | name               | completed |
    +--------------------------------------+--------------------+-----------+
    | c906ff08-60e6-44e6-8f49-ed56a0853e85 | Do amazing things! |         0 |
    | 2912a79e-8486-4bc3-a4c5-460793a575ab | Be awesome!        |         0 |
    +--------------------------------------+--------------------+-----------+
```

## Recap  
At this point, we have an application that now stores its data in an external database running in a separate container. We learned a little bit about container networking and saw how service discovery can be performed using DNS.  

But there's a good chance you are starting to feel a little overwhelmed with everything you need to do to start up this application. We have to create a network, start containers, specify all of the environment variables, expose ports, and more! That's a lot to remember and it's certainly making things harder to pass to someone else.  

In the next section, we'll talk about Docker Compose. With Docker Compose, we can share our applications stacks in a much easier way and let other spin them up with a single (and simple) command!  


