+++
date = '2025-11-25T11:13:31+07:00'
draft = false 
title = 'BuildDocker'
+++

Tạo 1 image nginx tối giản

## Chuẩn bị một thư mục sạch để build
$ mkdir build \
$ cd build

## Lấy mã nguồn nginx
$ git clone --depth=1 --branch=release-1.29.3 https://github.com/nginx/nginx.git

## Thực hiện build theo hướng dẫn
Theo hướng dẫn sẽ dùng môi trường build ubuntu và các phụ thuộc: gcc make libpcre3-dev zlib1g-dev libssl-dev
Tạo môi trường build trong docker ubuntu (Tại sao không ?) \
$ docker run -it --rm --net host -v $PWD/:/root -w /root/ ubuntu bash

Cài đặt các phụ thuộc \
$ apt update \
$ apt install gcc make libpcre3-dev zlib1g-dev libssl-dev

Build theo hướng dẫn: \
$ cd nginx \
$ auto/configure \
$ make \
$ make install \
$ /usr/local/nginx/sbin/nginx

## Kiểm tra các phụ thuộc mà chương trình cần dùng
$ ldd /usr/local/nginx/sbin/nginx

linux-vdso.so.1 (0x00007ffc73be1000) \
libcrypt.so.1 => /lib/x86_64-linux-gnu/libcrypt.so.1 (0x00007d7980d98000) \
libpcre.so.3 => /lib/x86_64-linux-gnu/libpcre.so.3 (0x00007d7980d20000) \
libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007d7980d04000) \
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007d7980af2000) \
/lib64/ld-linux-x86-64.so.2 (0x00007d7980ec4000) \
Cần build các phụ thuộc: libz, libpcre, libcrypt, libc.


## Build thư viện và tạo Dockerfile
$ docker run -it --rm -v $PWD/:/root/ -w /root/ alpine:3.22.2 sh \
$ apk add --no-cache build-base

Thêm vào Dockerfile:
```
# syntax=docker/dockerfile:1.7

############################
# Stage 1: build tiny static HTTP server
############################
FROM alpine:3.22.2 AS nginx-server-builder

WORKDIR /src

RUN apk add --no-cache build-base cmake git
```

## Build libz: https://zlib.net/ 
$ wget https://zlib.net/zlib-1.3.1.tar.gz \
$ tar xvf zlib-1.3.1.tar.gz  \
$ cmake -S zlib-1.3.1 -B build_zlib \
$ make -C build_zlib \
$ make -C build_zlib install \

Thêm vào Dockerfile:
```
RUN wget https://zlib.net/zlib-1.3.1.tar.gz && \
tar xvf zlib-1.3.1.tar.gz && \
cmake -S zlib-1.3.1 -B build_zlib && \
make -C build_zlib && \
make -C build_zlib install
```

## BUild libpcre: https://github.com/PCRE2Project/pcre2.git
$ git clone --depth=1 --branch=pcre2-10.47 https://github.com/PCRE2Project/pcre2.git
$ cmake -S pcre2 -B build_pcre2 \
$ make -C build_pcre2 \
$ make -C build_pcre2 install \

Thêm vào Dockerfile:
```
RUN git clone --depth=1 --branch=pcre2-10.47 https://github.com/PCRE2Project/pcre2.git && \
cmake -S pcre2 -B build_pcre2 && \
make -C build_pcre2 && \
make -C build_pcre2 install
```

## Build libgpg-error
$ wget https://gnupg.org/ftp/gcrypt/gpgrt/libgpg-error-1.56.tar.bz2 \
$ tar xvf libgpg-error-1.56.tar.bz2 \
$ cd libgpg-error-1.56 \
$ ./configure \
$ make \
$ make install \
$ cd - \

Thêm vào Dockerfile:
```
RUN wget https://gnupg.org/ftp/gcrypt/gpgrt/libgpg-error-1.56.tar.bz2 && \
tar xvf libgpg-error-1.56.tar.bz2 && \
cd libgpg-error-1.56 && \
./configure && \
make &&\
make install
```

## Build libcrypt 
$ wget https://gnupg.org/ftp/gcrypt/libgcrypt/libgcrypt-1.11.2.tar.bz2 \
$ tar xvf libgcrypt-1.11.2.tar.bz2 \
$ cd libgcrypt-1.11.2/ \
$ ./configure \
$ make \
$ make install \
$ cd -

