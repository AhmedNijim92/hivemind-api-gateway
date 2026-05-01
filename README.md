# API Gateway

API Gateway for the HiveMind platform. Routes requests to downstream microservices and handles JWT-based authentication filtering.

## Details

| Property | Value |
|----------|-------|
| **Port** | `8080` |
| **Database** | None |
| **Role** | API Gateway + JWT Filter |

## Build & Run

```bash
# Build
mvn clean package

# Run
java -jar target/*.jar

# Docker
docker build -t hivemind/api-gateway .
```

## Links

- [Main Repository](https://github.com/AhmedNijim92/hivemind-backend)
