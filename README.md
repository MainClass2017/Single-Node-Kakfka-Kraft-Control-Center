# Single-Node-Kakfka-Kraft-Control-Center

Step 1: Install Java 17
Kafka 8.x runs best on Java 17.

2)	Download tar
curl -O https://packages.confluent.io/archive/8.2/confluent-8.2.0.tar.gz

3)3)	Extract TAR files run this command 
tar xzf confluent-8.2.0.tar.gz



Bash
apt update
apt install -y openjdk-17-jdk
# Verify installation
java -version
Step 2: Create Directory Structure
Bash
mkdir -p /app/data/kafka-data/broker
mkdir -p /app/data/kafka-data/controller
mkdir -p /app/data/kafka-logs
mkdir -p /app/data/certs
mkdir -p /app/data/control-center
***********************************************Self Sign Cert*********************************************

cd /app/data/certs

# Generate CA
openssl req -new -x509 -keyout ca-key.pem -out ca-cert.pem -days 3650 -nodes \
  -subj "/CN=KafkaCA/OU=Kafka/O=MyOrg/L=City/ST=State/C=IN"


# Generate Keystore — NOTE: keypass and storepass on separate clean lines
keytool -genkeypair -noprompt \
  -alias kafka-broker \
  -keyalg RSA \
  -keysize 2048 \
  -validity 3650 \
  -keystore /app/data/certs/pocv4kafka_server_ks.jks \
  -storepass 'pocv4kafka#2025' \
  -keypass 'pocv4kafka#2025' \
  -dname "CN=d-as-db-cmn-kfka-8-187, OU=Kafka, O=MyOrg, L=City, ST=State, C=IN"
  
# Generate CSR
keytool -certreq \
  -alias kafka-broker \
  -keystore /app/data/certs/pocv4kafka_server_ks.jks \
  -storepass 'pocv4kafka#2025' \
  -file /app/data/certs/broker.csr
  
# Create SAN extension file — IP + Hostname
cat > /app/data/certs/san.ext <<EOF
subjectAltName=IP:192.168.8.187,DNS:d-as-db-cmn-kfka-8-187
EOF

# Sign the CSR with CA
openssl x509 -req \
  -CA /app/data/certs/ca-cert.pem \
  -CAkey /app/data/certs/ca-key.pem \
  -in /app/data/certs/broker.csr \
  -out /app/data/certs/broker-signed.crt \
  -days 3650 \
  -CAcreateserial \
  -extfile /app/data/certs/san.ext
  
# Import CA cert into keystore
keytool -import -noprompt \
  -alias CARoot \
  -keystore /app/data/certs/pocv4kafka_server_ks.jks \
  -storepass 'pocv4kafka#2025' \
  -file /app/data/certs/ca-cert.pem
  
# Import signed broker cert into keystore
keytool -import -noprompt \
  -alias kafka-broker \
  -keystore /app/data/certs/pocv4kafka_server_ks.jks \
  -storepass 'pocv4kafka#2025' \
  -file /app/data/certs/broker-signed.crt

# Create Truststore and import CA
keytool -import -noprompt \
  -alias CARoot \
  -keystore /app/data/certs/pocv4kafka_server_ts.jks \
  -storepass 'pocv4kafka#2025' \
  -file /app/data/certs/ca-cert.pem
  
# Verify SAN in signed cert
openssl x509 -in /app/data/certs/broker-signed.crt -noout -text | grep -A2 "Subject Alternative"

Expected output:

X509v3 Subject Alternative Name:
    IP Address:192.168.8.187, DNS:d-as-db-cmn-kfka-8-187

****************************************************************************************************************************************

Step 3: Configure the Controller
File: /app/data/confluent-8.2.0/etc/kafka/controller.properties

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/etc/kafka# cat controller.properties

process.roles=controller
node.id=1
cluster.id=Mk3HxZ1QTyqKkLmNpQr8Ag
controller.quorum.voters=1@192.168.8.187:9091
controller.quorum.bootstrap.servers=192.168.8.187:9091

listeners=CONTROLLER://192.168.8.187:9091
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:SASL_SSL

# SASL Settings
sasl.mechanism.controller.protocol=SCRAM-SHA-256
sasl.enabled.mechanisms=SCRAM-SHA-256
# This line below fixes the "No serviceName defined" error
listener.name.controller.sasl.enabled.mechanisms=SCRAM-SHA-256

authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
super.users=User:admin
listener.name.controller.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";

# --- Add these to controller.properties ---

# Tell the controller which mechanism to use when acting as a client
sasl.mechanism.controller.protocol=SCRAM-SHA-256

