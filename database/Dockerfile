FROM postgres:11.6-alpine
COPY ./scripts /docker-entrypoint-initdb.d
ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd
