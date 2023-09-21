# Kafka-Stack-on-Kubernetes (KSOK)

## Overview

KSOK deploys a Kafka stack on Kubernetes utilizing bitnami helm charts. 

## Enabling Bitnami Helm Charts

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

## Kafka

KSOK utilizes the Bitnami [kafka helm chart](https://github.com/bitnami/charts/tree/main/bitnami/kafka) to deploy Kafka on Kubernetes.

### values.yaml

The [kafka-values.yaml](kafka-values.yaml) encapsulates the contents of the Bitnami kafka [values.yaml](https://github.com/bitnami/charts/blob/main/bitnami/kafka/values.yaml) file with custom values for a small number of parameters.

#### replicaCount

The default KSOK Kafka cluster size is 3, so the replicaCount parameter is as follows:

```
replicaCount: 3
```

#### kraft

The KSOK Kafka deployment utilizes the kraft protocol instead of Zookeeper. Accordingly, the kafka-values.yaml file has the following configuration:

```
kraft:
  enabled: true
```

#### persistence

The default KSOK Kafka deployment enavbles persistence and utilizes the Kubernetes manual storage class, resulting in the following persistence parameter.

```
persistence:
  enabled: true
  storageClass: "manual"
  selector:
    matchLabels:
      app: kafka
```

#### resources

The default KSOK resources parameter is as follows:

```
resources:
  requests:
    cpu: 1000m
    memory: 2048Mi
  limits:
    cpu: 2000m
    memory: 4096Mi
```

#### readinessProbe and livenessProbe

```
livenessProbe:
  enabled: true
  initialDelaySeconds: 120
  timeoutSeconds: 15
  failureThreshold: 3
  periodSeconds: 10
  successThreshold: 1
  
readinessProbe:
  enabled: true
  initialDelaySeconds: 120
  failureThreshold: 3
  timeoutSeconds: 15
  periodSeconds: 10
  successThreshold: 1
```

#### Persistent Volumes

The initial KSOK Kafka deployment utilizes the manual storage class, with files written to the Kafka pod host machines. Accordingly, there are three persistentvolume files that need to be deployed before the Kafka helm install.

Before using the bundled pv files, the spec.nodeAffinity.required.nodeSelectorTerms.matchExpressions.values entry in each file needs to be changed to a hostname or ip address corresponding to the desired Kafka host. Once the PV files are updated, execute the following command:

```
sh create-pv.sh
```

### Helm Install Command

There is currently an issue with the latest version of the bitnami/kafka helm chart:

```
helm install -f kafka-schema-registry-values.yaml -n kafka kafka bitnami/kafka
Error: parse error at (schema-registry/charts/common/templates/_labels.tpl:14): unclosed action
```

Accordingly, need to rollback to an earlier version, resulting in the following helm install command:

```
export NAMESPACE=kafka
export DEPLOYMENT_NAME=kafka
export VERSION=23.0.6

helm install -n $NAMESPACE $DEPLOYMENT_NAME -f kafka-values.yaml bitnami/kafka --version=$VERSION
```

### Helm Delete Command

```
export NAMESPACE=kafka
export DEPLOYMENT_NAME=kafka

helm delete -n $NAMESPACE $DEPLOYMENT_NAME
```

## Kafka Connect

### Overview

Kafka Connect provides a platform for streaming messages to Kafka, where a variety of other applications consume, process, and execute analytics on message streams.

There are two elements to the KSOK kafka connect deployment: kafka schema registry and kafka connect.

### Kafka Schema Registry Deployment

#### values.yaml

The values.yaml file is in the form of the [kafka-schema-registry-values.yaml](kafka-schema-registry-values.yaml) file. The kafka-schema-registry-values.yaml disables the embedded Kafka deployment and assumes Kafka is deployed to the kafka namespace:

```
kafka:
  enabled: false
externalKafka:
  brokers: ["PLAINTEXT://kafka-0.kafka-headless.kafka.svc.cluster.local:9092"]
```

#### Helm Install Command

As is the case with the bitnami Kafka chart, the latest version of the Kafka schema registry helm chart has the following issue:

```
helm install -f kafka-schema-registry-values.yaml -n kafka kafka-schema-registry bitnami/schema-registry
Error: parse error at (schema-registry/charts/common/templates/_labels.tpl:14): unclosed action 
```

Accordingly, the helm install is as follows:

```
export VERSION=13.0.1

helm install -f kafka-schema-registry-values.yaml -n kafka kafka-schema-registry bitnami/schema-registry --version=$VERSION
```

### Kafka Connect Deployment

#### Kafka Connect Helm Repository

```
helm repo add confluentinc https://confluentinc.github.io/cp-helm-charts/
helm repo update
```

#### values.yaml

The values.yaml file is in the form of the [kafka-connect-values.yaml](kafka-connect-values.yaml) file. An example as follows assumes Kafka and the schema registry are deployed to the kafka namespace:

```
kafka:
  bootstrapServers: "PLAINTEXT://kafka-headless.kafka:9092"

cp-schema-registry:
  url: "kafka-schema-registry.kafka"
```

#### Helm Install Command

```
helm install -f kafka-connect-values.yaml kafka-connect -n kafka cp-helm-charts/cp-kafka-connect
```

## Kafka UI

### Overview

KSOK uses the provectus [kafka-ui](https://github.com/provectus/kafka-ui-charts) Helm chart to install the provectus [kafka-ui](https://github.com/provectus/kafka-ui).

### Adding the kafka-ui Helm Repository

The repo is added as follows:

```
helm repo add kafka-ui https://provectus.github.io/kafka-ui-charts
helm repo update
```

### values.yaml

The [kafka-ui-values.yaml](kafka-ui-values.yaml) corresponds to the values.yaml file used in the kafka-ui helm install. The overridden elements of the values.yaml file are as follows:

#### yamlApplicationConfig

The bootstrapServers value specifies deployment of Kafka to the kafka namespace:

```
yamlApplicationConfig:
  kafka:
    clusters:
      - name: kafka
        bootstrapServers: PLAINTEXT://kafka-headless.kafka:9092
```

#### resources

```
resources:
  limits:
    cpu: 1000m
    memory: 1024Mi
  requests:
    cpu: 500m
    memory: 512Mi
```

### Helm Install Command

```
helm install -n kafka kafka-ui -f kafka-ui-values.yaml kafka-ui/kafka-ui
```
