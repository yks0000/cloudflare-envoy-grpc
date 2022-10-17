The sample grpc-service is taken from [grpc.io](https://grpc.io/docs/languages/python/basics/)

## Environment Details

1. **Language:** `Python3.8+`
2. **Hosting:** `Azure/AWS or any Internet accessible VM running Ubuntu`
3. **Local Proxy for TLS Termination:** `Envoy 1.18.2`
4. **Edge Proxy:** `Cloudflare Orange Cloud (Free Plan)`
5. **SSL:** `Lets Encrypt`

## High Level Steps

### Starting gRPC Server

1. Clone this repo on Azure/AWS or any Internet accessible VM running Ubuntu.
2. Install Python3.8+ on Azure VM
3. Install required python package
4. Build gRPC App
5. Start gRPC server

### Starting Envoy Proxy

1. Download Trusted Certificate from Lets Encrypt. This is required as Cloudflare can only accept traffic on 443 with Trusted CA. Read [here for details](https://support.cloudflare.com/hc/en-us/articles/360050483011-Understanding-Cloudflare-gRPC-support)
2. Create Envoy config
3. Start Envoy Proxy

### Configure Cloudflare

As we are using Free plan, there is no option to have custom origin. As an alternative, we need to have a domain under Cloudflare DNS. For this PoC, we already have a domain managed by Cloudflare i.e. [yogeshsharma.me](https://yogeshsharma.me). 

1. Create a sub-domain and proxy it through cloudflare orange cloud.
2. Enable gRPC routing on CloudFlare.

## Detailed Steps

### Pre-requisite

1. Azure/AWS or any Internet accessible VM running Ubuntu with port 80, 443 allowed to Internet.
2. Cloudflare account with a domain under its management.
3. Understanding of basic Linux commands
4. Python3.8+ installed on VM. If Ubuntu, you can use `deadsnakes` repo for Installation. Refer steps as available on [GitHub gist](https://gist.github.com/plembo/6bc141a150cff0369574ce0b0a92f5e7)


### Starting gRPC Server

After login to VM, 

1. Create a directory and switch to it
    ```bash
    mkdir -p /app/
    cd /app/
    ```

2. Clone [this](https://github.com/yks0000/cloudflare-envoy-grpc) repo
    ```bash
     git clone https://github.com/yks0000/cloudflare-envoy-grpc.git /app/grpc-service
    ```
3. Installed required packages

    ```bash
    pip3 install grpcio
    pip3 install grpcio-tools
    ```
4. Build Proto

    ```bash
    cd /app/grpc-service/grpc-app/python/route_guide
    python3 -m grpc_tools.protoc -I../../protos --python_out=. --grpc_python_out=. ../../protos/route_guide.proto
    ```
   
   The generated code files are called `route_guide_pb2.py` and `route_guide_pb2_grpc.py`

5. Run gRPC Server on Azure VM

    ```bash
    cd /app/grpc-service/grpc-app/python/route_guide
    python3 route_guide_server.py
    ```
   
6. Test gRPC locally from Azure VM. (Run only once from another terminal)

    ```bash
    cd /app/grpc-service/grpc-app/python/route_guide
    python3 route_guide_local_client.py
    ```
   It will return a Unary and Stream gRPC output from gRPC server.

### Get Certificate from Lets Encrypt

1. Configure a domain in DNS (In this example, [grpc-server.yogeshsharma.me]())

   ```bash
   $ nslookup grpc-server.yogeshsharma.me
   Server:		127.0.0.53
   Address:	127.0.0.53#53
   
   Non-authoritative answer:
   Name:	grpc-server.yogeshsharma.me
   Address: 104.21.75.47
   Name:	grpc-server.yogeshsharma.me
   Address: 172.67.213.195
   Name:	grpc-server.yogeshsharma.me
   Address: 2606:4700:3030::ac43:d5c3
   Name:	grpc-server.yogeshsharma.me
   Address: 2606:4700:3032::6815:4b2f
   ```

2. Please visit [https://certbot.eff.org/](https://certbot.eff.org/) to check for certbot [installation specific to Ubuntu](https://certbot.eff.org/instructions?ws=other&os=ubuntufocal)
3. Run `certbot certonly --standalone` once installed and enter required details to get certificate for your domain. Make sure port 80 is allowed on your VM to be accessible from Internet.
4. `certbot` will download Lets Encrypt signed certificate in `/etc/letsencrypt/live/` or check for output emitted by `certbot`.

Example:

   ```bash
   $ certbot certonly --standalone
   Saving debug log to /var/log/letsencrypt/letsencrypt.log
   Enter email address (used for urgent renewal and security notices)
    (Enter 'c' to cancel): xxxxxxx@xxxxxxxx.com
   
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Please read the Terms of Service at
   https://letsencrypt.org/documents/LE-SA-v1.3-September-21-2022.pdf. You must
   agree in order to register with the ACME server. Do you agree?
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   (Y)es/(N)o: Y
   
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Would you be willing, once your first certificate is successfully issued, to
   share your email address with the Electronic Frontier Foundation, a founding
   partner of the Let's Encrypt project and the non-profit organization that
   develops Certbot? We'd like to send you email about our work encrypting the web,
   EFF news, campaigns, and ways to support digital freedom.
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   (Y)es/(N)o: N
   Account registered.
   Please enter the domain name(s) you would like on your certificate (comma and/or
   space separated) (Enter 'c' to cancel): grpc-server.yogeshsharma.me
   Requesting a certificate for grpc-server.yogeshsharma.me
   
   Successfully received certificate.
   Certificate is saved at: /etc/letsencrypt/live/grpc-server.yogeshsharma.me/fullchain.pem
   Key is saved at:         /etc/letsencrypt/live/grpc-server.yogeshsharma.me/privkey.pem
   This certificate expires on 2023-01-15.
   These files will be updated when the certificate renews.
   Certbot has set up a scheduled task to automatically renew this certificate in the background.
   
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   If you like Certbot, please consider supporting our work by:
    * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
    * Donating to EFF:                    https://eff.org/donate-le
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   ```

### Starting Envoy Proxy

1. [Install Envoy on Ubuntu Linux¶](https://www.envoyproxy.io/docs/envoy/latest/start/install#install-envoy-on-ubuntu-linux)
2. SSL enabled config is available in `envoy/envoy-ssl.yaml`. Start envoy with command as below:

```bash
envoy -c /app/grpc-service/envoy/envoy-ssl.yaml
```

### Configure Cloudflare

1. Go to DNS management of your domain in Cloudflare
2. Click edit for sub-domain that your have created for grpc and enable Orange cloud (proxy).
3. In Network Tab > Enable gRPC.

For more details check [cloudflare documentation](https://support.cloudflare.com/hc/en-us/articles/360050483011-Understanding-Cloudflare-gRPC-support).

## Sending Request and Verifying

1. Clone this repo in your local machine
2. Set Environment variable `GRPC_SERVER_ADDRESS` pointing to cloudflare domain which is proxied to Azure VM
3. Run Client. Make sure `grpcio` and `grpcio-tools` packages are installed.

    ```bash
    # cd to cloned path 
    cd grpc-app/python/route_guide/
    python3 route_guide_client.py
    ```
   
4. Verify Envoy log


```bash
timestamp=2022-10-17T05:50:44.221Z client.address=49.207.223.245:0 client.local.address=10.1.0.4:443 upstream.cluster=service_grpc_backend upstream.host=127.0.0.1:50051 request.bytes=5 request.duration=2 useragent=grpc-python/1.49.1 grpc-c/27.0.0 (osx; chttp2) authority=grpc-server.yogeshsharma.me request.id=9e6044a6-a6a8-45cd-b54a-88d6a243eddb x-for=49.207.223.245 method=POST path=/routeguide.RouteGuide/GetFeature protocol=HTTP/2 response.bytes=7 status=200
timestamp=2022-10-17T05:50:44.460Z client.address=49.207.223.245:0 client.local.address=10.1.0.4:443 upstream.cluster=service_grpc_backend upstream.host=127.0.0.1:50051 request.bytes=43 request.duration=14 useragent=grpc-python/1.49.1 grpc-c/27.0.0 (osx; chttp2) authority=grpc-server.yogeshsharma.me request.id=a4ccc414-f7a7-47e1-86ad-b54a877a82be x-for=49.207.223.245 method=POST path=/routeguide.RouteGuide/ListFeatures protocol=HTTP/2 response.bytes=5495 status=200
timestamp=2022-10-17T05:50:44.902Z client.address=49.207.223.245:0 client.local.address=10.1.0.4:443 upstream.cluster=service_grpc_backend upstream.host=127.0.0.1:50051 request.bytes=220 request.duration=2 useragent=grpc-python/1.49.1 grpc-c/27.0.0 (osx; chttp2) authority=grpc-server.yogeshsharma.me request.id=376a78b6-16cd-4d1b-98fe-993904e975ec x-for=49.207.223.245 method=POST path=/routeguide.RouteGuide/RecordRoute protocol=HTTP/2 response.bytes=13 status=200
timestamp=2022-10-17T05:50:45.133Z client.address=49.207.223.245:0 client.local.address=10.1.0.4:443 upstream.cluster=service_grpc_backend upstream.host=127.0.0.1:50051 request.bytes=118 request.duration=1 useragent=grpc-python/1.49.1 grpc-c/27.0.0 (osx; chttp2) authority=grpc-server.yogeshsharma.me request.id=5875515c-4117-474b-a8e2-dc7caf58010c x-for=49.207.223.245 method=POST path=/routeguide.RouteGuide/RouteChat protocol=HTTP/2 response.bytes=46 status=200
```

We can see useragent as `grpc-python` with protocol `http2`

## Ensure CloudFlare is sending gRPC

We are getting a response from gRPC server which is proxied via CloudFlare and Envoy. To ensure Cloudflare is actually proxying, we will disable gRPC in `Network` tab to check if our gRPC python client failed to connect.

In CloudFlare, under domain management > Network > Switch off `gRPC`

Run Client again and expect it to fail

```bash
grpc._channel._InactiveRpcError: <_InactiveRpcError of RPC that terminated with:
	status = StatusCode.UNKNOWN
	details = "Stream removed"
	debug_error_string = "UNKNOWN:Error received from peer ipv4:172.67.213.195:443 {grpc_message:"Stream removed", grpc_status:2, created_time:"2022-10-17T13:09:46.149885+05:30"}"
```


Enable gRPC in Cloudflare again and this time it client should connect successfully.
