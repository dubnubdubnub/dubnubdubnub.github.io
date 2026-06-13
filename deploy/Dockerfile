# Static portfolio served by nginx.
# The repo already contains the built site (index.html, assets/, *.hdr),
# so there's no build step — copy it in, lift the nginx config into place,
# and drop deploy/ so the manifests aren't web-served.
FROM nginx:1.27-alpine

COPY . /usr/share/nginx/html
RUN mv /usr/share/nginx/html/deploy/nginx.conf /etc/nginx/conf.d/default.conf \
    && rm -rf /usr/share/nginx/html/deploy

EXPOSE 80