Thêm vào Dockerfile:
```
RUN wget https://gnupg.org/ftp/gcrypt/libgcrypt/libgcrypt-1.11.2.tar.bz2 && \
tar xvf libgcrypt-1.11.2.tar.bz2 && \
cd libgcrypt-1.11.2/ && \
./configure && \
make && \
make install
```

## Build nginx:
$ git clone --depth=1 --branch=release-1.29.3 https://github.com/nginx/nginx.git \
$ cd nginx \
$ auto/configure --with-cc-opt='-O2 -static' --with-ld-opt='-static'
    --with-cc-opt='-O2 -g0 -Wp,-D_FORTIFY_SOURCE=2 -fstack-protector-strong --param=ssp-buffer-size=4 -Wformat -Werror=format-security -fPIC'
    --with-ld-opt='-Wl,-z,relro -Wl,-z,now'
$ make \
$ make install \
$ /usr/local/nginx/sbin/nginx \

Thêm vào Dockerfile:
```
RUN git clone --depth=1 --branch=release-1.29.3 https://github.com/nginx/nginx.git && \
cd nginx && \
auto/configure \
--with-zlib=/src/zlib-1.3.1 \
--with-cc-opt='-O2 -g0 -Wp,-D_FORTIFY_SOURCE=2 -fstack-protector-strong --param=ssp-buffer-size=4 -Wformat -Werror=format-security -fPIC' \
--with-ld-opt='-Wl,-z,relro -Wl,-z,now' &&\
make && \
make install
```

# Tạo image cuối 

Thêm vào Dockerfile:
```
############################
# Stage 2: final image
############################
FROM scratch

COPY --from=nginx-server-builder /usr/local/nginx /usr/local/nginx

RUN ls /ust/local/

ENTRYPOINT ["/usr/local/nginx/sbin/nginx"]
```



```
# syntax=docker/dockerfile:1.7

############################
# Stage 1: build tiny static HTTP server
############################
FROM alpine:3.22.2 AS nginx-server-builder

WORKDIR /src

RUN apk add --no-cache build-base cmake git

RUN wget https://zlib.net/zlib-1.3.1.tar.gz && \
tar xvf zlib-1.3.1.tar.gz && \
cmake -S zlib-1.3.1 -B build_zlib && \
make -C build_zlib && \
make -C build_zlib install

RUN git clone --depth=1 --branch=pcre2-10.47 https://github.com/PCRE2Project/pcre2.git && \
cmake -S pcre2 -B build_pcre2 && \
make -C build_pcre2 && \
make -C build_pcre2 install

RUN wget https://gnupg.org/ftp/gcrypt/gpgrt/libgpg-error-1.56.tar.bz2 && \
tar xvf libgpg-error-1.56.tar.bz2 && \
cd libgpg-error-1.56 && \
./configure && \
make && \
make install

RUN wget https://gnupg.org/ftp/gcrypt/libgcrypt/libgcrypt-1.11.2.tar.bz2 && \
tar xvf libgcrypt-1.11.2.tar.bz2 && \
cd libgcrypt-1.11.2/ && \
./configure && \
make && \
make install

RUN git clone --depth=1 --branch=release-1.29.3 https://github.com/nginx/nginx.git && \
cd nginx && \
auto/configure \
--with-zlib=/src/zlib-1.3.1 \
--with-cc-opt='-O2 -g0 -Wp,-D_FORTIFY_SOURCE=2 -fstack-protector-strong --param=ssp-buffer-size=4 -Wformat -Werror=format-security -fPIC' \
--with-ld-opt='-Wl,-z,relro -Wl,-z,now -static' \
--prefix=/root/nginx &&\
make && \
make install

############################
# Stage 2: final image
############################
FROM scratch

WORKDIR /root/

COPY --from=nginx-server-builder /etc/passwd /etc/passwd
COPY --from=nginx-server-builder /etc/group /etc/group
COPY --from=nginx-server-builder /root/nginx  /root/nginx

ENTRYPOINT ["/root/nginx/sbin/nginx", "-g", "daemon off;"]

```
