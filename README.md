### Just some commands I have been using

docker compose up
docker compose down
docker compose rm -svf

## Make a topic
# Been using this one 
docker exec -it kafka-1 /bin/kafka-topics \
    --bootstrap-server localhost:19092 \
    --create \
    --topic test-topic \
    --replication-factor 1 --partitions 1

# This one seems to handle replica assignment over just letting it do its own thing? Idk
docker exec -it kafka-1 /bin/kafka-topics \
    --bootstrap-server localhost:19092 \
    --create \
    --topic test-topic \
    --replica-assignment 101

## Delete a topic
docker exec -it kafka-1 /bin/kafka-topics \
    --bootstrap-server localhost:19092 \
    --delete \
    --topic test-topic

## Make a produced within the container on a made topic
# This one connects to the internal plaintext
docker exec -it kafka-1 /bin/kafka-console-producer \
    --bootstrap-server localhost:19092 \
    --topic test-topic

# For ssl
- make sure that the certs are all made, the keystores are made, etc.
docker exec -it kafka-1 /bin/kafka-console-producer  \
    --bootstrap-server localhost:19093    \ 
    --topic test-topic \ 
    --producer.config /etc/kafka/secrets/client-creds/client-ssl.properties

## Make a consumer inside the container
docker exec -it kafka-1 /bin/kafka-console-consumer \
    --bootstrap-server localhost:19093 \
    --topic test-topic \
    --consumer.config /etc/kafka/secrets/client-creds/client-ssl.properties \
    --from-beginning


## Start a container to monitor network traffic ( actually kinda fun )
# Have yet to see any traffic... hmmm.... 
docker run --rm -it --net container:kafka-1  \
    nicolaka/netshoot  \
    tcpdump -c 40 -X port 19092 or port 19093


## Commands used for creating keys. NOTE These are self signed keys. NOT SECURE. Just for learning.
# Authority key and certificate - make sure to already have the ca.cnf in the directory
openssl req -new -nodes \
   -x509 \
   -days 365 \
   -newkey rsa:2048 \
   -keyout /home/atracy/repositories/opensearch/secrets/ca.key \
   -out /home/atracy/repositories/opensearch/secrets/ca.crt \
   -config /home/atracy/repositories/opensearch/secrets/ca.cnf

# Create a PEM file from the above created files
cat /home/atracy/repositories/opensearch/secrets/ca.crt /home/atracy/repositories/opensearch/secrets/ca.key > /home/training/learn-kafka-courses/fund-kafka-security/ca.pem

# Create the server key and auth certificate
openssl req -new \
    -newkey rsa:2048 \
    -keyout /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.key \
    -out /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.csr \
    -config /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.cnf \
    -nodes

# Creating self signed certs, so sign them
openssl x509 -req \
    -days 3650 \
    -in /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.csr \
    -CA /home/atracy/repositories/opensearch/secrets/ca.crt \
    -CAkey /home/atracy/repositories/opensearch/secrets/ca.key \
    -CAcreateserial \
    -out /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.crt \
    -extfile /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.cnf \
    -extensions v3_req

# Convert server certs to pkcs12 format
openssl pkcs12 -export \
    -in /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.crt \
    -inkey /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.key \
    -chain \
    -CAfile /home/atracy/repositories/opensearch/secrets/ca.pem \
    -name kafka-1 \
    -out /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.p12 \
    -password pass:confluent

# Creating the broker's keystore and importing
keytool -importkeystore \
    -deststorepass confluent \
    -destkeystore /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka.kafka-1.keystore.pkcs12 \
    -srckeystore /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1.p12 \
    -deststoretype PKCS12  \
    -srcstoretype PKCS12 \
    -noprompt \
    -srcstorepass confluent


# Making the client keystore and importing it

- This step is
cd /home/atracy/repositories/opensearch/secrets/kafka-1-creds
mkdir client-creds


keytool -keystore /home/training/learn-kafka-courses/fund-kafka-security/client-creds/kafka.client.truststore.pkcs12 \
    -alias CARoot \
    -import \
    -file /home/training/learn-kafka-courses/fund-kafka-security/ca.crt \
    -storepass confluent  \
    -noprompt \
    -storetype PKCS12


# You can do validation, etc
keytool -list -v \
    -keystore /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka.kafka-1.keystore.pkcs12 \
    -storepass confluent

# Save the keys
sudo tee /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1_sslkey_creds << EOF >/dev/null
confluent
EOF

sudo tee /home/atracy/repositories/opensearch/secrets/kafka-1-creds/kafka-1_keystore_creds << EOF >/dev/null
confluent
EOF



## Data-Prepper later for opensearch
# get the image
docker pull opensearchproject/data-prepper:latest

# running the container with simple configuration on the pipeline only
docker run --name data-prepper -p 4900:4900 \
           -v ${PWD}/pipelines.yaml:/usr/share/data-prepper/pipelines/pipelines.yaml \
           opensearchproject/data-prepper:latest

# running the container with server configuration settings as well
docker run --name data-prepper -p 4900:4900 \
           -v ${PWD}/pipelines.yaml:/usr/share/data-prepper/pipelines/pipelines.yaml \
           -v ${PWD}/data-prepper-config-secrets.yaml:/usr/share/data-prepper/config/data-prepper-config.yaml \
           -v ${PWD}/secrets/kafka-1-creds/client-creds/kafka.client.truststore.pkcs12:/usr/share/data-prepper/kafka.client.truststore.pkcs12 \
           opensearchproject/data-prepper:latest

# certs created above don't seem to work. So i want to attempt creating certs how data-prepper recommends, then use those for kafka as well.
docker run --name data-prepper -p 4900:4900 \
           -v ${PWD}/pipelines.yaml:/usr/share/data-prepper/pipelines/pipelines.yaml \
           -v ${PWD}/data-prepper-config.yaml:/usr/share/data-prepper/config/data-prepper-config.yaml \
           -v ${PWD}/data-prepper-secrets/default-keystore.p12:/usr/share/data-prepper/default-keystore.p12 \
           opensearchproject/data-prepper:latest




## Following data-prepper key gen steps then attempting to use that key for server and client
- openssl genrsa -des3 -passout pass:password -out default_key.key 2048
- openssl rsa -passin pass:password -in default_key.key -outform PEM -pubout -out public.pem
- openssl req -new -passin pass:password -key default_key.key -out certificate.csr -subj "/L=test/O=Example Com Inc./OU=Example Com Inc. Root CA/CN=Example Com Inc. Root CA"
- openssl x509 -req -days 3650 -in certificate.csr -passin pass:password -signkey default_key.key -out certificate.crt
- openssl pkcs12 -export -in certificate.crt -passin pass:password -inkey default_key.key -out default-keystore.p12 -passout pass:password

#### Next Dilemma
- When Creating the certs localhost is happy and fine, BUT to get docker to build my data-prepper I need to configure the bootstrap server to be off my hostname IP address. Since I am using an explicit IP I need to add that to the cert as an additional domain.

....
 
damn.

Let's do that then.

- https://www.littlebigextra.com/how-to-add-subject-alt-names-or-multiple-domains-in-a-key-store-and-self-signed-certificate/
- Basically in my initial construction of the certs, etc. Update the kafka-1.cnf file and in the [alt-names] add IP.1=xxx.xxx.xxx.1 where the ip address is your machines localhost. 


