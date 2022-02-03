FROM postgres:11.6-alpine

ENV POSTGRES_DB=tp01 \
    POSTGRES_USER=swano \
    POSTGRES_PASSWORD=123456789

COPY 01-CreateSchema.sql /docker-entrypoint-initdb.d/
COPY 02-InsertData.sql /docker-entrypoint-initdb.d/

