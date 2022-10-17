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

1. Azure/AWS or any Internet accessible VM running Ubuntu with port 443 allowed to Internet.
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

### Get Certificate from Lets Encrypt

1. Configure a domain in DNS (In this example, [grpc-server.yogeshsharma.me]())
2. Please visit [https://certbot.eff.org/](https://certbot.eff.org/) to check for certbot [installation specific to Ubuntu](https://certbot.eff.org/instructions?ws=other&os=ubuntufocal)
3. Run `certbot certonly --standalone` once installed and enter required details to get certificate.
4. `certbot` will download Lets Encrypt signed certificate in `/etc/letsencrypt/live/` or check for output emitted by `certbot`.

Example:

Certificate with Chain: `/etc/letsencrypt/live/grpc-server.yogeshsharma.me/fullchain.pem`
Private Key: `/etc/letsencrypt/live/grpc-server.yogeshsharma.me/privkey.pem`

### Starting Envoy Proxy

1. [Install Envoy on Ubuntu Linux¶](https://www.envoyproxy.io/docs/envoy/latest/start/install#install-envoy-on-ubuntu-linux)
2. SSL enabled config is available in `envoy/envoy-ssl.yaml`

```bash
envoy -c /app/grpc-service/envoy/envoy-ssl.yaml
```

### Configure Cloudflare

1. Go to DNS management of your domain in Cloudflare
2. Enable Orange cloud (proxied) for configured grpc domain.
3. In Network Tab > Enable gRPC.

For more details check [cloudflare documentation](https://support.cloudflare.com/hc/en-us/articles/360050483011-Understanding-Cloudflare-gRPC-support).

## Sending Request and Verifying

1. Clone this repo in your local machine
2. Set Environment variable `GRPC_SERVER_ADDRESS` pointing to cloudflare domain which is proxied to Azure VM
3. Run Client

    ```bash
    cd /app/grpc-service/grpc-app/python
    python3 route_guide_client.py
    ```
   
4. Verify Envoy log


```bash
timestamp=2022-10-17T05:50:44.221Z client.address=49.207.223.245:0 client.local.address=10.1.0.4:443 upstream.cluster=service_grpc_backend upstream.host=127.0.0.1:50051 request.bytes=5 request.duration=2 useragent=grpc-python/1.49.1 grpc-c/27.0.0 (osx; chttp2) authority=grpc-server.yogeshsharma.me request.id=9e6044a6-a6a8-45cd-b54a-88d6a243eddb x-for=49.207.223.245 method=POST path=/routeguide.RouteGuide/GetFeature protocol=HTTP/2 response.bytes=7 status=200
timestamp=2022-10-17T05:50:44.460Z client.address=49.207.223.245:0 client.local.address=10.1.0.4:443 upstream.cluster=service_grpc_backend upstream.host=127.0.0.1:50051 request.bytes=43 request.duration=14 useragent=grpc-python/1.49.1 grpc-c/27.0.0 (osx; chttp2) authority=grpc-server.yogeshsharma.me request.id=a4ccc414-f7a7-47e1-86ad-b54a877a82be x-for=49.207.223.245 method=POST path=/routeguide.RouteGuide/ListFeatures protocol=HTTP/2 response.bytes=5495 status=200
timestamp=2022-10-17T05:50:44.902Z client.address=49.207.223.245:0 client.local.address=10.1.0.4:443 upstream.cluster=service_grpc_backend upstream.host=127.0.0.1:50051 request.bytes=220 request.duration=2 useragent=grpc-python/1.49.1 grpc-c/27.0.0 (osx; chttp2) authority=grpc-server.yogeshsharma.me request.id=376a78b6-16cd-4d1b-98fe-993904e975ec x-for=49.207.223.245 method=POST path=/routeguide.RouteGuide/RecordRoute protocol=HTTP/2 response.bytes=13 status=200
timestamp=2022-10-17T05:50:45.133Z client.address=49.207.223.245:0 client.local.address=10.1.0.4:443 upstream.cluster=service_grpc_backend upstream.host=127.0.0.1:50051 request.bytes=118 request.duration=1 useragent=grpc-python/1.49.1 grpc-c/27.0.0 (osx; chttp2) authority=grpc-server.yogeshsharma.me request.id=5875515c-4117-474b-a8e2-dc7caf58010c x-for=49.207.223.245 method=POST path=/routeguide.RouteGuide/RouteChat protocol=HTTP/2 response.bytes=46 status=200
```