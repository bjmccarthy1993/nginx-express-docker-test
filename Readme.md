# Test project to learn NginX and play around with different configurations

## Docker
* Start Docker desktop on your machine
* Execute the following command while in the working directory of the project:
```bash
docker build -t myapp:1.0 .
```
This will run the dockerfile in the current project (tagging it with the name 'myapp' and tag '1.0'), and create an image from it.


To create a container from the image, you can run:
```bash
docker run -p 8080:3000 myapp:1.0
```

* the paramater after '-p' is mapping ports from container to host -> host:container


You can stop the container by first running ```docker ps``` to get the container ids of the running containers,
and then running ```docker stop <id>``` for the id of the container you wish to stop


## Docker Compose
This is a tool to simplify the process of defining and running multiple containers.

We can run:
```bash
docker compose build
```
* This will simply build the containers based on the docker-compose configuration

We can run:
```bash
docker compose up --build -d 
```

* '--build' will build the containers before starting them, and '-d' will run them in detached mode (in the background)

You can run:
```bash
docker compose down
```
to stop all running containers (from the docker-compose file in this directory)



## Nginx
* Note that the conf file in this directory needs to be copied into wherever nginx is running, in order for it to actually be what nginx uses. It is simply here to keep it in source control with the rest of the project files.
* Note that in this tutorial, I downloaded nginx (for windows) from the nginx website, copied the folder (after unzipping) to my program-files folder. (I probably should have put it in 'Program Files x86' since it was 32 bit, but it worked all the same) and then added that folder to my path environment variable (specifically user environment variables).
* The windows nginx folder structure is different to the linux folder structure. When configuring the container (which is linux-based) I had to do things a bit differently when configuring nginx.conf.

### nginx.conf
#### worker_processes
* sets the number of worker processes that nginx should start with
* Instead of using a new process for every incoming connection, nginx uses worker processes that handle many connections
using a single-threaded event loop. Each worker process runs independently and handles its own set of connections.
* The number of worker processes directy affects how well nginx can handle traffic.
    * This should be adjusted according to the server's hardware (cpu cores) and expected traffic load.
* For development, 1 worker process is enough. But in production, it's recommended to set the worker process number
to match the number of cpu cores available on the server where nginx is running.
    * This allows nginx to efficiently uses all cores.
* You can also set the value to 'auto', which will tell nginx to automatically detect how many cpu cores are available
on the server, and to start the same number of worker nodes.

#### worker_connections
* This sets the number of simultaneous connections that can be open per worker process.
* The higher this number, the more memory is used.
    * The actual number of simultaneous connections cannot exceed the current limit on the max number of open files.


#### Server block
* Defines how nginx should handle request for a particular domain or ip address.
* Defines where nginx is listening (which port) (How to listen for connections)
* Defines where it is available to accept the request (which domain or subdomain the configuration applies to)
* Defines what it should do with those requests (how to route the requests)

listen -> The ip address and port on which the server will accept requests
server_name -> Which domain or ip address this server block should respond too
location -> The root (/) URL, will apply to all requests unless more specific location blocks are defined.
proxy_pass -> Tells Nginx to "pass" the request to another server, making it act as a reverse proxy.

