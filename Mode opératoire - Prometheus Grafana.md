## Mode opératoire - Prometheus Grafana

Cassandra 4.1.11 avec Docker Compose (Cluster 2 nœuds)

Cluster Cassandra déployé via Docker Compose avec 4 nœuds sur 2 racks différents dans 2 datacenters.


#### 1°) Démarrage du cluster

#### Étape 1 : Préparation de l'environnement

```bash
cd ~
# A ne pas faire en production évidemment : 
sudo systemctl stop apparmor
sudo systemctl stop ufw
sudo systemctl disable apparmor
sudo systemctl disable ufw
```

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

#### Vérifier que les 2 conteneurs sont UP sinon attendre (non listé ou encore en train de joindre : 'UJ')
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
bash cassandra-stress write no-warmup n=25000 cl=one -rate threads=3 -node 10.17.64.5
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
  Thread Count: 3
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
  CQL Version: CQL3
  Protocol Version: V5
  Username: null
  Password: null
  Auth Provide Class: null
  Max Pending Per Connection: 128
  Connections Per Host: 8
  Compression: NONE
Node:
  Nodes: [10.17.64.5]
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
  truststore=null; truststore-password=null; keystore=null; keystore-password=null; ssl-protocol=TLS; ssl-alg=null; ssl-ciphers=TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA;
Port:
  Native Port: 9042
  JMX Port: 7199
Graph:
  File: null
  Revision: unknown
  Title: null
  Operation: WRITE
TokenRange:
  Wrap: false
  Split Factor: 1

