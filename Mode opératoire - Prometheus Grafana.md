## Mode opératoire - Prometheus Grafana

Cassandra 4.1.11 avec Docker Compose (Cluster 2 nœuds)

Cluster Cassandra déployé via Docker Compose avec 4 nœuds sur 2 racks différents dans 2 datacenters.


#### 1°) Démarrage du cluster

#### Étape 1 : Préparation de l'environnement

```bash
cd ~
sudo rm -Rf ~/cassandra-monitoring
```

#### Ici, on va simplement cloner le projet :
```bash
git clone https://github.com/crystalloide/cassandra-monitoring

cd ~/cassandra-monitoring
```
#### Vérifier le contenu ou créer le fichier docker compose de notre cluster 4 noeuds cassandra :
```bash
cat Cluster_2_noeuds_1_rack_1_DC_Prometheus_Grafana.yml
```

#### 2°) Créer les répertoires de volumes :
```bash
sudo rm -Rf ~/cassandra-monitoring/docker/cassandra*
mkdir -p ~/cassandra-monitoring/docker/cassandra01 ~/cassandra-monitoring/docker/cassandra02 
```
```bash
mkdir -p ~/cassandra-monitoring/docker/cassandra01-conf ~/cassandra-monitoring/docker/cassandra02-conf
```
#### On affiche les répertoires créés :
```bash
ls ~/cassandra-monitoring/docker
```
##### Affichage : 
```bash
     cassandra01       cassandra02       
     cassandra01-conf  cassandra02-conf 
```

#### 3°) Démarrage du cluster avec Docker Compose

```bash
# Démarrer le cluster en arrière-plan
docker compose -f Cluster_2_noeuds_1_rack_1_DC_Prometheus_Grafana.yml up  -d
```
#### Suivre les logs pour vérifier le démarrage (dans un autre terminal si besoin)
```bash
cd ~/cassandra-monitoring
docker compose -f Cluster_2_noeuds_1_rack_1_DC_Prometheus_Grafana.yml logs
```

#### Dans un autre terminal, pour suivre  :
```bash
cd ~
docker ps -a 
```
#### Affichage (exemple) ': 
```bash
# 
# CONTAINER ID   IMAGE              COMMAND                  CREATED              STATUS                             PORTS                                                                                                                                                       NAMES
# cefc35985646   cassandra:latest   "docker-entrypoint.s…"   About a minute ago   Up 14 seconds (health: starting)   7001/tcp, 9160/tcp, 0.0.0.0:7200->7000/tcp, [::]:7200->7000/tcp, 0.0.0.0:7299->7199/tcp, [::]:7299->7199/tcp, 0.0.0.0:9242->9042/tcp, [::]:9242->9042/tcp   cassandra02
# 1e6d94687116   cassandra:latest   "docker-entrypoint.s…"   About a minute ago   Up About a minute (healthy)        7001/tcp, 9160/tcp, 0.0.0.0:7199->7199/tcp, [::]:7199->7199/tcp, 0.0.0.0:7100->7000/tcp, [::]:7100->7000/tcp, 0.0.0.0:9142->9042/tcp, [::]:9142->9042/tcp   cassandra01
```

**Note** : Le démarrage complet peut prendre 5 minutes car les nœuds démarrent séquentiellement avec healthchecks. L'ordre de démarrage est :

1. cassandra01 (1er seed, peut démarrer en 1er)
2. cassandra03 (2ème seed, attend cassandra01 healthy)


#### Pour visualiser les logs de cassandra01 : 
```bash
docker logs cassandra01
```

#### 4°) Vérification du cluster (après 5-10 minutes)

#### Regarder les ports à l'écoute :
```bash
netstat -anl | grep 0:
```

#### Vérifier que les 4 conteneurs sont UP sinon attendre (non listé ou encore en train de joindre : 'UJ')
```bash
cd ~/cassandra-monitoring
docker compose -f Cluster_2_noeuds_1_rack_1_DC_Prometheus_Grafana.yml ps
```

