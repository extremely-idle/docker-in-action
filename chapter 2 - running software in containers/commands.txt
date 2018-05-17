-- 2.1
docker help

docker help cp

-- 2.2.1
- Create a new container in detached mode from latest NGINX image
docker run --detach --name web nginx:latest

- The -d flag is shorthand for --detach
docker run -d --name mailer dockerinaction/ch2_mailer

-- 2.2.2
- Create an interactive container
docker run --interactive --tty --link web:web --name web_test busybox:latest /bin/sh

- Verify container
wget -O - http://web:80

- Interactive container linking web and mailed containers with shorthand flags
docker run -it --name agent --link web:insideweb --link mailer:dockerinaction/ch2_mailer dockerinaction/ch2_agent

-- 2.2.3 Restarting containers
docker restart web

- Getting a list of running containers
docker ps

- Getting logs for a container
docker logs web

- Stopping containers
docker stop web

-- 2.3 PID Namespaces
docker run -d --name namespaceA busybox:latest /bin/sh -c "sleep 30000"
docker run -d --name namespaceB busybox:latest /bin/sh -c "nc -l -p 0.0.0.0:80"

- Grab running processes within a container
docker exec namspaceA ps

- Create a container without it's own PID namespace
docker run --pid host busybox:latest ps

- Create two NGIX process within the same container
docker run -d --name webConflict nginx:latest

docker exec webConflict nginx -g 'daemon off;'

- Create two NGINX processes within different containers
docker run -d --name webA nginx:latest

docker run -d --name webB nginx:latest

-- 2.4.1 Flexible Container Identification
- Create two containers with the same name
docker run -d --name webid nginx

- Rename container and try again
docker rename webid webid-old

- Containers can be found using their unqiue ID assigned by Docker, e.g. 40c2ef96946dc21bb86bd39912f182563d65a04dd02f6e686db56eb866123630
docker exec 40c2ef96946dc21bb86bd39912f182563d65a04dd02f6e686db56eb866123630 ps

- or a shortened version (first 12 characters)
docker stop 40c2ef96946d

- Use docker create to create a container without starting it (docker run creates and starts a container)
docker create nginx

- You can capture the container ID in the shell
CID=$(docker create nginx:latest)
echo $CID

- ...or in a file
docker create --cidfile /tmp/web.cid nginx

- ...or via docker ps
CID=$(docker ps --latest --quiet)
echo $CID

- ...with shortended flag
CID=$(docker ps -l -q)
echo $CID

-- 2.4.2 Container State and Dependencies
- Script using docker run/create
MAILER_CID=$(docker run -d dockerinaction/ch2_mailer)
WEB_CID=$(docker create nginx)

AGENT_CID=$(docker create --link $WEB_CID:insideweb --link $MAILER_CID:insidemailer dockerinaction/ch2_agent)

- To see all containers
docker ps -a

- Starting containers
docker start $AGENT_CID

- Script using docker run
MAILER_CID=$(docker run -d dockerinaction/ch2_mailer)
WEB_CID=$(docker run -d nginx)

AGENT_CID=$(docker run -d --link $WEB_CID:insideweb --link $MAILER_CID:insidemailer dockerinaction/ch2_agent)

-- 2.5 Building Environment-Agnostic Systems
- Create and run a wordpress container
docker run -d --name wp --read-only wordpress:4

docker inspect --format "{{.State.Running}}" wp

docker logs wp

- Wordpress has a dependency on MySQL, so run up a container for that too
docker run -d --name wpdb -e MYSQL_ROOT_PASSWORD=ch2demo mysql:5

- Create a different Wordpress container that uses the MySQL container
docker run -d --name wp2 --link wpdb:mysql -p 80 --read-only wordpress:4

- Start container with specific volumes for read only exceptions
docker run -d --name wp3 --link wpdb:mysql -p 80 -v /run/lock/apache2/ -v /run/apache2/ --read-only wordpress:4

SQL_CID=$(docker create -e MYSQL_ROOT_PASSWORD=ch2demo mysql:5)

docker start $SQL_CID
MAILER_CID=$(docker create dockerinaction/ch2_mailer)
docker start $MAILER_CID

WP_CID=$(docker create --link $SQL_CID:mysql -p 80 -v /run/lock/apache2/ -v /run/apache2/ --read-only wordpress:4)
docker start $WP_CID

AGENT_CID=$(docker create --link $WP_CID:insideweb --link $MAILER_CID:insidemailer dockerinaction/ch2_agent)
docker start $AGENT_CID

- 2.5.2 Environment Variable Injection
- Example of injecting an environment variable into a container
docker run --env MY_ENVIRONMENT_VAR="this is a test" busybox:latest env

docker create --env WORDPRESS_DB_HOST=<dbhost_name>
              --env WORDPRESS_DB_USER=site_admin
              --env WORDPRESS_DB_PASSWORD=MeowMix42
              wordpress:4

-Updated provisioning Script
DB_CID=$(docker run -d -e MYSQL_ROOT_PASSWORD=ch2demo mysql:5)

MAILER_CID=$(docker run -d dockerinaction/ch2_mailer)

if [ ! -n "$CLIENT_ID" ]; then
  echo "Client ID is not set"
  exit 1
fi

WP_CID=$(docker create --link $DB_CID:mysql
                       --name wp_$CLIENT_ID
                       -p 80
                       -v /run/lock/apache2/
                       -v /run/apache2/
                       -e WORDPRESS_DB_NAME=$CLIENT_ID
                       --read-only wordpress:4)
docker start $WP_CID

AGENT_CID=$(docker create --name agent_$CLIENT_ID
                          --link $WP_CID:insideweb
                          --link $MAILER_CID:insidemailer
                          dockerinaction/ch2_agent)
docker start $AGENT_CID

-- 2.6 Building Durable Containers
- 2.6.1 Container that always restarts...
docker run -d --name backoff-detector --restart always busybox date

docker logs -f backoff-detector

docker exec backoff-detector echo Just a Test

-2.6.2 Keeping containers running
docker run -d -p 80:80 --name lamp-test tutum/lamp

docker top lamp-test

docker exec lamp-test ps

docker exec lamp-test kill <PID>

docker run workpress:4 cat /entrypoint.sh

docker run --entrypoint="cat" wordpress:4 /entrypoint.sh

- 2.7 Cleaning up
docker rm <container-name>

- Automatically cleaning up containers when they enter the "Exit" state
docker run --rm
           --name auto-exit-test
           busybox:latest
           echo Hello World

docker rm -vf $(docker ps -a -q)