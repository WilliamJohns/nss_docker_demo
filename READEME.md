### Start the Local Database image
```
docker run --rm \
    -p 5432:5432 \
    -v $(pwd)/.volumes/database/:/var/lib/postgresql/data \
    -e POSTGRES_DB=demo_db \
    -e POSTGRES_USER=demo \
    -e POSTGRES_PASSWORD=VerySecure \
    postgres:13-alpine
```