# Runner HMAC Auth Removal

Architecture specifies OpenZiti as the sole authentication mechanism for Runner communication. HMAC shared secret (`DOCKER_RUNNER_SHARED_SECRET`) should be removed.

## Affected services

### agynio/platform (docker-runner)

- Remove HMAC authentication from the Runner gRPC server
- Remove `DOCKER_RUNNER_SHARED_SECRET` environment variable and configuration

### agynio/agents-orchestrator

- Remove HMAC credential configuration for Runner calls
- Remove any HMAC interceptor/middleware from Runner gRPC client
