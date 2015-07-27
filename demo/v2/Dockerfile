FROM ubuntu:14.04
MAINTAINER Tran Huu Cuong "tranhuucuong91@gmail.com"

# using apt-cacher-ng proxy for caching deb package
RUN echo 'Acquire::http::Proxy "http://172.17.42.1:3142/";' > /etc/apt/apt.conf.d/01proxy

RUN apt-get update -qq \
    && apt-get install -y nginx

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

