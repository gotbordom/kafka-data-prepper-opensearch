This repo is intended for learning how to connect kafka to opensearch using a data-prepper. Given this it is using self signed keys for openssl. In a prod system this should not be done this way.

#### Dependencies
1. docker
2. docker-compose
3. openssl

#### Using this repo
1. Clone the repo: `git clone git@github.com:gotbordom/kafka-data-prepper-opensearch.git`
2. Step into the repo: `cd /path/to/kafka-data-prepper-opensearch/`
3. Follow steps for creating self signed openssl certs, these steps are below
4. Spin up all the containers: `docker compose up`
5. Start a kafka producer in the docker container. If you have kafka locally it can be done there as well, so just adjust the docker command below accordingly.
```bash
// In the docker container
docker exec -it kafka-1 /bin/kafka-console-producer \
    --bootstrap-server localhost:19093 \
    --topic test-topic \
    --producer.config /etc/kafka/secrets/client-creds/client-ssl.properties
```
6. Should see the message written into the producer printed out by the terminal running all the containers.


