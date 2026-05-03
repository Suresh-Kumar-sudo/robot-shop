FROM nginx:1.25-alpine

RUN adduser -D appuser

COPY entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/entrypoint.sh

COPY default.conf.template /etc/nginx/conf.d/default.conf.template

COPY static /usr/share/nginx/html

ENV CATALOGUE_HOST=catalogue \
    USER_HOST=user \
    CART_HOST=cart \
    SHIPPING_HOST=shipping \
    PAYMENT_HOST=payment \
    RATINGS_HOST=ratings 

RUN chown -R appuser:appuser /etc/nginx/conf.d \
    && chown -R appuser:appuser /var/cache/nginx \
    && chown -R appuser:appuser /var/log/nginx \
    && touch /var/run/nginx.pid \
    && chown appuser:appuser /var/run/nginx.pid 

USER appuser

EXPOSE 8080

HEALTHCHECK CMD wget --no-verbose --tries=1 --spider http://localhost:8080 || exit 1

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]