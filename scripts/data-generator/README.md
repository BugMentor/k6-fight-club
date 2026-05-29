# Synthetic Data Generator

This utility populates the databases for the K6 Fight Club projects with 1 million synthetic records for consistent stress testing.

## Building the Generator

Navigate to this directory and build the generator:
```bash
cd /Users/matias.magni/Documents/dev/mine/k6-fight-club/scripts/data-generator/
go build -o data-generator main.go
```

## Running the Generator

You must run the generator individually against each project's database. Ensure the respective PostgreSQL instance is running.

### 1. Go GraphQL Project
```bash
./data-generator -db="postgres://payment:payment@localhost:5434/payments?sslmode=disable" -users=1000000 -payments=1000000 -clean=true
```

### 2. Java Spring GraphQL Project
```bash
./data-generator -db="postgres://payment_user:payment_pass@localhost:55432/payment_db?sslmode=disable" -users=1000000 -payments=1000000 -clean=true
```

### 3. Java Spring SOAP Project
```bash
./data-generator -db="postgres://payment_user:payment_pass@localhost:55432/payment_db?sslmode=disable" -users=1000000 -payments=1000000 -clean=true
```

*Note: Ensure credentials and ports match your specific `docker-compose.yaml` configuration for each project.*
