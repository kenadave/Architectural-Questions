docker compose up --build
         │
         ├─ reads docker-compose.yml + .env
         │
         ├─ for each service with build::
         │    runs Dockerfile
         │    Stage 1: JDK + Maven → compiles JAR
         │    Stage 2: JRE only   → copies JAR → final image
         │
         ├─ pulls images for: zookeeper, kafka, redis (from Docker Hub)
         │
         ├─ creates flight-net bridge network
         │
         ├─ starts containers in dependency wave order
         │    injects env vars from .env
         │    maps ports to your laptop
         │    connects all to flight-net
         │
         └─ runs health checks every 15s
              services marked healthy → dependents start
              all healthy → system ready

Your laptop:8080 → api-gateway → routes to internal services
                                  all on flight-net, invisible outside
