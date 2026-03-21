# Runner HMAC Auth Removal

HMAC shared secret authentication was removed from the Runner architecture. OpenZiti mTLS is the sole authentication mechanism for Orchestrator ↔ Runner communication.

Affects `agynio/k8s-runner` and `agynio/agents-orchestrator`.
