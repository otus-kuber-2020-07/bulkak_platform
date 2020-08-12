#https://hub.docker.com/r/bulkak/bulkak-platform-nginx
FROM nginx:latest

EXPOSE 8000

RUN rm -rf /etc/nginx/nginx.conf && mkdir /app
COPY src/conf/nginx/nginx.conf /etc/nginx/

ADD src/app /app

WORKDIR /app
USER 1001