Connected to cluster: archivage-db, max pending requests per connection 128, max connections per host 8
Datacenter: ccu; Host: /10.17.64.5:9042; Rack: rack1
Datacenter: ccu; Host: /10.17.64.6:9042; Rack: rack1
Created keyspaces. Sleeping 1s for propagation.
Sleeping 2s...
Running WRITE with 3 threads for 25000 iteration
type                                               total ops,    op/s,    pk/s,   row/s,    mean,     med,     .95,     .99,    .999,     max,   time,   stderr, errors,  gc: #,  max ms,  sum ms,  sdv ms,      mb
WARN  15:43:31,966 Query 'com.datastax.driver.core.Statement$1@d6b44c4;' generated server side warning(s): `USE <keyspace>` with prepared statements is considered to be an anti-pattern due to ambiguity in non-qualified table names. Please consider removing instances of `Session#setKeyspace(<keyspace>)`, `Session#execute("USE <keyspace>")` and `cluster.newSession(<keyspace>)` from your code, and always use fully qualified table names (e.g. <keyspace>.<table>). Keyspace used: keyspace1, statement keyspace: keyspace1, statement id: e1d2f4aa887d02751110f92a52de4f20
total,                                                     4,       4,       4,       4,    55.4,     1.8,   109.2,   109.2,   109.2,   109.2,    1.0,  0.00000,      0,      0,       0,       0,       0,       0
total,                                                   164,     160,     160,     160,    19.1,     1.6,    93.1,    96.9,   193.5,   193.5,    2.0,  0.67580,      0,      0,       0,       0,       0,       0
total,                                                   401,     237,     237,     237,    12.6,     1.3,    91.4,    94.2,    94.4,    94.4,    3.0,  0.40796,      0,      0,       0,       0,       0,       0
total,                                                   645,     244,     244,     244,    12.2,     1.1,    91.5,    93.8,    95.1,    95.1,    4.0,  0.29149,      0,      1,       3,       3,       0,      37
total,                                                  1000,     355,     355,     355,     8.5,     0.9,    89.3,    92.4,    92.8,    92.8,    5.0,  0.25182,      0,      0,       0,       0,       0,       0
total,                                                  1334,     334,     334,     334,     8.9,     0.8,    90.3,    92.0,    95.9,    95.9,    6.0,  0.20908,      0,      0,       0,       0,       0,       0
total,                                                  1692,     358,     358,     358,     8.4,     0.8,    91.0,    93.1,    93.3,    93.3,    7.0,  0.18014,      0,      0,       0,       0,       0,       0
total,                                                  2082,     390,     390,     390,     7.6,     0.7,    90.8,    92.1,    92.4,    92.4,    8.0,  0.16028,      0,      0,       0,       0,       0,       0
total,                                                  2480,     398,     398,     398,     7.6,     0.7,    89.7,    93.5,    93.8,    93.8,    9.0,  0.14409,      0,      1,      88,      88,       0,      39
total,                                                  2928,     448,     448,     448,     6.7,     0.7,    88.5,    91.0,    93.4,    93.4,   10.0,  0.13390,      0,      0,       0,       0,       0,       0
total,                                                  3334,     406,     406,     406,     7.4,     0.7,    90.0,    92.7,    93.1,    93.1,   11.0,  0.12185,      0,      1,       4,       4,       0,      37
total,                                                  3791,     457,     457,     457,     6.5,     0.7,    88.8,    92.5,    92.9,    92.9,   12.0,  0.11382,      0,      0,       0,       0,       0,       0
total,                                                  4321,     530,     530,     530,     5.6,     0.6,    85.3,    90.4,    93.3,    93.3,   13.0,  0.11042,      0,      0,       0,       0,       0,       0
total,                                                  4814,     493,     493,     493,     5.9,     0.6,    88.1,    91.2,    91.8,    91.8,   14.0,  0.10407,      0,      1,       4,       4,       0,      39
total,                                                  5127,     313,     313,     313,     9.5,     0.6,    91.8,    97.7,   100.2,   100.2,   15.0,  0.09786,      0,      0,       0,       0,       0,       0
total,                                                  5436,     309,     309,     309,     9.7,     0.6,    92.5,    99.3,   100.1,   100.1,   16.0,  0.09245,      0,      0,       0,       0,       0,       0
total,                                                  5910,     474,     474,     474,     6.4,     0.6,    89.7,    93.1,    99.9,    99.9,   17.0,  0.08788,      0,      0,       0,       0,       0,       0
total,                                                  6203,     293,     293,     293,     9.9,     0.6,    92.0,    98.4,    99.1,    99.1,   18.0,  0.08415,      0,      1,       4,       4,       0,      40
total,                                                  6679,     476,     476,     476,     6.4,     0.6,    89.6,    91.9,   131.5,   131.5,   19.0,  0.08093,      0,      0,       0,       0,       0,       0
total,                                                  7291,     612,     612,     612,     4.9,     0.6,     1.4,    90.8,    92.9,    92.9,   20.0,  0.08168,      0,      0,       0,       0,       0,       0
total,                                                  7793,     502,     502,     502,     5.9,     0.6,    87.9,    92.0,    92.7,    92.7,   21.0,  0.07827,      0,      0,       0,       0,       0,       0
total,                                                  8238,     445,     445,     445,     6.7,     0.6,    88.7,    92.5,    94.0,    94.0,   22.0,  0.07455,      0,      1,      93,      93,       0,      36
total,                                                  8654,     416,     416,     416,     7.2,     0.6,    89.6,    92.3,    97.7,    97.7,   23.0,  0.07112,      0,      1,       8,       8,       0,      37
total,                                                  8877,     223,     223,     223,    13.3,     0.6,    93.0,   100.0,   131.7,   131.7,   24.0,  0.07108,      0,      0,       0,       0,       0,       0
total,                                                  9159,     282,     282,     282,    10.7,     0.6,    92.7,   100.0,   100.1,   100.1,   25.0,  0.06953,      0,      0,       0,       0,       0,       0
total,                                                  9625,     466,     466,     466,     6.4,     0.6,    89.7,    91.9,    92.9,    92.9,   26.0,  0.06691,      0,      0,       0,       0,       0,       0
total,                                                 10077,     452,     452,     452,     6.4,     0.6,    89.7,    92.8,    95.7,    95.7,   27.0,  0.06439,      0,      1,       4,       4,       0,      39
total,                                                 10502,     425,     425,     425,     7.1,     0.6,    90.9,    93.1,    96.2,    96.2,   28.0,  0.06197,      0,      0,       0,       0,       0,       0
total,                                                 10890,     388,     388,     388,     7.7,     0.6,    91.3,    96.4,    98.8,    98.8,   29.0,  0.05977,      0,      0,       0,       0,       0,       0
total,                                                 11316,     426,     426,     426,     7.0,     0.6,    90.1,    95.0,    99.5,    99.5,   30.0,  0.05769,      0,      0,       0,       0,       0,       0
total,                                                 11865,     549,     549,     549,     5.5,     0.5,    88.1,    91.4,    93.1,    93.1,   31.0,  0.05678,      0,      0,       0,       0,       0,       0
total,                                                 12322,     457,     457,     457,     6.5,     0.5,    89.6,    92.4,    93.8,    93.8,   32.0,  0.05498,      0,      1,      92,      92,       0,      39
total,                                                 12676,     354,     354,     354,     8.4,     0.5,    88.7,    98.0,   100.0,   100.0,   33.0,  0.05350,      0,      1,       9,       9,       0,      36
total,                                                 13048,     372,     372,     372,     8.1,     0.5,    90.8,    99.0,   100.1,   100.1,   34.0,  0.05198,      0,      0,       0,       0,       0,       0
total,                                                 13470,     422,     422,     422,     7.1,     0.5,    68.2,   100.1,   100.1,   100.1,   35.0,  0.05043,      0,      2,       8,       8,       0,      78
total,                                                 13842,     372,     372,     372,     8.0,     0.5,    87.7,    95.6,   100.1,   100.1,   36.0,  0.04908,      0,      0,       0,       0,       0,       0
total,                                                 14359,     517,     517,     517,     5.8,     0.5,    61.0,    90.8,    92.4,    92.4,   37.0,  0.04814,      0,      1,       3,       3,       0,      42
total,                                                 14801,     442,     442,     442,     6.8,     0.5,    64.9,    98.2,   129.7,   129.7,   38.0,  0.04689,      0,      2,       6,       6,       0,      79
total,                                                 15087,     286,     286,     286,    10.4,     0.5,    90.4,   100.0,   100.4,   100.4,   39.0,  0.04649,      0,      0,       0,       0,       0,       0
total,                                                 15518,     431,     431,     431,     7.0,     0.5,    67.8,    98.8,   100.1,   100.1,   40.0,  0.04529,      0,      1,       3,       3,       0,      40
total,                                                 16184,     666,     666,     666,     4.5,     0.5,    32.1,    89.9,    91.6,    91.6,   41.0,  0.04651,      0,      1,      88,      88,       0,      39
total,                                                 16839,     655,     655,     655,     4.6,     0.5,    31.9,    89.9,    92.9,    92.9,   42.0,  0.04716,      0,      1,       3,       3,       0,      39
total,                                                 17413,     574,     574,     574,     5.2,     0.5,    61.9,    90.6,    99.2,    99.2,   43.0,  0.04666,      0,      1,       3,       3,       0,      40
total,                                                 18150,     737,     737,     737,     4.1,     0.5,     1.2,    88.3,    90.3,    90.3,   44.0,  0.04815,      0,      1,       4,       4,       0,      39
total,                                                 18821,     671,     671,     671,     4.5,     0.5,     1.6,    89.5,    90.6,    90.6,   45.0,  0.04831,      0,      1,       3,       3,       0,      39
total,                                                 19486,     665,     665,     665,     4.5,     0.5,    32.3,    87.8,    93.1,    93.1,   46.0,  0.04827,      0,      1,       3,       3,       0,      40
total,                                                 20164,     678,     678,     678,     4.4,     0.5,    32.0,    90.8,    92.1,    92.1,   47.0,  0.04824,      0,      0,       0,       0,       0,       0
total,                                                 21051,     887,     887,     887,     3.3,     0.5,    20.1,    65.9,    67.6,    67.6,   48.0,  0.05049,      0,      1,       4,       4,       0,      39
total,                                                 21967,     916,     916,     916,     3.3,     0.5,    16.2,    78.2,    96.5,    96.5,   49.0,  0.05295,      0,      1,       5,       5,       0,      39
total,                                                 23011,    1044,    1044,    1044,     2.9,     0.5,    14.7,    66.9,    68.4,    68.5,   50.0,  0.05615,      0,      1,       3,       3,       0,      39
total,                                                 24075,    1064,    1064,    1064,     2.8,     0.5,    13.3,    67.9,    80.9,    85.7,   51.0,  0.05896,      0,      0,       0,       0,       0,       0
total,                                                 25000,    1232,    1232,    1232,     2.5,     0.5,     1.3,    67.9,    83.6,    83.6,   51.8,  0.06385,      0,      0,       0,       0,       0,       0