# Provide the credentials for the controller's internal client
# This MUST match the credentials you have in your SCRAM database/JAAS
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="admin@123";

# SSL Settings
ssl.keystore.location=/app/data/certs/pocv4kafka_server_ks.jks
ssl.keystore.password=pocv4kafka#2025
ssl.key.password=pocv4kafka#2025
ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
ssl.truststore.password=pocv4kafka#2025
ssl.client.auth=required
ssl.endpoint.identification.algorithm=

# Log/General Settings
log.dirs=/app/data/kafka-data/controller
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.requg.replication.faest.max.bytes=104857600
num.partitions=1
offsets.topic.replication.factor=1
transaction.state.loon.state.log.minctor=1
transacti.isr=1
share.coordinator.state.topic.replication.factor=1
share.coordinator.state.topic.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
confluent.telemetry.enabled=false
*******************************************************************************************************************************************
Step 4: Configure the Broker (Single Port 9093)

File: /app/data/confluent-8.2.0/etc/kafka/broker.properties

process.roles=broker
node.id=2
cluster.id=Mk3HxZ1QTyqKkLmNpQr8Ag
controller.quorum.voters=1@192.168.8.187:9091
controller.quorum.bootstrap.servers=192.168.8.187:9091

# Added CONTROLLER here so the broker knows how to reach the quorum
listeners=CLIENT://192.168.8.187:9093
advertised.listeners=CLIENT://192.168.8.187:9093
inter.broker.listener.name=CLIENT
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:SASL_SSL,CLIENT:SASL_SSL

# SASL Settings
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
sasl.enabled.mechanisms=SCRAM-SHA-256
# Fixes the "No serviceName defined" error for the broker
#listener.name.sasl_ssl.sasl.enabled.mechanisms=SCRAM-SHA-256
listener.name.client.sasl.enabled.mechanisms=SCRAM-SHA-256

# 1. Explicitly set the mechanism for the controller listener
listener.name.controller.sasl.enabled.mechanisms=SCRAM-SHA-256

listener.name.controller.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";
sasl.mechanism.controller.protocol=SCRAM-SHA-256

authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
super.users=User:admin
allow.everyone.if.no.acl.found=false

#listener.name.sasl_ssl.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";
listener.name.client.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";

# SSL Settings
ssl.keystore.location=/app/data/certs/pocv4kafka_server_ks.jks
ssl.keystore.password=pocv4kafka#2025
ssl.key.password=pocv4kafka#2025
ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
ssl.truststore.password=pocv4kafka#2025
ssl.client.auth=required
ssl.endpoint.identification.algorithm=

# Log Settings
log.dirs=/app/data/kafka-data/broker
num.network.threads=3
num.io.threads=8
num.partitions=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
confluent.telemetry.enabled=false

*********************************************************************************************************************************
Step 5: Format Storage with SCRAM
You must wipe existing data and format with the bootstrap user.


