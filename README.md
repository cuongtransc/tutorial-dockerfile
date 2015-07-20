# Introduction to Dockerfile
Based on https://blog.docker.com/2015/04/docker-tutorial-6-dockerfile-part-1/, thanks [John Willis](https://twitter.com/botchagalupe).

<!-- MarkdownTOC -->

- [Dockerfile: step by step](#dockerfile-step-by-step)
    - [v1: Basic command FROM, RUN, CMD](#v1-basic-command-from-run-cmd)
    - [v2: Command COPY and EXPOSE](#v2-command-copy-and-expose)
    - [v3: Command VOLUME](#v3-command-volume)
    - [v4: Final Dockerfile](#v4-final-dockerfile)
- [Experience](#experience)
    - [Best practices for writing Dockerfiles](#best-practices-for-writing-dockerfiles)
    - [Caching everywhere](#caching-everywhere)
    - [Thin vs Flat](#thin-vs-flat)
    - [Using base images?](#using-base-images)

<!-- /MarkdownTOC -->

# Dockerfile: step by step
## v1: Basic command FROM, RUN, CMD

```dockerfile
FROM ubuntu:14.04
MAINTAINER Tran Huu Cuong "tranhuucuong91@gmail.com"

# using apt-cacher-ng proxy for caching deb package
RUN echo 'Acquire::http::Proxy "http://172.17.42.1:3142/";' >> /etc/apt/apt.conf.d/01proxy

RUN apt-get install -y nginx

CMD ["nginx", "-g", "daemon off;"]
```

**Command**
```bash
docker build -t test/nginx .
docker images | head

# run docker container
cid=`docker run -d test/nginx`

# get docker container ip
docker exec $cid ip a
# or
docker inspect --format '{{.NetworkSettings.IPAddress}}' $cid

nip=`docker inspect --format '{{.NetworkSettings.IPAddress}}' $cid`
curl $nip

# clean
docker stop $cid && docker rm $cid
```


**Finding index.html**
```bash
docker run -it --rm test/nginx /bin/sh

find / -name index.html
```

```bash
# result
/usr/share/nginx/html/index.html
/usr/share/doc/adduser/examples/adduser.local.conf.examples/skel.other/index.html
```

## v2: Command COPY and EXPOSE

```dockerfile
FROM ubuntu:14.04
MAINTAINER Tran Huu Cuong "tranhuucuong91@gmail.com"

# using apt-cacher-ng proxy for caching deb package
RUN echo 'Acquire::http::Proxy "http://172.17.42.1:3142/";' >> /etc/apt/apt.conf.d/01proxy

RUN apt-get update -qq \
    && apt-get install -y nginx

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**index.html**
```
<h1>DockerDayVietnam 2015</h1>
Hello everybody!
```


**Command**
```bash
cat index.html

docker build -t test/nginx .
docker images | head

# run docker container
cid=`docker run -d -P test/nginx`

# get docker container ip
nip=`docker inspect --format '{{.NetworkSettings.IPAddress}}' $cid`
curl $nip

# clean
docker stop $cid && docker rm $cid
```


## v3: Command VOLUME
```dockerfile
FROM ubuntu:14.04
MAINTAINER Tran Huu Cuong "tranhuucuong91@gmail.com"

# using apt-cacher-ng proxy for caching deb package
RUN echo 'Acquire::http::Proxy "http://172.17.42.1:3142/";' >> /etc/apt/apt.conf.d/01proxy

RUN apt-get update -qq \
    && apt-get install -y nginx

# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log
RUN ln -sf /dev/stderr /var/log/nginx/error.log

VOLUME ["/usr/share/nginx/html"]
COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

**Command**
```bash
docker build -t test/nginx .

# mount host directory as a data volume
docker run -it --rm -p 80:80 -v $(pwd)/data_web:/usr/share/nginx/html test/nginx
```

## v4: Final Dockerfile
```dockerfile

FROM ubuntu:14.04
MAINTAINER Tran Huu Cuong "tranhuucuong91@gmail.com"

# using apt-cacher-ng proxy for caching deb package
RUN echo 'Acquire::http::Proxy "http://172.17.42.1:3142/";' >> /etc/apt/apt.conf.d/01proxy

COPY build-nginx /tmp/build-nginx
RUN DEBIAN_FRONTEND=noninteractive bash /tmp/build-nginx/build-nginx-ubuntu-14.04.sh

# make utf-8 enabled by default
ENV LANG en_US.utf8

# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log
RUN ln -sf /dev/stderr /var/log/nginx/error.log

# Define working directory
WORKDIR /etc/nginx

# Create mount point
VOLUME ["/etc/nginx/"]

# Expose port
EXPOSE 80 443

COPY ./docker-entrypoint.sh /
RUN chmod 755 /docker-entrypoint.sh
ENTRYPOINT ["/docker-entrypoint.sh"]

CMD ["nginx", "-g", "daemon off;"]
```

**Command**
```bash
docker build -t test/nginx:1.7 .

# run webapp load balancing
docker-compose up
```

**Load balancing: Round Robin**

http://web-counter.coclab.lan/

http://web-counter.coclab.lan/server-name

**Load balancing: Stick**

http://stick.web-counter.coclab.lan/

http://stick.web-counter.coclab.lan/server-name


# Experience
## Best practices for writing Dockerfiles
https://docs.docker.com/articles/dockerfile_best-practices/

## Caching everywhere
- Caching software packages and third-party source code.
- Dockerfile có 1 phiên bản cho lab và 1 phiên cho product.
Với phiên bản cho lab, tối thiểu hóa "thời gian phải chờ".
Ví dụ: thời gian phải chờ khi thực hiện update repo: apt-get update

## Thin vs Flat
**Đóng gói phần mềm: có 2 phong cách.**
- **Thin (like micro design)**: one process per container.
- **Flat (like monolithic design)**: all-in-one, docker container like VM.
=> tùy trường hợp để sử dụng.

## Using base images?
**Base dựa trên image nào?**
Cân bằng giữa **tính tiện dụng** và **tính nhỏ gọn**.
- **Tiện dụng**: dùng ubuntu, có nhiều phần mềm trong repo. Khi run container, có thể execute vào container và có sẵn các phần mềm để hỗ trợ việc xem thông tin, chỉnh sửa.
Ví dụ: như vi editor.
- **Tính nhỏ gọn**: để có được tính nhỏ gọn, cần cắt giảm các gói phần mềm không bắt buộc, làm giảm tính tiện dụng.

```
docker images:
ubuntu              14.04               6d4946999d4f        4 weeks ago         188.3 MB
debian              wheezy              1265e16d0c28        3 months ago        84.98 MB
alpine              latest              31f630c65071        4 weeks ago         5.254 MB
```
```
dpkg -la|wc -l
ubuntu 189
debian 109
```

Có ý kiến biện minh cho việc dùng image base ubuntu: "Tôi tạo ra nhiều image base trên ubuntu. Vì các image được tổ chức theo layer nên `tổng kích thước = size(ubuntu) + sum([size(app) for app in apps])`.

Điều đó đúng. Nhưng:
- Trên thực tế, mỗi người base trên một images khác nhau nên tính tận dụng lại không cao.
- Nếu deploy trên nhiều server chưa có docker images nào, nếu images của mình nhỏ gọn là một điều tốt.

=> lại một lần nữa cần phải cân bằng, tùy trường hợp sử dụng.
Sự vật, hiện tượng có rất nhiều cặp đối lập. Cần biết các mặt đối lập và cân bằng.

