---
layout: post
title: Development environment with docker
date: 25-09-2016 21:00

---

#Development environment with docker

For the last couple of months I've switched to permanently using docker for development.
Containers are awesome. In this post I'll try to address some of the main pain points I was faced with early on and how I've managed to resolve them.

### Persistence
The one thing that I've found to be particularly painful has been persistence. I like the idea of stateless machines as much as the next guy. Nevertheless it's quite annoying whenever I've had to restore from a postgres backup or re-running migrations again and again.

There are two options work around this limitation.  

 * Option 1: Forget about containerizing your database. Your database is not stateless. Run it on your host and connect to it as you would any other backing service. After all, this is what you would do in production, right? 
 * Option 2: Use an intermediary data-only container and back up that volume. You can move it around hosts or even share it with your colleagues.
 
The later option is the one we'll be covering in this post. First off, a quick primer on docker volumes.  
Docker volumes live beyond your containers lifecycle and thus available to the host, even after you've stopped the container.
This is a double edged sword, if you aren't careful, you'll from time to time find yourself running out of diskspace.
I find it somewhat convenient as you could make changes outside of a running container. Though I would not recommend doing that, since any running container would be unaware of the changes and result in corrupted files.  
Here is a command I use to find information on a volume.   

	docker volume ls -q|head -n1| xargs docker volume inspect  

This command will pick the first container and display its details, information like labels if any were applied, name and a mount point. On my host, the folder is located in `/var/lib/docker/volumes/{image-tag}/_data.`

 We'll start off by creating a volume. For our specific use-case, I am using the official posgres image which can be found here [https://hub.docker.com/_/postgres/](https://hub.docker.com/_/postgres/). Further down, there are few options that we can override.   
 
For example, optional environment variables that can be used to define another location, like a subdirectory for the database files. Which is perfect for our use case. So we'll specify that as our amount point.

To create that data volume, you would write
 	
    docker create -v /var/lib/postgresql/data --name pgdata busybox /bin/true
 
This specifies the mount destination for the volume, and we've named it pgdata for reference later on. You can find more details about the volume by typing `docker inspect pgdata`.

So now when we launch a postgres container we tell docker to mount the volume from our pgdata container. We also have to tell postgres where we want out data. To launch such a container we would write.

	docker run -d --volumes-from pgdata \
	 -e PGDATA=/var/lib/potgresql/data \
	 -e POSTGRES_USER=my_user
	 --name postgres postgres 

To try it out, run `docker exec -it postgres bash`. And once you are in you should be able to access your database as your user `psql -U my_user`. 
We can now create a test database and insert some test data.
 	
	create table testing (name varchar(50), age int);
	insert into testing (name, age) values (('Bob', 23), ('Mike', 25));

Now go ahead and log out, stop and remove that container. You should now be able to start an identical container and your data will still be available. 

I've found this to be convenient to my workflow. I always have a postgres container running in the background. Only as opposed to one installed on my host, this is a lightweight container perfectly suited for the kind of usage involved in development. This also allows me to evaluate the latest versions of postgres and give the new features a spin without the hassle of setting it up or worries of losing data.  

### Networking
Another neat feature when using docker for development is the internal host resolving. It can sometimes get cumbersome connecting services across projects. What I do is create a network, and launch multiple services that need to communicate on the same network.  
As an example you could create a network like this one:

	docker network create simple_network
	
This will create a network bridge that can be referenced by name. So all subsequent containers launched within this network need only reference each-others names. And docker will take care of the name resolving.   
As an example the following command would start redis and expose the default port 6379.  

	docker run --network simple_network --name my_redis -d redis
 
 This means all containers launched within this network will be able to not only access this container, but simply reference it as `my_redis` and docker will take care of the internal resolving.  
This is particularly useful when your projects all share the same kind of resources. i.e a database, cache, MQ etc. 

I've become quite fond of using docker for development. Even managed to deploy a few services into production. And its definitely something I'll be using going forward.  

  

