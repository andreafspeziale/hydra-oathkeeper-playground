# Ory Hydra and Oathkeeper local Docker playground

## Resources

- https://www.ory.sh/hydra/docs/ (v1.10)
- https://www.ory.sh/oathkeeper/docs/ (v0.83)
- http://httpbin.org/#/Anything/get_anything__anything_

## Network

Create a Docker network

- `$ docker network create hydra-oathkeeper`

## Hydra

Run the Hydra database

```bash
$ docker run --rm \
  --network hydra-oathkeeper \
  --name hydra-psql \
  -p 5432:5432 \
  -e POSTGRES_USER=hydra \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=hydra \
  -d postgres:9.6
```

Run database migration

```bash
$ docker run -it --rm \
  --network hydra-oathkeeper \
  oryd/hydra:v1.10 \
  migrate sql --yes 'postgres://hydra:secret@hydra-psql:5432/hydra?sslmode=disable'
```

Run the Hydra server mounting the configuration file

```bash
$ docker run --rm \
  --network hydra-oathkeeper\
  --name hydra-server \
  -p 9000:4444 \
  -p 9001:4445 \
  -v $(pwd)/hydra-config.yaml:/hydra-config.yaml \
  oryd/hydra:v1.10 serve all --config /hydra-config.yaml --dangerous-force-http
```

### Create ClientId and ClientSecret

```bash
$ docker run -it --rm \
  --network hydra-oathkeeper\
  oryd/hydra:v1.10 \
  clients create \
    --endpoint http://hydra-server:4445 \
    --id some-consumer \
    --secret some-secret \
    --grant-types client_credentials \
    --response-types token \
    --scope headers
```

## Oathkeeper

Run the Oathkeeper server mounting the configuration and rules file.

- `client-credentials`: enable only the `oauth2_client_credentials` authenticator
  > It seems the default `oauth2_client_credentials` `scope_strategy` is `exact`
- `token-introspection`: enable only the `oauth2_introspection` authenticator

Both `authenticators` are enabled in the main config but I'm experimenting using them one at a time enabling the desired rule

```bash
$ docker run --rm \
  --network hydra-oathkeeper \
  --name oathkeeper-server \
  -p 4455:4455 \
  -p 4456:4456 \
  -v $(pwd)/oathkeeper-config.yaml:/oathkeeper-config.yaml \
  -v $(pwd)/AUTHENTICATOR_PER_RULE_FOLDER/rules.json:/rules.json \
  oryd/oathkeeper:v0.38 \
  --config /oathkeeper-config.yaml \
  serve
```

## TODO

- Move to `docker-compose`
