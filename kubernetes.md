AWS Secrets Manager:   /flight/prod/jwt-secret = "abc..."
      ↓ External Secrets Operator syncs every 1h
K8s Secret:            jwt-secret.JWT_SECRET = "abc..."
      ↓ Kubernetes mounts
Container env var:     JWT_SECRET=abc...
      ↓ Spring reads
@Value("${jwt.secret}") → abc...
