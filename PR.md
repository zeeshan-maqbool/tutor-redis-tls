# PR: Native TLS (rediss://) support for Redis

## Title

`feat: add REDIS_USE_TLS setting for native TLS (rediss://) Redis connections`

## Description

### Problem statement

Redis connections in the Open edX templates are hardcoded to the plaintext `redis://`
scheme across:

- all 7 Django `CACHES` entries (`django_redis.cache.RedisCache`),
- the Celery broker (`CELERY_BROKER_TRANSPORT` in `lms.env.yml` / `cms.env.yml`),
- the Event Bus (`EVENT_BUS_REDIS_CONNECTION_URL`).

Unlike MongoDB (`MONGODB_USE_SSL`) and SMTP (`SMTP_USE_TLS`), Redis has no TLS toggle.
This makes it impossible to connect Open edX directly to TLS-only managed Redis services
(Oracle Cloud Infrastructure Cache, AWS ElastiCache with in-transit encryption,
GCP Memorystore, Azure Cache for Redis) without deploying an out-of-band TLS proxy
(e.g. a stunnel sidecar) between Open edX and the cache.

### Summary of changes

| File | Change |
|---|---|
| `tutor/templates/config/defaults.yml` | Add `REDIS_USE_TLS: false` (new setting, default keeps current behaviour) |
| `tutor/templates/apps/openedx/settings/partials/common_all.py` | All 7 `CACHES` "LOCATION" entries: scheme becomes `{{ 'rediss' if REDIS_USE_TLS else 'redis' }}://`; when TLS is enabled, `?ssl_cert_reqs=none` is appended (see note below). `EVENT_BUS_REDIS_CONNECTION_URL`: hardcoded `'redis://@redis:6379/'` replaced with the scheme-switching template plus the existing username/password/host/port/db Jinja logic |
| `tutor/templates/apps/openedx/config/lms.env.yml` | `CELERY_BROKER_TRANSPORT: "{{ 'rediss' if REDIS_USE_TLS else 'redis' }}"` |
| `tutor/templates/apps/openedx/config/cms.env.yml` | Same as lms.env.yml |

### How to use / configuration

```bash
tutor config save \
  --set RUN_REDIS=false \
  --set REDIS_USE_TLS=true \
  --set REDIS_HOST=<your-managed-redis-endpoint> \
  --set REDIS_PORT=6379
```

Note on certificate validation: `ssl_cert_reqs=none` is appended automatically when
`REDIS_USE_TLS=true`, because several managed Redis services expose **IP-based** endpoints
and TLS certificates are issued for hostnames — hostname validation cannot succeed against
an IP. For endpoints reached by a **matching FQDN** with a publicly trusted certificate,
drop the `ssl_cert_reqs=none` suffix from the rendered `LOCATION` (a follow-up could make
this a dedicated `REDIS_SSL_CERT_REQS` setting).

### Testing done

Tested live against an OCI Cache (Redis 7, TLS-only) cluster:

- Celery worker connected directly to `rediss://<endpoint>:6379/0` (broker transport TLS).
- Django cache roundtrip verified through `django_redis` over `rediss://` (set/get OK).
- TLS handshake negotiated TLSv1.3 / AES-256-GCM against the managed endpoint.
- Plaintext `redis://` connections to the same endpoint are rejected by the service
  (TLS-only), confirming the connection is encrypted.
