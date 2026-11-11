# Shared runtime for the static sites in the fleet.
#
# Replaces four hand-maintained nginx configs that had drifted into three tiers:
# portfolio and kingdomskids ran non-root on :8080 with a full header set, while
# calendar-event-generator and powerpoint-extractor ran as ROOT on :80 with no
# CSP, no HSTS, no Referrer-Policy and a deprecated X-XSS-Protection.
#
# Pinned on purpose: `nginx-unprivileged:alpine` would let an unattended rebuild
# change the thing serving every site.
FROM nginxinc/nginx-unprivileged:1.30-alpine

USER root
# Alpine's security fixes for the packages the base ships with. The upstream tag is rebuilt on its own schedule, so
# without this a rebuild could keep a fixed CVE: bibletype's image scan rejected libxslt 1.1.43-r3 (CVE-2026-87636,
# CVE-2026-9947, HIGH; fixed in -r4) on 2026-11-11 while nginx-unprivileged:1.30-alpine still carried it.
# `apk upgrade` stays within the Alpine release the base pins, so it moves patch versions only.
RUN apk upgrade --no-cache
RUN rm -f /etc/nginx/conf.d/default.conf
COPY templates/default.conf.template /etc/nginx/templates/default.conf.template
COPY conf/security-headers.conf /etc/nginx/robyrew/security-headers.conf
# Apps layer their own location blocks here. Created empty so the glob include
# does not fail when an app adds nothing.
RUN mkdir -p /etc/nginx/robyrew/app.d && touch /etc/nginx/robyrew/app.d/.keep
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

# No Docker healthcheck. Docker runs one as `runc exec` inside the container,
# and on vps01 every exec killed at its timeout left a pid charge and ~130 KB of
# kernel memory behind (AdGuard, Sep 2026: 8,189 pids charged to a one-process
# container, SUnreclaim 1.25 GB on a 2 GB host). That host's rule since then is
# no exec-based healthcheck on a long-running container. NONE rather than no
# instruction, so a healthcheck added by a later nginx-unprivileged base is
# overridden too; an app can still declare its own after FROM. /healthz stays in
# the server block for probes from outside the container.
HEALTHCHECK NONE