#### Vérifier le statut du cluster via nodetool
```bash
docker exec -it cassandra01 nodetool status
```
#### Vous devriez voir finalement les 2 nœuds cassandra avec le statut "UN" (Up Normal)
#### Le résultat devrait ressembler à :

```bash
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address          Load        Tokens  Owns (effective)  Host ID                               Rack
UN  10.17.64.6    80.06 KiB       16      52.0%             6747a2fb-3f5a-4342-91b5-1f1d177366af  rack1
UN  10.17.64.5    119.82 KiB      16      48.5%             e2efa530-2ac0-4957-8827-60860279295b  rack1

```


#### 5°) Accès au monitoring : 

##### Avec un navigateur, aller sur l'UI Grafana  :  login 'admin' et mot de passe 'admin' :-)
```bash
http://localhost:3000
```

##### Avec un navigateur, aller sur l'UI Prometheus  : 
```bash
http://localhost:9090/query
```

```bash
##### Métriques exposées par l'exporter
curl http://localhost:8480/metrics | head -30

##### Etat des targets dans Prometheus :
```bash
curl http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -E "health|job|instance"
```


____
##### 6°) Nous allons utiliser l'outil "cassandra-stress" pour générer plusieurs centaines d'écritures sur le noeud. 
#####     Exécutez la commande suivante sur le premier terminal :
____
```bash
docker exec -it cassandra01 bash 
```

```bash
cd /opt/cassandra/tools/bin
```

##### bash cassandra-stress write n=1000 -node 10.17.64.5
```bash
bash cassandra-stress write no-warmup n=25000 cl=one -rate threads=1 -node 10.17.64.5
```

##### Assurez-vous que le second terminal vous reste visible pendant l'exécution de cassandra-stress.
##### L'outil cassandra-stress va écrire 25 000 lignes sur le noeud.

###### Affichage en retour du benchmark : 
```text

******************** Stress Settings ********************
Command:
  Type: write
  Count: 25,000
  No Warmup: true
  Consistency Level: ONE
  Target Uncertainty: not applicable
  Key Size (bytes): 10
  Counter Increment Distibution: add=fixed(1)
Rate:
  Auto: false
  Thread Count: 1
  OpsPer Sec: 0
Population:
  Sequence: 1..25000
  Order: ARBITRARY
  Wrap: true
Insert:
  Revisits: Uniform:  min=1,max=1000000
  Visits: Fixed:  key=1
  Row Population Ratio: Ratio: divisor=1.000000;delegate=Fixed:  key=1
  Batch Type: not batching
Columns:
  Max Columns Per Key: 5
  Column Names: [C0, C1, C2, C3, C4]
  Comparator: AsciiType
  Timestamp: null
  Variable Column Count: false
  Slice: false
  Size Distribution: Fixed:  key=34
  Count Distribution: Fixed:  key=5
Errors:
  Ignore: false
  Tries: 10
Log:
  No Summary: false
  No Settings: false
  File: null
  Interval Millis: 1000
  Level: NORMAL
Mode:
  API: JAVA_DRIVER_NATIVE
  Connection Style: CQL_PREPARED
  Protocol Version: V5
  Username: null
  Password: null
  Auth Provide Class: null
  Max Pending Per Connection: 128
  Connections Per Host: 8
  Compression: NONE
Node:
  Nodes: [192.168.100.151]
  Is White List: false
  Datacenter: null
Schema:
  Keyspace: keyspace1
  Replication Strategy: org.apache.cassandra.locator.SimpleStrategy
  Replication Strategy Options: {replication_factor=1}
  Table Compression: null
  Table Compaction Strategy: null
  Table Compaction Strategy Options: {}
