FROM nginx:stable

# Clear out existing configuration
RUN rm /etc/nginx/conf.d/default.conf

VOLUME /var/log/nginx
VOLUME /run/proxy

COPY template /template
COPY bin /usr/local/bin

ENTRYPOINT ["/usr/local/bin/reverse-proxy"]
