# Downloading Kafka
```bash
$ wget "https://downloads.apache.org/kafka/3.9.0/kafka_2.13-3.9.0.tgz"
$ tar xfz kafka_2.13-3.9.0.tgz
$ rm kafka_2.13-3.9.0.tgz
$ mv kafka_2.13-3.9.0/ ~/kafka
$ export PATH=~/kafka/bin:"$PATH"
```

# Configuring Kafka
## Create the configuration files for the Kafka brokers in the directory `~/kafka/config/kafka1.properties`:
- broker.id=1    #1 
- log.dirs=<PATH-TO-YOUR-USER-DIRECTORY>/kafka/data/kafka1    #2
- listeners=PLAINTEXT://:9092,CONTROLLER://:9192    #3
- process.roles=broker,controller    #4
- controller.quorum.voters=1@localhost:9192,2@localhost:9193,\
- 3@localhost:9194    #5
- controller.listener.names=CONTROLLER    #6
- listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT    #7
1.   The ID of the broker (here 1)
2.  The directory where the broker should store the files. Please adapt the directory.
3.  We’re using port 9092 for normal broker communication and port 9192 for the controller. You can call the listeners whatever you want.
4.  Runs the broker in dual mode
5.  We need to provide the connection string for all of our controllers.
6.  Which listener should be used for the controller?
7.  We don’t use TLS for our test environment. 

## Configure Broker 2 in ~/kafka/config/kafka2.properties:

- broker.id=2
- log.dirs=<PATH-TO-YOUR-USER-DIRECTORY>/kafka/data/kafka2
- listeners=PLAINTEXT://:9093,CONTROLLER://:9193
- process.roles=broker,controller
- controller.quorum.voters=1@localhost:9192,2@localhost:9193,3@localhost:9194
- controller.listener.names=CONTROLLER
- listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

## finally, we’ll configure Broker 3 in ~/kafka/config/kafka3.properties:
- broker.id=3
- log.dirs=<PATH-TO-YOUR-USER-DIRECTORY>/kafka/data/kafka3
- listeners=PLAINTEXT://:9094,CONTROLLER://:9194
- process.roles=broker,controller
- controller.quorum.voters=1@localhost:9192,2@localhost:9193,3@localhost:9194
- controller.listener.names=CONTROLLER
- listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

# Preparing the data directories

```bash
$ mkdir -p ~/kafka/data/kafka1 ~/kafka/data/kafka2 ~/kafka/data/kafka3    #1

$ export KAFKA_CLUSTER_ID="$(kafka-storage.sh random-uuid)"    #2

$ kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c ~/kafka/config/kafka1.properties    #3
Formatting metadata directory <PATH-TO-YOUR-USER-DIRECTORY>
/kafka/data/kafka1 with metadata.version 3.9-IV0.

$ kafka-storage.sh format -t $KAFKA_CLUST7ER_ID -c ~/kafka/config/kafka2.properties    #3
Formatting metadata directory <PATH-TO-YOUR-USER-DIRECTORY>
/kafka/data/kafka2 with metadata.version 3.9-IV0.

$ kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c ~/kafka/config/kafka3.properties    #3
Formatting metadata directory <PATH-TO-YOUR-USER-DIRECTORY>
/kafka/data/kafka3 with metadata.version 3.9-IV0.
```
1. Creates the data directories
2. Kafka cluster ID
3. Formats the directories

# Starting Kafka

-  Start the first broker
```bash
~/kafka/bin/kafka-server-start.sh ~/kafka/config/kafka1.properties
``` 
-  Start the second broker in a new terminal
```bash
~/kafka/bin/kafka-server-start.sh ~/kafka/config/kafka2.properties
``` 
-  Start the third broker in a new terminal
```bash
~/kafka/bin/kafka-server-start.sh ~/kafka/config/kafka3.properties
```

Of course, we can leave it at that, but we can also stop the brokers one after another and replace them with brokers in daemon mode. This way, we won’t be able to see the logs, but we also won’t need to have three terminal windows open all the time:
```bash
# Stop Broker 1 with Ctrl-C
~/kafka/bin/kafka-server-start.sh \
-daemon ~/kafka/config/kafka1.properties    #1

# Stop Broker 2 with Ctrl-C
~/kafka/bin/kafka-server-start.sh \
-daemon ~/kafka/config/kafka2.properties    #1

# Stop Broker 3 with Ctrl-C
~/kafka/bin/kafka-server-start.sh
-daemon ~/kafka/config/kafka3.properties    #1
```
#1. Starts the broker in the background

To check whether everything is still running, we can use the following command:
```bash
$ kafka-broker-api-versions.sh \
--bootstrap-server localhost:9092,localhost:9093,localhost:9094
```

# Stopping Kafka
To stop all Kafka brokers, we can simply execute the following command:
```bash
~/kafka/bin/kafka-server-stop.sh
```
But this command stops all Kafka brokers. To stop a single broker, we can create a new script 
`~/kafka/bin/kafka-broker-stop.sh`:

```bash
#!/bin/bash
BROKER_ID="$1"

if [ -z "$BROKER_ID" ]; then
echo "usage ./kafka-broker-stop.sh [BROKER-ID]"
exit 1
fi

PIDS=$(ps ax | grep -i 'kafka\.Kafka' | grep java \
| grep "kafka${BROKER_ID}.properties" | grep -v grep | awk '{print $1}')

if [ -z "$PIDS" ]; then
echo "No kafka server to stop"
exit 1
else
kill -s TERM $PIDS
fi
```
Instead of stopping all brokers, it will stop only the one for which the broker ID matches. To stop broker 1, for example, we can use the following command:

```bash
$ chmod +x ~/kafka/bin/kafka-broker-stop.sh
$ ~/kafka/bin/kafka-broker-stop.sh 1
```
We can check with the kafka-broker-api-versions.sh script to find out whether this worked:

```bash
$ kafka-broker-api-versions.sh \
--bootstrap-server localhost:9092,localhost:9093,localhost:9094
Connection to node -1 (localhost/127.0.0.1:9092) could not be established.
Broker may not be available.    #1
localhost:9093 (id: 2 rack: null) -> (
# A lot of text
)
localhost:9094 (id: 3 rack: null) -> (
# A lot of text
)
#1 Broker 1 is missing, and there’s an error message instead.
```