The "Upstreams" are the servers that Nginx forwards the requests to. (note that in our app, we've given the upstream a name of 'nodejs_cluster')
* "upstream" name is based on the flow of data
* Upstream servers -> refers to the traffic going from a client toward the source or higher level infrastructure, in this case the application server
* Downstream servers -> Traffic going back down towards the client.

Common headers that can be forwarded to the server from nginx:
```conf
location / {
    # Pass client information to the backend
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port $server_port;

    # Forward client browser and session information
    proxy_set_header User-Agent $http_user_agent;
    proxy_set_header Cookie $http_cookie;
    proxy_set_header Accept-Language $http_accept_language;
    proxy_set_header Referer $http_referer;

    # Forward client's authorization data
    proxy_set_header Authorization $http_authorization;

    # Custom headers (optional)
    proxy_set_header X-Custom-Header "MyAppSpecificValue";
}
```

In order for the browser to see the types of the files being sent back, we set ```include mime.types```. mime.types is a file in nginx that maps file extensions to mime types so they can be set as the content type in http headers, so the browser can know how to render them.

#### Load balancing algorithms
This tells NginX which algorithm to use when deciding which server to send request to. The default is round-robin -> send requests in a sequence.

Other choices are:
* Round Robin
* Lease Connections
* IP Hash (good for session persistence)
* Weighted Round Robin
* Weighted Least Connections


### Running nginx
Save your new config (in the nginx program files directory), and in the command line run:
```bash
start nginx
```
This will allow you to start nginx without your terminal hanging.
To stop nginx, you can use ```nginx -s stop``` and to reload the configuration without killing the server, you can use ```nginx -s reload```


### Setting up Https
We need an SSL (or TLS) Certificate.
* SSL certificates enable encryption by using public-key cryptography
* The certificate 'validates' the server
* When a user connects to a website via HTTPS, the web server provides its SSL certificate, which contains a public key.
* The client (browser) can then use this key to establish a secure, encrypted session with the server.

We can generate a self-signed certificate.
* Useful for testing or internal sites, but not recommended for production.
* We should obtain a Certificate Authority(CA)-signed certificate.
    * This is a certificate issues and authenticated by a trusted certificate authority.
    * The Certificate Authority verifies the identity of the organization requesting the certificate.
    * This will be trusted by the browser and will not generate any warnings.

#### using openssl to generate a certificate
openssl = and open-source tool used to generate keys, certificates, and manage secure connections.

* We will keep this in a folder in this project, but in the tutorial she created a folder in the home directory.

We use the following command to generate the certificates:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx-selfsigned.key -out nginx-selfsigned.crt
```
* openssl req -> initiates a certificate request generation process
* -x509 -> tells OpenSSL to output a certificate in this standard format
* -nodes -> tells OpenSSL not to encrypt the private key with a passphrase
* -days 365 -> specifies the validity period of the certificate, in this case 1 year
* -newkey rsa:2048 -> Creates a 2048-bit RSA key pair, RSA is the most common public-key encryption algorithm
* -keyout nginx-selfsigned.key -> specifies the output for the generated private key, in this case "nginx-selfsigned.key"
* -out nginx-selfsigned.crt -> specifies the output file for the self-signed certificate, in this case "nginx-selfsigned.crt". This contains the public key.

Note that I used Ubuntu (WSL) to run openssl since windows doesn't have it installed, and it was quickest to just run Windows Subsystem for Linux.

Once you run the command, the following prompts were given and I gave the following answers (note that you can leave some fields blank so I did):
* Country Name (2 letter code) [AU]: ----> NZ
* State or Province Name (full name) [Some-State]: ------> 'blank'
* Locality Name (eg, city) []: -----> Auckland
* Organization Name (eg, company) [Internet Widgits Pty Ltd]: ------> Development
* Organization Unit Name (eg, section) []: -------> 'blank'
* Common Name (eg, server FQDN or YOUR name) []: -------> 'blank'
* Email Address []: -------> 'blank'


#### Configure HTTPS - for NGINX server
We change the 'listen' line from 8080 to 443 and add ssl, then we add:
* ```ssl_certificate <path-to-certificate>```
* ```ssl_certificate_key <path-to-key>```

Note that when adding the filepath in windows, you will need to either add a second '\' to your backslashes, or just use forwardslashes.

#### Redirect http to https
You can add a second server context to the config file, and add a location which returns a 301 redirect response.


Note: 443 is the default for https, which is why you don't need to specifiy the port in your redirect. 80 is the default for http, so if we specify to listen on port 80 instead of 8080, we won't need to specify the port when we enter the url into the browser.



## Running Nginx in a container
* run ```docker pull nginx``` to pull the latest nginx image from dockerhub.

* Create a dockerfile that uses the nginx image as a base, and copy our files in.

* Create a common network so that the nginx container can talk to the other containers:
```bash
docker network create <chosen-network-name>
```

We can then run our other containers in this network like so:
```bash
docker run -p 3001:3000 --name app1 -d --network <chosen-network-name> simple-web-app
```

However we can just add it to the docker-compose file to simplify things for our node servers. There is a networks section that we can add for each container declaration.


NOTE: Within the docker network, we need to make sure nginx isn't trying to get content from the express containers using their mapped port (eg if the container is listening on 3000 and it is being mapped to 3001, make sure the nginx conf points to 3000 since they are within the same network).

Note: Http to Https redirection stopped working in the containerized solution. There seemed to be an issue with port mapping and the redirect. The only way I could get the redirect working was by mapping port 443 in the container to port 443 on my machine, which isn't exactly what I wanted to do. But until I have a better understanding of nginx, it will do for this project.