Transport:
  Truststore: null
  Truststore Password: null
  Keystore: null
  Keystore Password: null
  SSL Protocol: TLS
  SSL Algorithm: null
  SSL Ciphers: TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA
Port:
  Native Port: 9042
  JMX Port: 7199
JMX:
  Username: null
  Password: *not set*
Graph:
  File: null
  Revision: unknown
  Title: null
  Operation: WRITE
TokenRange:
  Wrap: false
  Split Factor: 1
Credentials file:
  File: *not set*
  CQL username: *not set*
  CQL password: *not set*
  JMX username: *not set*
  JMX password: *not set*
  Transport truststore password: *not set*
  Transport keystore password: *not set*
Reporting:
  Output frequency: 1s
  Header frequency: *not set*

Connected to cluster: formation, max pending requests per connection 128, max connections per host 8
Datacenter: datacenter1; Host: /10.17.64.5:9042; Rack: rack1
Created keyspaces. Sleeping 1s for propagation.
Sleeping 2s...
Running WRITE with 1 threads for 25000 iteration

type                                               total ops,    op/s,    pk/s,   row/s,    mean,     med,     .95,     .99,    .999,     max,   time,   stderr, errors,  gc: #,  max ms,  sum ms,  sdv ms,      mb
WARN  18:52:05,485 Query 'com.datastax.driver.core.Statement$1@63e6389a;' generated server side warning(s): `USE <keyspace>` with prepared statements is considered to be an anti-pattern due to ambiguity in non-qualified table names. Please consider removing instances of `Session#setKeyspace(<keyspace>)`, `Session#execute("USE <keyspace>")` and `cluster.newSession(<keyspace>)` from your code, and always use fully qualified table names (e.g. <keyspace>.<table>). Keyspace used: keyspace1, statement keyspace: keyspace1, statement id: e1d2f4aa887d02751110f92a52de4f20
total,                                                    36,      36,      36,      36,    14.0,     1.6,    90.6,    98.0,    98.0,    98.0,    1.0,  0.00000,      0,      0,       0,       0,       0,       0
...
....
......

total,                                                 23718,     794,     794,     794,     1.2,     0.5,     0.6,    57.1,    76.7,    76.7,   50.0,  0.07457,      0,      0,       0,       0,       0,       0
total,                                                 24660,     942,     942,     942,     1.1,     0.4,     0.6,    54.6,    58.4,    58.4,   51.0,  0.07400,      0,      0,       0,       0,       0,       0
total,                                                 25000,    1028,    1028,    1028,     1.0,     0.5,     0.6,     0.7,    59.1,    59.1,   51.3,  0.07521,      0,      0,       0,       0,       0,       0


Results:
Op rate                   :      487 op/s  [WRITE: 487 op/s]
Partition rate            :      487 pk/s  [WRITE: 487 pk/s]
Row rate                  :      487 row/s [WRITE: 487 row/s]
Latency mean              :    2.0 ms [WRITE: 2.0 ms]
Latency median            :    0.5 ms [WRITE: 0.5 ms]
Latency 95th percentile   :    0.8 ms [WRITE: 0.8 ms]
Latency 99th percentile   :   78.2 ms [WRITE: 78.2 ms]
Latency 99.9th percentile :   89.3 ms [WRITE: 89.3 ms]
Latency max               :   98.0 ms [WRITE: 98.0 ms]
Total partitions          :     25,000 [WRITE: 25,000]
Total errors              :          0 [WRITE: 0]
Total GC count            : 3
Total GC memory           : 215.107 MiB
Total GC time             :    0.1 seconds
Avg GC time               :   31.7 ms
StdDev GC time            :   35.0 ms
Total operation time      : 00:00:51

END
```

##### Observer les métriques ici : 
```text
http://localhost:3000/d/PbhrxMuiz/cassandra-dashboard?orgId=1&from=now-5m&to=now&timezone=browser&var-cluster=formation&var-datacenter=$__all&refresh=5s
```
