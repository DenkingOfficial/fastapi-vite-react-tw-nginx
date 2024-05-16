# FastAPI + React + Vite + Tailwind + Nginx

This template provides a minimal setup to get FastAPI backend working with React in Vite with Tailwind CSS, HMR and some ESLint rules inside Docker.

## How to use

### Single app on server

1) Clone repo with all submodules

`git clone --recurse-submodules https://github.com/DenkingOfficial/fastapi-vite-react-tw-nginx`

2) Remove .gitmodules file and .git from every folder to use this as a boilerplate (from cloned repo, from fastapi-backend-template and vite-react-tw-template)

3) Edit docker-compose.yml however you like (change service names, container names, contexts and ports)

4) If you don't need another nginx server for reverse proxying multiple apps run:

`docker compose up --build`

### Multiple apps

If you need another nginx server for reverse proxying multiple apps then there are two options:

1) Using [this](https://github.com/nginx-proxy/nginx-proxy) amazing docker image

2) Using [my ugly implementation](https://github.com/DenkingOfficial/nginx-reverse-proxy) of the same kind of thing

For using any of these you need to create a docker network with any name you prefer:

`docker network create my-network`

### First option

1) Run nginx-proxy container using this command:

`docker run --detach --name nginx-proxy --publish 80:80 --volume /var/run/docker.sock:/tmp/docker.sock:ro nginxproxy/nginx-proxy:1.5`

2) Connect this container to the network you made:

`docker network connect my-network nginx-proxy`

3) Edit docker-compose-proxy.yml file:

    * Change service names, container names and contexts
    * Change network name to name you specified
    * Change exposed port to port where your web will run
    * Change VIRTUAL_HOST environment variable to your server name on which you want your app to run

4) If you have multiple apps using this template repeat editing process for all of them

    **!!!IMPORTANT!!!**

    **NETWORK SHOULD BE THE SAME**

5) For each app run:

`docker compose -f docker-compose-proxy.yml up --build`

### Second option

1) Clone my reverse proxy repo using:

    `git clone https://github.com/DenkingOfficial/nginx-reverse-proxy`

2) Edit docker-compose-my-proxy.yml file:

    * Change service names, container names and contexts
    * Change network name to name you specified
    * Change exposed port to port where your web will run

3) If you have multiple apps using this template repeat editing process for all of them

    **!!!IMPORTANT!!!**

    **NETWORK SHOULD BE THE SAME**


4) Open cloned nginx-reverse-proxy repo. There's a `nginx.conf` file, which you need to edit

    ```
    events {
        worker_connections 1024;
    }

    http {
        # Change app name and container:port
        upstream my-first-app {
            server my-container-name:80;
        }

        server {
            listen 80;
            server_name localhost; # Change server name to your server

            location / {
                proxy_pass http://my-first-app; # Change app name if changed in line 7
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
    }
    ```

    Set your `my-first-app`, `my-container-name` and change server_name from `localhost` to server name you prefer to host app on

    If you want to host multiple apps, you can add more upstreams and servers in the config, for example:

    ```
    events {
        worker_connections 1024;
    }

    http {
        upstream my-first-app {
            server my-first-container-name:80;
        }

        upstream my-second-app {
            server my-second-container-name:8080;
        }

        server {
            listen 80;
            server_name localhost;

            location / {
                proxy_pass http://my-first-app;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }

        server {
            listen 80;
            server_name some-domain.com;

            location / {
                proxy_pass http://my-second-app;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
    }
    ```

3) Set a network you made in **nginx-reverse-proxy** `docker-compose.yml` file:

    ```
    version: "3.8"

    services:
    nginx-reverse-proxy:
        image: nginx:latest
        container_name: "nginx-reverse-proxy"
        ports:
        - "80:80"
        volumes:
        - ./nginx.conf:/etc/nginx/nginx.conf
        networks:
        - my-network

    networks:
    my-network:
        external: true
    ```

4) Run `docker compose up --build` in nginx-reverse-proxy repo

5) Return to template repo and run:

    `docker compose -f docker-compose-my-proxy.yml up --build`

At this point your app/apps should be working