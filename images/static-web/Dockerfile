# Shared runtime for the static sites in the fleet.
#
# Replaces four hand-maintained nginx configs that had drifted into three tiers:
# portfolio and kingdomskids ran non-root on :8080 with a full header set, while
# calendar-event-generator and powerpoint-extractor ran as ROOT on :80 with no
# CSP, no HSTS, no Referrer-Policy and a deprecated X-XSS-Protection.
#
# Pinned on purpose: `nginx-unprivileged:alpine` would let an unattended rebuild
# change the thing serving every site.
FROM nginxinc/nginx-unprivileged:1.29-alpine

USER root
RUN rm -f /etc/nginx/conf.d/default.conf
COPY templates/default.conf.template /etc/nginx/templates/default.conf.template
COPY conf/security-headers.conf /etc/nginx/robyrew/security-headers.conf
RUN chown -R nginx:nginx /etc/nginx/templates
USER nginx

# Restricting the filter is what keeps nginx's own $uri/$host out of envsubst's
# reach — without it any same-named environment variable would be substituted
# into the config before nginx ever parses it.
ENV NGINX_ENVSUBST_FILTER="^(WEB_|CSP_)"
ENV WEB_PORT=8080 \
    WEB_FALLBACK="=404" \
    CSP_SCRIPT_EXTRA="" \
    CSP_STYLE_EXTRA="" \
    CSP_IMG_EXTRA="" \
    CSP_FONT_EXTRA="" \
    CSP_CONNECT_EXTRA=""

EXPOSE 8080

HEALTHCHECK --interval=60s --timeout=10s --start-period=10s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8080/healthz >/dev/null 2>&1 || exit 1
