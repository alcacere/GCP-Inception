# 0004 · Keep NGINX as the Cloud Run ingress container with a PHP-FPM sidecar

**Status:** Proposed (to validate: sidecar startup ordering and FastCGI performance)

## Context
PHP-FPM speaks FastCGI, not HTTP, so something must translate. The load balancer
already terminates TLS and routes paths, so NGINX loses most of its original jobs, but
not the FastCGI translation. Cloud Run supports several containers per instance that
share `localhost`.

## Decision
The `wordpress` Cloud Run service runs two containers: `nginx` (ingress, port 8080,
plain HTTP, FastCGI to `127.0.0.1:9000`) and `wordpress` (PHP-FPM). A container
dependency and a startup probe make NGINX start only after PHP-FPM is ready. TLS and
certificate code is removed from the NGINX image.

## Alternatives considered
- **`php:apache` style image (Apache + mod_php in one container).** Simpler, but
  breaks "one process per container" and drops the original NGINX config.
- **Supervisor running NGINX + PHP-FPM in one container.** An anti-pattern: two
  processes, a process manager as PID 1, and blurred health signals.
- **FrankenPHP or another HTTP-native PHP server.** Interesting, but a bigger change
  than this phase intends.

## Consequences
- The original design principle survives on a serverless platform, which makes a
  good talking point.
- Two containers share the instance's CPU and memory limits, so they are sized together.
- If validation shows problems, fallback: the Apache-based single container, recorded
  in a new ADR that supersedes this one.
