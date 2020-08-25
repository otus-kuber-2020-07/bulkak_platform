#https://hub.docker.com/r/bulkak/bulkak-platform-nginx
FROM nginx:latest

EXPOSE 8000

RUN rm -rf /etc/nginx/nginx.conf
COPY src/conf/nginx/nginx.conf /etc/nginx/
RUN mkdir /app
RUN chown -R 1001 /var/cache/nginx && \
    chown -R 1001 /app && \
    touch /var/run/nginx.pid && \
    chown -R 1001 /var/run/nginx.pid


USER 1001
