FROM nginx:1.21-alpine

MAINTAINER Marcelo Milhomem <milhomem.is4web@gmail.com>

WORKDIR /usr/share/nginx/html

ADD files/etc /etc

# Default values for supported variables
ENV CLIENT_MAX_BODY_SIZE=1m \
    NGINX_PORT=80 \
    NGINX_HOST=_ \
    STATIC_FILES_ROOT=/usr/share/nginx/html

#
# Generate {filename}.template files into {filename} with envvars (only existing) expanded on every run
#
CMD [ "sh", "-c", "for filename in /etc/nginx/conf.d/*.template; do envsubst \"`env | awk -F = '{printf \" $$%s\", $$1}'`\" < ${filename} > ${filename%.template}; done && nginx -g 'daemon off;'" ]