Results:
Op rate                   :      483 op/s  [WRITE: 483 op/s]
Partition rate            :      483 pk/s  [WRITE: 483 pk/s]
Row rate                  :      483 row/s [WRITE: 483 row/s]
Latency mean              :    6.1 ms [WRITE: 6.1 ms]
Latency median            :    0.6 ms [WRITE: 0.6 ms]
Latency 95th percentile   :   83.8 ms [WRITE: 83.8 ms]
Latency 99th percentile   :   92.8 ms [WRITE: 92.8 ms]
Latency 99.9th percentile :  100.0 ms [WRITE: 100.0 ms]
Latency max               :  193.5 ms [WRITE: 193.5 ms]
Total partitions          :     25,000 [WRITE: 25,000]
Total errors              :          0 [WRITE: 0]
Total GC count            : 25
Total GC memory           : 970.863 MiB
Total GC time             :    0.4 seconds
Avg GC time               :   17.8 ms
StdDev GC time            :   31.7 ms
Total operation time      : 00:00:51

END
```

##### Autre essai : 
```bash
bash cassandra-stress write no-warmup n=25000 cl=QUORUM -rate threads=5 -node 10.17.64.6
```


###### Affichage en retour du 2nd benchmark : 
```text

Results:
Op rate                   :      845 op/s  [WRITE: 845 op/s]
Partition rate            :      845 pk/s  [WRITE: 845 pk/s]
Row rate                  :      845 row/s [WRITE: 845 row/s]
Latency mean              :    5.9 ms [WRITE: 5.9 ms]
Latency median            :    0.5 ms [WRITE: 0.5 ms]
Latency 95th percentile   :   84.2 ms [WRITE: 84.2 ms]
Latency 99th percentile   :   92.7 ms [WRITE: 92.7 ms]
Latency 99.9th percentile :   99.9 ms [WRITE: 99.9 ms]
Latency max               :  192.4 ms [WRITE: 192.4 ms]
Total partitions          :     25,000 [WRITE: 25,000]
Total errors              :          0 [WRITE: 0]
Total GC count            : 12
Total GC memory           : 463.992 MiB
Total GC time             :    0.1 seconds
Avg GC time               :   11.0 ms
StdDev GC time            :   23.3 ms
Total operation time      : 00:00:29

END
```


##### Observer les métriques ici : 
```text
http://localhost:3000/d/PbhrxMuiz/cassandra-dashboard?orgId=1&from=now-5m&to=now&timezone=browser&var-cluster=formation&var-datacenter=$__all&refresh=5s
```
