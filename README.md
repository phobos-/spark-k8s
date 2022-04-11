# Spark Streaming on k8s

## Prerequisites

- minikube `brew install minikube kafkacat`
- docker (I had to sue docker desktop with my m1 setup) `brew install docker` (`https://docs.docker.com/desktop/mac/apple-silicon/`)

## Starting minikube

```bash
minikube stop
minikube delete
minikube start --insecure-registry="localhost:5000" --memory 7851 --cpus 4
minikube addons enable registry
```

### Geting minikube ip and master address

```bash
minikube ip
minikube ssh -- nslookup host.minikube.internal
kubectl cluster-info
```
### Running minikube dashboard

```bash
minikube dashboard --url &
kubectl proxy --address='0.0.0.0' --disable-filter=true --port=5885 &
```

## App Development

### Build Kafka

```bash
docker-compose up --build -d zookeeper kafka
docker-compose logs -f
```

### Build Postgres

```bash
docker-compose up --build -d postgres
```

### Bulding Spark docker image

#### Download jar files

```bash
sh jarfile_download.sh
```
#### Build the image

```bash
docker-compose up --build -d spark
```
### Push Location Dataset to Kafka

```bash
docker-compose exec spark python produce_data.py
```
### Check the Kafka Topic

```bash
kcat -b localhost:9092 -o beginning -t locations -C -c2 | jq
```

### Run spark streaming locally

```bash
docker-compose exec spark \
spark-submit --verbose --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.1.1 \
--master local --jars /opt/spark/jars/postgresql-42.2.5.jar \
stream_process.py postgres kafka:29092
```

### Check the result

```
docker-compose exec postgres psql -h localhost -U admin appdb;

select * from locations limit 10;
select count(*) from locations limit 10;
```

## App K8s Deployment

```bash
cd k8s/
```

### Spark Image in Registry

#### start registry proxy

```bash
docker run -d --rm -it --network=host alpine ash -c "apk add socat && socat TCP-LISTEN:5000,reuseaddr,fork TCP:192.168.49.2:5000"
```
#### tag and push image to registry

```bash
docker tag spark-streaming-k8s_spark localhost:5000/stream-spark:v4
docker push localhost:5000/stream-spark:v4
```
### Run Spark Streaming

#### Download Spark

```bash
wget https://archive.apache.org/dist/spark/spark-3.1.1/spark-3.1.1-bin-hadoop3.2.tgz && \
tar -xzvf spark-3.1.1-bin-hadoop3.2.tgz && \
rm -rf spark-3.1.1-bin-hadoop3.2.tgz
```
#### Create Spark Service Account

```bash
kubectl create serviceaccount spark
kubectl create clusterrolebinding spark-role --clusterrole=edit  --serviceaccount=default:spark --namespace=default
kubectl cluster-info
kubectl proxy --address='0.0.0.0' --disable-filter=true --port=8443 &

```

#### Spark Submit Streaming

```bash
kubectl apply -f k8s-config.yaml
```

```
./spark-3.1.1-bin-hadoop3.2/bin/spark-submit --verbose --master k8s://http://localhost:8443 \
--deploy-mode cluster \
--conf spark.kubernetes.driver.pod.name=location-streaming-app \
--conf spark.kubernetes.container.image.pullPolicy=Always \
--conf spark.executor.instances=1 \
--conf spark.executor.memory=2G \
--conf spark.executor.cores=1 \
--conf spark.driver.memory=1G \
--conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
--conf spark.kubernetes.container.image=localhost:5000/stream-spark:v4 \
--name locations-streaming \
--jars local:///opt/spark/jars/commons-pool2-2.9.0.jar,local:///opt/spark/jars/postgresql-42.2.5.jar,local:///opt/spark/jars/spark-streaming_2.12-3.1.1.jar,local:///opt/spark/jars/spark-token-provider-kafka-0-10_2.12-3.1.1.jar,local:///opt/spark/jars/kafka-clients-2.6.0.jar,local:///opt/spark/jars/spark-sql-kafka-0-10_2.12-3.1.1.jar,local:///opt/spark/jars/spark-streaming-kafka-0-10-assembly_2.12-3.1.1.jar local:///opt/work-dir/stream_process.py postgres kafka:9093
```

## Results

### Logs

```bash
kubectl logs -f pod/location-streaming-app
```

### Spark UI
```bash
kubectl port-forward --address 0.0.0.0 pod/location-streaming-app 4040:4040 &
http://localhost:4040/
```

### Postgres

```bash
docker-compose exec postgres psql -h localhost -U admin appdb;

select * from locations limit 10;
select count(*) from locations limit 10;
```
