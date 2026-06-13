# Static portfolio served by nginx.
# The repo already contains the built site (index.html, assets/, *.hdr),
# so there's no build step — we just copy it into nginx's web root.
FROM nginx:1.27-alpine

# Custom config: long-cache hashed assets, no-cache index.html.
COPY deploy/nginx.conf /etc/nginx/conf.d/default.conf

# Copy the site. .dockerignore keeps .git/deploy/.github out of the image.
COPY . /usr/share/nginx/html

# nginx:alpine already runs on :80 and has a sensible default CMD.
EXPOSE 80