# Clean old data
rm -rf /app/data/kafka-data/controller/*
rm -rf /app/data/kafka-data/broker/*

# Format Controller
/app/data/confluent-8.2.0/bin/kafka-storage format \
  --cluster-id Mk3HxZ1QTyqKkLmNpQr8Ag \
  --config /app/data/confluent-8.2.0/etc/kafka/controller.properties \
  --add-scram 'SCRAM-SHA-256=[name=admin,password=admin@123]'

# Format Broker
/app/data/confluent-8.2.0/bin/kafka-storage format \
  --cluster-id Mk3HxZ1QTyqKkLmNpQr8Ag \
  --config /app/data/confluent-8.2.0/etc/kafka/broker.properties \
  --add-scram 'SCRAM-SHA-256=[name=admin,password=admin@123]'

*************************************************************************************************************************************************
Step 6: Start Services
Set heap size to 400M each to stay within your 2GB RAM limit.

*****cat kafka-broker.service
[Unit]
Description=Confluent KRaft Broker
After=network.target kafka-controller.service
Requires=kafka-controller.service

[Service]
Type=simple
User=root
Environment="KAFKA_OPTS=-Djava.security.auth.login.config=/app/data/confluent-8.2.0/etc/kafka/jaas/broker_jaas.conf"
KAFKA_HEAP_OPTS="-Xmx400M -Xms400M"
ExecStart=/app/data/confluent-8.2.0/bin/kafka-server-start /app/data/confluent-8.2.0/etc/kafka/broker.properties
ExecStop=/app/data/confluent-8.2.0/bin/kafka-server-stop
Restart=on-failure
RestartSec=10
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target

*****cat kafka-controller.service
[Unit]
Description=Confluent KRaft Controller
After=network.target

[Service]
Type=simple
User=root
Environment="KAFKA_OPTS=-Djava.security.auth.login.config=/app/data/confluent-8.2.0/etc/kafka/jaas/controller_jaas.conf"
KAFKA_HEAP_OPTS="-Xmx400M -Xms400M"
ExecStart=/app/data/confluent-8.2.0/bin/kafka-server-start /app/data/confluent-8.2.0/etc/kafka/controller.properties
ExecStop=/app/data/confluent-8.2.0/bin/kafka-server-stop
Restart=on-failure
RestartSec=5
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target


# Start Controller and Kakfa service
export KAFKA_HEAP_OPTS="-Xmx400M -Xms400M"

/app/data/confluent-8.2.0/bin/kafka-server-start -daemon /app/data/confluent-8.2.0/etc/kafka/controller.properties

sleep 5

# Start Broker
/app/data/confluent-8.2.0/bin/kafka-server-start -daemon /app/data/confluent-8.2.0/etc/kafka/broker.properties

Step 7: Produce and Consume Data
Create a client config file first:
File: /tmp/client.properties

Properties
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";
ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
ssl.truststore.password=pocv4kafka#2025
ssl.keystore.location=/app/data/certs/pocv4kafka_server_ks.jks
ssl.keystore.password=pocv4kafka#2025
ssl.key.password=pocv4kafka#2025

1. Create a Topic:

/app/data/confluent-8.2.0/bin/kafka-topics --create --topic my-data --bootstrap-server 192.168.8.187:9093 --command-config /tmp/client.properties

2. Produce Data:

/app/data/confluent-8.2.0/bin/kafka-console-producer --topic my-data --bootstrap-server 192.168.8.187:9093 --producer.config /tmp/client.properties


3. Consume Data:

/app/data/confluent-8.2.0/bin/kafka-console-consumer --topic my-data --from-beginning



***************************************************************Control Center *************************************************************

cd /app/data
curl -L -O https://packages.confluent.io/confluent-control-center-next-gen/archive/confluent-control-center-next-gen-2.5.0.tar.gz
tar -xvzf confluent-control-center-next-gen-2.5.0.tar.gz

mkdir -p /app/data/confluent-control-center


root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/etc/kafka# cat   broker.properties
process.roles=broker
node.id=2
cluster.id=Mk3HxZ1QTyqKkLmNpQr8Ag
controller.quorum.voters=1@192.168.8.187:9091
controller.quorum.bootstrap.servers=192.168.8.187:9091

# Added CONTROLLER here so the broker knows how to reach the quorum
listeners=CLIENT://192.168.8.187:9093
advertised.listeners=CLIENT://192.168.8.187:9093
inter.broker.listener.name=CLIENT
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:SASL_SSL,CLIENT:SASL_SSL

# SASL Settings
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256
sasl.enabled.mechanisms=SCRAM-SHA-256
# Fixes the "No serviceName defined" error for the broker
#listener.name.sasl_ssl.sasl.enabled.mechanisms=SCRAM-SHA-256
listener.name.client.sasl.enabled.mechanisms=SCRAM-SHA-256

# 1. Explicitly set the mechanism for the controller listener
listener.name.controller.sasl.enabled.mechanisms=SCRAM-SHA-256

listener.name.controller.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";
sasl.mechanism.controller.protocol=SCRAM-SHA-256

authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
super.users=User:admin
allow.everyone.if.no.acl.found=false

#listener.name.sasl_ssl.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";
listener.name.client.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";

# SSL Settings
ssl.keystore.location=/app/data/certs/pocv4kafka_server_ks.jks
ssl.keystore.password=pocv4kafka#2025
ssl.key.password=pocv4kafka#2025
ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
ssl.truststore.password=pocv4kafka#2025
ssl.client.auth=required
ssl.endpoint.identification.algorithm=

# Log Settings
log.dirs=/app/data/kafka-data/broker
num.network.threads=3
num.io.threads=8
num.partitions=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
confluent.telemetry.enabled=false

# --- CONFLUENT CONTROL CENTER & METRICS SETTINGS ---
# Enable the metrics reporter so C3 can see broker health
metric.reporters=io.confluent.metrics.reporter.ConfluentMetricsReporter
confluent.metrics.reporter.bootstrap.servers=192.168.8.187:9093


# Security for the reporter to talk to the broker
confluent.metrics.reporter.security.protocol=SASL_SSL
confluent.metrics.reporter.sasl.mechanism=SCRAM-SHA-256
confluent.metrics.reporter.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";
# SSL Settings for the reporter (Matches your broker certs)
confluent.metrics.reporter.ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
confluent.metrics.reporter.ssl.truststore.password=pocv4kafka#2025
confluent.metrics.reporter.ssl.keystore.location=/app/data/certs/pocv4kafka_server_ks.jks
confluent.metrics.reporter.ssl.keystore.password=pocv4kafka#2025
confluent.metrics.reporter.ssl.key.password=pocv4kafka#2025

# Single-node overrides (Prevents C3 from waiting for 3 brokers)
confluent.controlcenter.internal.topics.replication=1
confluent.metrics.topic.replication=1
confluent.license.topic.replication.factor=1
confluent.metadata.topic.replication.factor=1
root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/etc/kafka#
###################################################################################################################################################

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/etc/kafka# cat controller.properties
process.roles=controller
node.id=1
cluster.id=Mk3HxZ1QTyqKkLmNpQr8Ag
controller.quorum.voters=1@192.168.8.187:9091
controller.quorum.bootstrap.servers=192.168.8.187:9091

listeners=CONTROLLER://192.168.8.187:9091
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:SASL_SSL

# SASL Settings
sasl.mechanism.controller.protocol=SCRAM-SHA-256
sasl.enabled.mechanisms=SCRAM-SHA-256
# This line below fixes the "No serviceName defined" error
listener.name.controller.sasl.enabled.mechanisms=SCRAM-SHA-256

authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer
super.users=User:admin
listener.name.controller.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";

# --- Add these to controller.properties ---

# Tell the controller which mechanism to use when acting as a client
sasl.mechanism.controller.protocol=SCRAM-SHA-256

# Provide the credentials for the controller's internal client
# This MUST match the credentials you have in your SCRAM database/JAAS
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="admin@123";

# SSL Settings
ssl.keystore.location=/app/data/certs/pocv4kafka_server_ks.jks
ssl.keystore.password=pocv4kafka#2025
ssl.key.password=pocv4kafka#2025
ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
ssl.truststore.password=pocv4kafka#2025
ssl.client.auth=required
ssl.endpoint.identification.algorithm=

# Log/General Settings
log.dirs=/app/data/kafka-data/controller
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.requg.replication.faest.max.bytes=104857600
num.partitions=1
offsets.topic.replication.factor=1
transaction.state.loon.state.log.minctor=1
transacti.isr=1
share.coordinator.state.topic.replication.factor=1
share.coordinator.state.topic.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
confluent.telemetry.enabled=false
root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/etc/kafka#
##########################################################################################################################################

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/etc/kafka# cat admin.properties
bootstrap.servers=192.168.8.187:9093
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";
ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
ssl.truststore.password=pocv4kafka#2025
ssl.endpoint.identification.algorithm=
root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/etc/kafka#


##########################################################################################################################################

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/etc/kafka/jaas# ls
broker_jaas.conf  controller_jaas.conf
root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/etc/kafka/jaas# cat broker_jaas.conf
KafkaServer {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="admin"
  password="admin@123";
};
KafkaClient {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="admin"
  password="admin@123";
};

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-8.2.0/etc/kafka/jaas# cat controller_jaas.conf
KafkaServer {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="admin"
  password="admin@123";
};

KafkaClient {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="admin"
  password="admin@123";
};

########################################################################################################################################

root@d-as-db-cmn-kfka-8-187:/app/data/confluent-control-center-next-gen-2.5.0/etc/confluent-control-center# cat control-center-d.properties
bootstrap.servers=192.168.8.187:9093
confluent.controlcenter.id=10
confluent.controlcenter.data.dir=/app/data/confluent-control-center
confluent.controlcenter.internal.topics.replication=1
#confluent.controlcenter.internal.topics.partitions=1
confluent.controlcenter.streams.num.stream.threads=1
confluent.controlcenter.command.topic.replication=1
confluent.controlcenter.command.topic.partitions=1
confluent.monitoring.interceptor.topic.replication=1
confluent.metrics.topic.replication=1
confluent.controlcenter.replication.factor=1

confluent.controlcenter.streams.enable=false
confluent.controlcenter.usage.data.collection.enable=false

#confluent.controlcenter.prometheus.url=http://localhost:9090
#confluent.controlcenter.prometheus.enable=true
#confluent.controlcenter.prometheus.alertmanager.yaml.refresh.enabled=false

confluent.controlcenter.kafka.security.protocol=SASL_SSL
confluent.controlcenter.kafka.sasl.mechanism=SCRAM-SHA-256

confluent.controlcenter.kafka.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";

confluent.controlcenter.kafka.ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
confluent.controlcenter.kafka.ssl.truststore.password=pocv4kafka#2025
confluent.controlcenter.kafka.ssl.keystore.location=/app/data/certs/pocv4kafka_server_ks.jks
confluent.controlcenter.kafka.ssl.keystore.password=pocv4kafka#2025
confluent.controlcenter.kafka.ssl.key.password=pocv4kafka#2025

confluent.controlcenter.kafka.ssl.endpoint.identification.algorithm=

confluent.controlcenter.kafka.admin.security.protocol=SASL_SSL
confluent.controlcenter.kafka.admin.sasl.mechanism=SCRAM-SHA-256
confluent.controlcenter.kafka.admin.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";
confluent.controlcenter.kafka.admin.ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
confluent.controlcenter.kafka.admin.ssl.truststore.password=pocv4kafka#2025
confluent.controlcenter.kafka.admin.ssl.keystore.location=/app/data/certs/pocv4kafka_server_ks.jks
confluent.controlcenter.kafka.admin.ssl.keystore.password=pocv4kafka#2025
confluent.controlcenter.kafka.admin.ssl.key.password=pocv4kafka#2025
confluent.controlcenter.kafka.admin.ssl.endpoint.identification.algorithm=

confluent.controlcenter.streams.security.protocol=SASL_SSL
confluent.controlcenter.streams.sasl.mechanism=SCRAM-SHA-256
confluent.controlcenter.streams.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin@123";
confluent.controlcenter.streams.ssl.truststore.location=/app/data/certs/pocv4kafka_server_ts.jks
confluent.controlcenter.streams.ssl.truststore.password=pocv4kafka#2025
confluent.controlcenter.streams.ssl.keystore.location=/app/data/certs/pocv4kafka_server_ks.jks
confluent.controlcenter.streams.ssl.keystore.password=pocv4kafka#2025
confluent.controlcenter.streams.ssl.key.password=pocv4kafka#2025
confluent.controlcenter.streams.ssl.endpoint.identification.algorithm=

#confluent.controlcenter.rest.authentication.method=BASIC
#confluent.controlcenter.rest.authentication.realm=ControlCenter
#confluent.controlcenter.rest.authentication.roles=Administrators,Restricted

###########################################################################################################################################
root@d-as-db-cmn-kfka-8-187:/etc/systemd/system# cat kafka-broker.service
[Unit]
Description=Confluent KRaft Broker
After=network.target kafka-controller.service
Requires=kafka-controller.service

[Service]
Type=simple
User=root
Environment="KAFKA_OPTS=-Djava.security.auth.login.config=/app/data/confluent-8.2.0/etc/kafka/jaas/broker_jaas.conf"
KAFKA_HEAP_OPTS="-Xmx400M -Xms400M"
ExecStart=/app/data/confluent-8.2.0/bin/kafka-server-start /app/data/confluent-8.2.0/etc/kafka/broker.properties
ExecStop=/app/data/confluent-8.2.0/bin/kafka-server-stop
Restart=on-failure
RestartSec=10
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
root@d-as-db-cmn-kfka-8-187:/etc/systemd/system# cat kafka-controller.service
[Unit]
Description=Confluent KRaft Controller
After=network.target

[Service]
Type=simple
User=root
Environment="KAFKA_OPTS=-Djava.security.auth.login.config=/app/data/confluent-8.2.0/etc/kafka/jaas/controller_jaas.conf"
KAFKA_HEAP_OPTS="-Xmx400M -Xms400M"
ExecStart=/app/data/confluent-8.2.0/bin/kafka-server-start /app/data/confluent-8.2.0/etc/kafka/controller.properties
ExecStop=/app/data/confluent-8.2.0/bin/kafka-server-stop
Restart=on-failure
RestartSec=5
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
root@d-as-db-cmn-kfka-8-187:/etc/systemd/system# cat confluent-control-center.service
[Unit]
Description=Confluent Control Center
After=network.target kafka-broker.service

[Service]
Type=simple
User=root
Group=root
Environment="CONTROL_CENTER_HEAP_OPTS=-Xmx500M -Xms500M"
Environment=CONTROL_CENTER_OPTS=-Djava.security.auth.login.config=/app/data/confluent-control-center-next-gen-2.5.0/etc/confluent-control-center/c3_jaas.conf
# UPDATE THESE TWO PATHS:
ExecStart=/app/data/confluent-control-center-next-gen-2.5.0/bin/control-center-start /app/data/confluent-control-center-next-gen-2.5.0/etc/confluent-control-center/control-center-d.properties

ExecStop=/app/data/confluent-control-center-next-gen-2.5.0/bin/control-center-stop
Restart=on-failure

[Install]
WantedBy=multi-user.target
root@d-as-db-cmn-kfka-8-187:/etc/systemd/system#

#####################################################################################################################################################

