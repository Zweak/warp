Forked from [cmj2002/warp-docker](https://github.com/cmj2002/warp-docker)

# warp-docker

Run official [Cloudflare WARP](https://1.1.1.1/) client in Docker.

> [!NOTE]
> Cannot guarantee that the [GOST](https://github.com/ginuerzh/gost) and WARP client contained in the image are the latest versions. If necessary, please [build your own image](#build).

## Usage

### Start the container

To run the WARP client in Docker, just write the following content to `docker-compose.yml` and run `docker-compose up -d`.

```yaml
version: '3'

services:
  warp:
    image: zweak/warp
    container_name: warp
    restart: always
    ports:
      - '1080:1080'
    environment:
      - WARP_SLEEP=2
      # - WARP_LICENSE_KEY= # optional
    cap_add:
      - NET_ADMIN
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=0
      - net.ipv4.conf.all.src_valid_mark=1
    volumes:
      - ./data:/var/lib/cloudflare-warp

networks:
  warp1:
```

Try it out to see if it works:

```bash
curl --socks5 127.0.0.1:1080 https://cloudflare.com/cdn-cgi/trace
```

If the output contains `warp=on` or `warp=plus`, the container is working properly. If the output contains `warp=off`, it means that the container failed to connect to the WARP service.

### Configuration

You can configure the container through the following environment variables:
  
- `WARP_SLEEP`: The time to wait for the WARP daemon to start, in seconds. The default is 2 seconds. If the time is too short, it may cause the WARP daemon to not start before using the proxy, resulting in the proxy not working properly. If the time is too long, it may cause the container to take too long to start. If your server has poor performance, you can increase this value appropriately.

- `WARP_LICENSE_KEY`: The license key of the WARP client, which is optional. If you have subscribed to WARP+ service, you can fill in the key in this environment variable. If you have not subscribed to WARP+ service, you can ignore this environment variable.
  
Data persistence: Use the host volume `./data` to persist the data of the WARP client. You can change the location of this directory or use other types of volumes. If you modify the `WARP_LICENSE_KEY`, please delete the `./data` directory so that the client can detect and register again.

### Change proxy type

The container uses [GOST](https://github.com/ginuerzh/gost) to provide proxy, where the environment variable `GOST_ARGS` is used to pass parameters to GOST. The default is `-L :1080`, that is, to listen on port 1080 in the container at the same time through HTTP and SOCKS5 protocols. If you want to have UDP support or use advanced features provided by other protocols, you can modify this parameter. For more information, refer to [GOST documentation](https://v2.gost.run/en/).

If you modify the port number, you may also need to modify the port mapping in the `docker-compose.yml`.

### Health check

The health check of the container will verify if the WARP client inside the container is working properly. If the check fails, the container will automatically restart. Specifically, 15 seconds after starting, a check will be performed every 15 seconds. If the inspection fails for 3 consecutive times, the container will be marked as unhealthy and trigger an automatic restart.

```Dockerfile
HEALTHCHECK --interval=15s --timeout=5s --start-period=30s --retries=3 \
  CMD curl -fsS "https://cloudflare.com/cdn-cgi/trace" | grep -qE "warp=(plus|on)" || exit 1
```

If you don't want the container to restart automatically, you can remove `restart: always` from the `docker-compose.yml`. You can also modify the parameters of the health check through the `docker-compose.yml`.

### Use other versions

The tag of docker image is in the format of `{WARP_VERSION}-{GOST_VERSION}`, for example, `2023.10.120-2.11.5` means that the WARP client version is `2023.10.120` and the GOST version is `2.11.5`. If you want to use other versions, you can specify the tag in the `docker-compose.yml`.

You can also use the `latest` tag to use the latest version of the image.

> [!NOTE]
> Not all version combinations are available. Do check [the list of tags in Docker Hub](https://hub.docker.com/r/caomingjun/warp/tags) before you use one. If the version you want is not available, you can [build your own image](#build).

## Build

You can use Github Actions to build the image yourself.

1. Fork this repository.
2. Create necessary variables and secrets in the repository settings:
  1. variable `REGISTRY`: for example, `docker.io` (Docker Hub)
  2. variable `IMAGE_NAME`: for example, `caomingjun/warp`
  3. variable `DOCKER_USERNAME`: for example, `caomingjun`
  4. secret `DOCKER_PASSWORD`: generate a token in Docker Hub and fill in the token
3. Manually trigger the workflow `Build and push image` in the Actions tab.

This will build the image with the latest version of WARP client and GOST and push it to the specified registry. You can also specify the version of GOST by giving input to the workflow. Building image with custom WARP client version is not supported yet.

If you want to build the image locally, you can use [`.github/workflows/build-publish.yml`](.github/workflows/build-publish.yml) as a reference.

## Further reading

Read in my [blog post](https://blog.caomingjun.com/run-cloudflare-warp-in-docker/en/#How-it-works).