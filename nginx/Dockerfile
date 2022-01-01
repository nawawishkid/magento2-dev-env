FROM nginx:1.21

COPY etc/nginx.conf /etc/nginx/nginx.conf
COPY etc/vhost.conf /etc/nginx/conf.d/default.conf
COPY etc/certs/ /etc/nginx/ssl/
COPY docker-entrypoint.sh /usr/local/bin/docker-entrypoint

EXPOSE 80 443

ENV UPLOAD_MAX_FILESIZE 64M
ENV SERVER_NAME localhost
ENV FPM_HOST fpm
ENV FPM_PORT 9000
ENV MAGENTO_ROOT /app
ENV MAGENTO_RUN_MODE production
ENV DEBUG false

RUN ["chmod", "+x", "/usr/local/bin/docker-entrypoint"]

ENTRYPOINT ["/usr/local/bin/docker-entrypoint"]

CMD ["nginx", "-g", "daemon off;"]

