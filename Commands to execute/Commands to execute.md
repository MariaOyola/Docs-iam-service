# 1. Base de Datos 
------------------------------------


### Entrar a docker-infra
> cd C:\iam-service\docker-infra
----------------

### Reiniciar desde cero
> docker compose down -v
--------

### Levantar PostgreSQL
> docker compose --env-file .env.develop up postgres -d

-------------
### Ejecutar Liquibase
> docker compose --env-file .env.develop --profile tooling run --rm liquibase-iam update

------------------

## QA
> docker compose --env-file .env.qa up postgres -d

>docker compose --env-file .env.qa --profile tooling run --rm liquibase-iam update

--------------------

## STAGING
> docker compose --env-file .env.staging up postgres -d

> docker compose --env-file .env.staging --profile tooling run --rm liquibase-iam update

--------------

## MAIN
> docker compose --env-file .env.main up postgres -d

> docker compose --env-file .env.main --profile tooling run --rm liquibase-iam update

-------------------------------------------------

# 2. Backed 
