#### Summary
This repo is intended for learning how to connect kafka to opensearch using a data-prepper. Given this it is using self signed keys for openssl. In a prod system this should not be done this way.

#### Notes
* Be aware that this was all done locally, and at this time all my paths are hard coded. So be aware that directly copying this and running the steps will not work copy and paste for you.
* This relys on exposing the host machine's local IP addresss to docker to get around the issue of docker not correclty using localhost. For anyone else's implementation be sure to replace the following IP addresses with your own.
 * pipelines.yaml: bootstrap_server: point this to xxx.xxx.xxx.1:19093 for your hostmachine.
 * docker-compose.yml: KAFKA_ADVERTISED_LISTENERS for the ssl 
connection, point this also the your hostmachine ip.
 * kafka-1.cnf: whatever your hostname ip is needs to be added to the alt_names of the cert configuration BEFORE creating your certs, and keystores.
* For the previous step, there are ways to just have a command grab this for you. I haven't written it yet, when I do I will likey update this.

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


#### How the self signed certs were created for this work
* Make sure to read the notes at the top about IP Addresses to use before running all of these steps.
1. Authority key and cert
```bash
openssl req -new -nodes \
   -x509 \
   -days 365 \
   -newkey rsa:2048 \
   -keyout /home/atracy/repositories/opensearch/secrets/ca.key \
   -out /home/atracy/repositories/opensearch/secrets/ca.crt \
   -config /home/atracy/repositories/opensearch/secrets/ca.cnf
```
2. Create a pem file from cery and key
```bash
cat /home/atracy/repositories/opensearch/secrets/ca.crt /home/atracy/repositories/opensearch/secrets/ca.key > /home/atracy/repositories/opensearch/secrets/ca.pem
```
3. Server key and Authentication certificate
```bash
openssl req -new \
    -newkey rsa:2048 \
    -keyout /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.key \
    -out /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.csr \
    -config /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.cnf \
    -nodes
```
4. Create and sign self signed cert
```bash
openssl x509 -req \
    -days 3650 \
    -in /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.csr \
    -CA /home/atracy/repositories/opensearch/secrets/ca.crt \
    -CAkey /home/atracy/repositories/opensearch/secrets/ca.key \
    -CAcreateserial \
    -out /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.crt \
    -extfile /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.cnf \
    -extensions v3_req
```
5. Convert server cert to pkcs12 format
```bash
openssl pkcs12 -export \
    -in /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.crt \
    -inkey /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.key \
    -chain \
    -CAfile /home/atracy/repositories/opensearch/secrets/ca.pem \
    -name kafka-1 \
    -out /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.p12 \
    -password pass:confluent
```
6. Create the broker's keystore and import keys
```bash
keytool -importkeystore \
    -deststorepass confluent \
    -destkeystore /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka.kafka-1.keystore.pkcs12 \
    -srckeystore /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.p12 \
    -deststoretype PKCS12  \
    -srcstoretype PKCS12 \
    -noprompt \
    -srcstorepass confluent
```
7. Make the client's truststore
```bash
keytool -keystore /home/atracy/repositories/opensearch/secrets/kafka-1-creds/client-creds/kafka.client.truststore.pkcs12 \
    -alias CARoot \
    -import \
    -file /home/atracy/repositories/opensearch/secrets/ca.crt \
    -storepass confluent \
    -noprompt \
    -storetype PKCS12
```
8. Save the credentials for keystore and key
```bash
sudo tee /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1_ssl_key_creds << EOF >/dev/null
confluent
EOF

sudo tee /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1_keystore_creds << EOF >/dev/null
confluent
EOF
```
