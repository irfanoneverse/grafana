# games-staging OpenTelemetry Cleanup Guide

This guide removes an old Laravel OpenTelemetry setup from the `games-staging` Laravel EC2 while keeping the Grafana Alloy client stack.

games-staging values:

| Value                     | Setting                            |
| ------------------------- | ---------------------------------- |
| Instance name             | `games-staging`                    |
| Private IP                | `172.31.47.107`                    |
| Public IP                 | `3.0.83.58`                        |
| Laravel app directory     | `/data/laravel`                    |
| Laravel log directory     | `/data/laravel/storage/logs`       |
| Client stack directory    | `/opt/grafana-nodes/client-server` |
| Monitoring hub private IP | `172.31.27.45`                     |

Run these commands on the `games-staging` EC2 instance.

## 1. Stop Grafana Alloy Containers

Stop only the monitoring containers. This does not stop Nginx, PHP-FPM, Laravel, databases, or queues.

```text
cd /opt/grafana-nodes/client-server
docker compose down
docker ps
```

If the Compose directory is unavailable, stop only the known monitoring containers:

```text
docker stop alloy nginx-exporter phpfpm-exporter
docker ps
```

## 2. Check for OpenTelemetry Footprint

Look for old services, containers, and processes:

```text
sudo systemctl list-unit-files | grep -Ei 'otel|opentelemetry|collector'
docker ps --format '{{.Names}} {{.Image}}' | grep -Ei 'otel|opentelemetry|collector'
ps aux | grep -Ei 'otel|opentelemetry|collector' | grep -v grep
```

## 3. Remove OpenTelemetry Packages from Laravel

Run inside the Laravel app directory:

```text
cd /data/laravel
composer remove keepsuit/laravel-opentelemetry open-telemetry/sdk open-telemetry/exporter-otlp
```

Remove stale Laravel cache files:

```text
cd /data/laravel
rm -f bootstrap/cache/config.php
rm -f bootstrap/cache/services.php
rm -f bootstrap/cache/packages.php
rm -f config/opentelemetry.php
php artisan optimize:clear
php artisan optimize
```

If `composer remove` says a package is not installed, continue with the remaining cleanup checks.

## 4. Remove Remaining Code References

Scan for OpenTelemetry references:

```text
cd /data/laravel
grep -r "opentelemetry\|OpenTelemetry\|keepsuit" config/ bootstrap/ app/ 2>/dev/null
```

If middleware such as `app/Http/Middleware/TraceIdMiddleware.php` only existed to inject OpenTelemetry trace IDs, replace it with a safe no-op:

cat > app/Http/Middleware/TraceIdMiddleware.php << 'EOF'

<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class TraceIdMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        return $next($request);
    }
}
EOF

Then scan again:

```text
grep -r "opentelemetry\|OpenTelemetry\|keepsuit" config/ bootstrap/ app/ 2>/dev/null
```

If references remain, remove imports, service providers, middleware registrations, or config entries that point to deleted OpenTelemetry packages.

## 5. Remove `OTEL_*` Variables

Open the Laravel environment file:

```text
nano /data/laravel/.env
```

Remove lines that start with:

```text
OTEL_
```

Also remove OpenTelemetry variables from:

- deployment secrets
- CI variables
- supervisor or process manager configs
- hosting control panel environment settings

## 6. Fix Laravel Permissions

If Composer or cleanup commands were run as root, restore writable directory ownership:

```text
sudo chown -R www-data:www-data /data/laravel/storage
sudo chown -R www-data:www-data /data/laravel/bootstrap/cache
```

Adjust the user and group if the server runs PHP-FPM as a different user.

## 7. Clear Laravel Caches

```text
cd /data/laravel
php artisan optimize:clear
```

If `artisan` throws an OpenTelemetry class error, delete cache files first:

```text
rm -f bootstrap/cache/config.php
rm -f bootstrap/cache/packages.php
rm -f bootstrap/cache/services.php
php artisan optimize:clear
```

## 8. Stop and Disable Old OpenTelemetry Services

Stop known service names:

```text
sudo systemctl stop otelcol otel-collector aws-otel-collector
sudo systemctl disable otelcol otel-collector aws-otel-collector
sudo systemctl daemon-reload
```

If a service name does not exist, systemd will report it. Continue with the next step.

## 9. Uninstall OpenTelemetry Packages

On Ubuntu:

```text
sudo apt update
sudo apt remove -y otelcol otelcol-contrib aws-otel-collector
sudo apt autoremove -y
```

If a package is not installed, continue.

## 10. Delete Leftover OpenTelemetry Files

Review these locations first:

```text
ls -lah /etc/otelcol* /etc/otel-collector* /opt/aws/aws-otel-collector /var/lib/otelcol* /var/log/otel*
```

Delete only OpenTelemetry leftovers:

```text
sudo rm -rf /etc/otelcol* /etc/otel-collector* /opt/aws/aws-otel-collector /var/lib/otelcol* /var/log/otel*
```

## 11. Remove Old OpenTelemetry Docker Containers

List old containers:

```text
docker ps -a --format '{{.ID}} {{.Names}} {{.Image}}' | grep -Ei 'otel|opentelemetry|collector'
```

If any appear, remove only those containers after confirming they are not part of the new client stack:

```text
docker rm CONTAINER_ID
```

Remove old OpenTelemetry images only after containers are gone:

```text
docker images | grep -Ei 'otel|opentelemetry|collector'
docker rmi IMAGE_ID
```

## 12. Restart Grafana Alloy

Start the client monitoring stack again:

```text
cd /opt/grafana-nodes/client-server
docker compose up -d
docker compose ps
docker compose logs --tail=100 alloy
```

Confirm Alloy is configured with the expected values:

```text
grep -nE 'INSTANCE_NAME|ENVIRONMENT|LARAVEL_LOG_DIR|HUB_IP' /opt/grafana-nodes/client-server/.env
docker compose exec alloy env | grep -E 'INSTANCE_NAME|ENVIRONMENT|HUB_IP'
```

Expected games-staging values:

```text
INSTANCE_NAME=games-staging
ENVIRONMENT=staging
LARAVEL_LOG_DIR=/data/laravel/storage/logs
HUB_IP=172.31.27.45
```

Cleanup is complete when:

- no OpenTelemetry service is enabled
- no OpenTelemetry process is running
- no OpenTelemetry Docker container remains
- Laravel no longer references OpenTelemetry packages
- Alloy containers are healthy
- logs and metrics appear in Grafana
