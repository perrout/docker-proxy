#NGINX Proxy Reverse for Docker


##Step 1. Starting up nginx-proxy

To get started, let’s start up the **nginx-proxy** container. This can be accomplished either by a single **docker** command, or using **docker-compose**. Let’s cover both.

Before we get started, either way, we need to first create a Docker network that we will use to bridge all of these containers together.

	$ docker network create nginx-proxy

From now on, we need to ensure that we’re always adding new containers to the nginx-proxy Docker network.
Installing **nginx-proxy** with Docker

	$ docker run -d -p 80:80 --name nginx-proxy --net nginx-proxy -v /var/run/docker.sock:/tmp/docker.sock jwilder/nginx-proxy

###Installing nginx-proxy with docker-compose

First, create a new **docker-compose.yml** file in the directory of your choosing (one titled **nginx-proxy** is a good idea), and copy in the following text:

	version: "3"
	services:
	  nginx-proxy:
	    image: jwilder/nginx-proxy
	    container_name: nginx-proxy
	    ports:
	      - "80:80"
	    volumes:
	      - /var/run/docker.sock:/tmp/docker.sock:ro

	networks:
	  default:
	    external:
	      name: nginx-proxy

And then run the following **docker-compose** command to get started.

	$ docker-compose run -d

###How nginx-proxy works

As you can see from the code in both options, the container listens on port 80 and exposes the same port inside of the container. That allows all incoming traffic to flow though **nginx**.

You might be wondering what the */var/run/docker.sock:/tmp/docker.sock* line accomplishes. Essentially, this gives any container access to the host’s Docker socket, which contains information about a variety of Docker events, such as creating a new container, or shutting one down.

This means that every time you add a container, **nginx-proxy** sees the event through the socket, automatically creates the configuration file needed to route traffic, and restarts **nginx** to make the changes available immediately. nginx-proxy looks for containers with the VIRTUAL_HOSTvariable enabled, so that’s critical to our operations moving forward.

Also important to note is the **--net nginx-proxy** line in the Docker command, and the **networks: default: external: name: nginx-proxy** block in the **docker-compose.yml** file. These establish that all containers will communicate over that Docker network.

##Step 2. Adding a container to the proxy

Now that we have **nginx-proxy** running, we can start adding new containers, which will be automatically picked up and configured for. Because we covered it in the last Docker tutorial, and since it’s an easy implementation to try out, let’s use Wordpress as an example.

###Using docker-compose

Start by creating a ‘docker-compose.yml’ file in an empty directory and copying in the following.

	version: "3"

	services:
	   db:
	     image: mysql:5.7
	     volumes:
	       - db_data:/var/lib/mysql
	     restart: always
	     environment:
	       MYSQL_ROOT_PASSWORD: somewordpress
	       MYSQL_DATABASE: wordpress
	       MYSQL_USER: wordpress
	       MYSQL_PASSWORD: wordpress
	     container_name: wp_test_db

	   wordpress:
	     depends_on:
	       - db
	     image: wordpress:latest
	     expose:
	       - 80
	     restart: always
	     environment:
	       VIRTUAL_HOST: blog.domain.com
	       WORDPRESS_DB_HOST: db:3306
	       WORDPRESS_DB_USER: wordpress
	       WORDPRESS_DB_PASSWORD: wordpress
	     container_name: wp_test
	volumes:
	    db_data:

	networks:
	  default:
	    external:
	      name: nginx-proxy

Again, take note of the expose and environment: **VIRTUAL_HOST** options within the file. Also, the **networks** option at the bottom is necessary to allow this container to “speak” with **nginx-proxy**

In both cases, as long as your DNS is set up to route traffic properly, things should work correctly.

From here, you can start up any number of additional Wordpress site—or any type of service, for that matter—and have them be automatically added to the **nginx-proxy** network.

##Additional resources

Of course, be sure to check out [the extensive documentation](https://github.com/jwilder/nginx-proxy)  for **nginx-proxy** to learn more about how you can configure some more complex proxies, such as those using SSL, with multiple ports, or multiple networks.

We haven’t tested it out yet, but there’s a [“companion”](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)  to **nginx-proxy** called **letsencrypt-nginx-proxy-companion** that allows for the creation/renewal of Let’s Encrypt certificates automatically directly alongside the proxy itself.