# Runner HMAC Auth Removal

HMAC shared secret (`DOCKER_RUNNER_SHARED_SECRET`) was removed from the Runner architecture. OpenZiti mTLS is the sole authentication mechanism for Orchestrator ↔ Runner communication.

Affects `agynio/docker-runner` and `agynio/agents-orchestrator`.
