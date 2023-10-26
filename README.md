## Contexte
Ce projet a été réalisé dans le cadre d'un TP de Big Data. Il a pour objectif de comprendre comment utiliser Spark pour du traitement de données en temps réel.

## Outils et Versions

### Machine locale

- Docker v24.0.6
- JDK v1.8
- Maven v3.8.1 

### Containeurs Docker
- Hadoop v2.7.2
- Spark v2.2.0 (Scala v2.11.8)
- JDK v1.8

## Installation avec Docker

### Docker Image

L'image **totofunku/bigdata-cours:upgrade** ayant un système hdfs corrompu, j'ai pris une image propre n'ayant un système de fichier corrompu.

```sh
docker pull liliasfaxi/spark-hadoop:hv-2.7.2
```

### Docker Network

La commande ci-dessous permet de créer un réseau afin d'inter-connecter tous les containeurs Hadoop.

```bash
docker network create --driver=bridge hadoop
```

### Docker Run

Les commandes ci-dessous permettent de lancer les containeurs.

```sh
# Master node
docker run -itd --net=hadoop -p 9870:9870 -p 8042:8042 -p 8088:8088 -p 7077:7077 -p 16010:16010 --name hadoop-master --hostname hadoop-master liliasfaxi/spark-hadoop:hv-2.7.2
# Slave 1
docker run -itd -p 8040:8042 --net=hadoop --name hadoop-slave1 --hostname hadoop-slave1 liliasfaxi/spark-hadoop:hv-2.7.2
# Slave 2
docker run -itd -p 8041:8042 --net=hadoop --name hadoop-slave2 --hostname hadoop-slave2 liliasfaxi/spark-hadoop:hv-2.7.2
```

## Installation avec docker-compose

**Si vous avez déjà fait l'installation avec Docker, passer à l'étape suivante.**

```bash
# Exécution en background
docker-compose up -d
```

## Execution de Hadoop

Une fois que les 3 containeurs sont lancés, il faut lancer hadoop :

```sh
docker exec -it hadoop-master ./start-hadoop.sh
```

## Préparation de l'environnement de travail

Vu que l'image docker utilisée est une image propre, il faut reconstituer le système de fichier hdfs avec les bons fichiers et répertoires. Les fichiers *puchases.txt* et *purchases2.txt* existent déjà en local.

```sh
docker exec -it hadoop-master bash

# Création du fichie en local
cat <<EOF > file1.txt
Hello Spark Wordcount!
Hello Hadoop Also :)
EOF

# Création des répertoires 
hdfs dfs -mkdir -p .
hdfs dfs -mkdir -p input

# Upload des fichies locaux (file1.txt / purchases.txt / purchases2.txt)
hdfs dfs -put purchases*.txt input/
hdfs dfs -put file1.txt

# Vérification
hdfs dfs -ls -R

# Résultat attendu
# -rw-r--r--   2 root supergroup         44 2023-10-26 17:43 file1.txt
# drwxr-xr-x   - root supergroup          0 2023-10-26 17:45 input
# -rw-r--r--   2 root supergroup  211312924 2023-10-26 17:35 input/purchases.txt
# -rw-r--r--   2 root supergroup  243309628 2023-10-26 17:35 input/purchases2.txt
```

## Partie 1 : WordCount

La partie une consiste à écrire un prgramme permettant de calculer le nombre de fois que chaque mot est utilisé dans un fichier texte fourni en argument.

**Attention : Le programme renverra une erreur si le répertoire de sortie existe déjà.**

Avec spark-shell :
```sh
val lines = sc.textFile("file1.txt")
val words = lines.flatMap(_.split("\\s+"))
val wc = words.map(w => (w, 1)).reduceByKey(_ + _)
wc.saveAsTextFile("file1.count")
```

Avec spark-submit (publier en mode client):
```
spark-submit --class tn.insat.tp21.WordCountTask --master local --driver-memory 4g --executor-memory 2g --executor-cores 1 wordcount-1.jar input/purchases.txt output
```

Avec spark-submit (publier en mode cluster):
```
spark-submit --class tn.insat.tp21.WordCountTask --master yarn --deploy-mode cluster --driver-memory 4g --executor-memory 2g --executor-cores 1 wordcount-1.jar input/purchases.txt output2
```

## Partie 2 : Stream

Ce code permet de calculer le nombre de mots dans un stream de données toutes les secondes. Lors du test, le stream de donnée en input été un simple ouverture de port avec la commande suivante :

```sh
# Ouverture et mise en écoute du port 9999
nc -lvp 9999
```

Côté client :
![Alt text](src\stream-test-client-side.png "Stream result client side")

Côté serveur :
![Alt text](src\stream-test-server-side.png "Stream result server side")

Avec spark-submit :
```
spark-submit --class tn.insat.tp22.Stream --master local --driver-memory 4g --executor-memory 2g --executor-cores 1 stream-1.jar
```