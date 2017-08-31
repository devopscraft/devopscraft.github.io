---
layout: post
title:  "Postgresql Aktív-Passzív Cluster"
date:   2017-08-31
author: DevOpsCraft
categories: Jekyll
tags:	pgsql cluster
cover:  "/assets/instacode.png"
---

A Postgresql vagy pgsql egy népszerű relációs adatbázis. Eredetileg a Postgresql nem tartalmazott replikációs lehetőséget. Hivatalosan erre külső eszközöket ajánlottak, mint a pgpool vagy a Slony. Ez egy komoly hátránya volt a Postgresqlnek és a felhasználók emiatt inkább a MySQL-t választották ahol ez már nagyon régóta elérhető.

A 9.0-ás verzió megjelenésével vált elérhetővé először a beépített replikációs funkció, amit úgynevezett WAL szegmensek átvitelével valósítottak meg. Ez a megoldás egy aszinkron cluster kialakítását tette lehetővé. A Slave mindig egy kis időbeli eltolódással kapja meg a változásokat. Írni és olvasni lehet a master nod-on de a slave nod-on csak olvasni. Nézzük meg hogyan is lehet ezt létrehozni.

Először telepítsük mind a két szerverre a legfrissebb Postgresqlt. Ehhez szükségünk lesz a postgresql repository hozzáadására.

```bash
echo 'deb http://apt.postgresql.org/pub/repos/apt/ xenial-pgdg main' | tee /etc/apt/sources.list.d/postgresql.list
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
apt-get update
apt-get install -y postgresql-9.6 postgresql-contrib-9.6

systemctl enable postgresql
```

A master szerver IP-je 192.168.10.10 lesz. Engedélyezzük az arcive módot. Így kezd a rendszer WAL szegmenseket generálni. Majd engedélyezzük ezek átvitelét.

```bash
cd /etc/postgresql/9.6/main/
nano postgresql.conf

listen_addresses = '192.168.10.10'
wal_level = hot_standby
synchronous_commit = local
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/9.6/main/archive/%f'
max_wal_senders = 2
wal_keep_segments = 10
synchronous_standby_names = 'pgslave001'
```

Hozzuk létre a szükséges mappákat:

```bash
mkdir -p /var/lib/postgresql/9.6/main/archive/
chmod 700 /var/lib/postgresql/9.6/main/archive/
chown -R postgres:postgres /var/lib/postgresql/9.6/main/archive/
```

Módosítsuk az hálózati elérési engedélyeket a postgresql beállításaiban.

```bash
nano pg_hba.conf

# Localhost
host    replication     replica          127.0.0.1/32            md5

# PostgreSQL Master IP-je
host    replication     replica          192.168.10.10/32            md5

# PostgreSQL SLave IP-je
host    replication     replica          192.168.10.11/32            md5
```

Hozzunklétre egy felhasználót a postgresqlben aminek a nevében vágbemegy az átvitel. A felhasználónév ''replica'' lesz a jelszó pedig 'aqwe123@'.

```bash
systemctl restart postgresql

su - postgres
psql

CREATE USER replica REPLICATION LOGIN ENCRYPTED PASSWORD 'aqwe123@';
```
