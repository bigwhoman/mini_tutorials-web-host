# mini_tutorials-web-host
Tutorial on how to host a service on your server using docker and cloudflare<br>
<b>you have to add your domain to cloudflare before the steps below</b><br>
First, we need to create a <b>cloudflare certificate</b> for our domain, to do this go to : <br>
cloudflare dashboard --> ssl-tls --> client-certificates --> Create Certificate <br>
![image](https://user-images.githubusercontent.com/79264715/233497050-c85bd3a2-f042-4cc8-953f-b977d97e84f5.png)
<br>
Then we need to get a cloudflare API_KEY from cloudflare dashboard : <l>https://dash.cloudflare.com/profile/api-tokens</l>.<br>
After this, go to your server and add the files below
## Setup files
### .env file
```.env
DOMAINNAME_CLOUD_SERVER=example.com
CLOUDFLARE_EMAIL=email@example.com
CLOUDFLARE_API_KEY=YOUR_CLOUDFLARE_API_KEY
```
### docker-compose
```yaml
version: '3.7'

services:
  traefik:
    image: traefik:2.7
    container_name: traefik
    restart: always
    ports:
      - "80:80"
      - "8443:443"
    environment:
      - CF_API_EMAIL=$CLOUDFLARE_EMAIL
      - CF_API_KEY=$CLOUDFLARE_API_KEY
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=$CLOUDFLARE_EMAIL"
      - "--certificatesresolvers.myresolver.acme.storage=acme.json"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./acme.json:/acme.json
    networks:
      - traefik-proxy

  your_service:
    image: your_image
    container_name: your_container_name
    restart: always
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.your_service.rule=Host(`your_service.example.com`)"
      - "traefik.http.routers.your_service.entrypoints=websecure"
      - "traefik.http.routers.your_service.tls.certresolver=cloudflare"
    networks:
      - traefik-proxy

networks:
  traefik-proxy:
    external: true
```
<br>
In the code above, change "your_service.example.com" with your domain which you want the service on<br>
Now run these commands, 

```shell
docker network create traefik-proxy
docker-compose up -d
```

<br>
## Example
As an example, we want to host a simple static website on our domain.<br>
We would need the service of nginx for this task.
### our docker compose

```yaml
version: '3.7'

services:
  traefik:
    image: traefik:2.7
    container_name: traefik
    restart: always
    ports:
      - "80:80"
      - "8443:443"
    environment:
      - CF_API_EMAIL=$CLOUDFLARE_EMAIL
      - CF_API_KEY=$CLOUDFLARE_API_KEY
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=$CLOUDFLARE_EMAIL"
      - "--certificatesresolvers.myresolver.acme.storage=acme.json"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./acme.json:/acme.json
    networks:
      - traefik-proxy
  static_website:
    image: nginx:alpine
    container_name: static_website
    restart: always
    volumes:
      - ./static:/usr/share/nginx/html
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.static_website.rule=Host(`simple.example.com`)"
      - "traefik.http.routers.static_website.entrypoints=websecure"
      - "traefik.http.routers.static_website.tls.certresolver=cloudflare"
    networks:
      - traefik-proxy

networks:
  traefik-proxy:
    external: true
```

Now we create a directory in our projects root directory where our docker-compose.yaml file is located, as you can see, the static website uses nginx to serve our static website. In nginx volumes ./static folder is mapped to /usr/share/nginx/html in the nginx container which means that it should serve ./static whenever you call it.<br>
### static folder
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Minimal Static Website</title>
</head>
<body>
    <h1>Welcome to the Minimal Static Website</h1>
    <p>This is a simple static website hosted using Docker, Docker Compose, and Nginx.</p>
</body>
</html>
```
Now lets test if our project is working, after running the docker compose, if you go to https://simple.example.com:8443, you would see the html.
![image](https://user-images.githubusercontent.com/79264715/233499425-e0441c83-cb08-4555-97bd-fc35baa99f20.png) 
<br>
remember to always add https:// before your domain and notice that the port is 8443, it is because that in our configuration, traefik listens on port 8443 for https requests.
