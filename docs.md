# Guide d'installation Domibus 5.2 — Version OPTIMISÉE (V3)

**Projet** : Host-to-Host Bank of Africa — Access Point AS4 (Corner C2 client)
**OS cible** : Ubuntu 24.04 LTS (64 bits)
**Prérequis** : Oracle JDK 21 · Apache Tomcat 10.1 (embarqué) · MySQL 8
**Identifiants console finaux** : `admin` / `DomibusAmoaman2026`
**Statut** : Procédure testée de bout en bout sur VM réelle — intègre TOUTES les corrections

---

> **⚡ NOUVEAUTÉS DE CETTE VERSION (V3) — lis avant de commencer**
>
> Cette version corrige 5 pièges réels rencontrés lors d'une installation précédente :
>
> 1. **L'utilisateur admin est créé par Domibus lui-même au 1er démarrage** — PAS par le script SQL. La table `TB_USER` est donc NORMALEMENT vide juste après les scripts SQL. Ne pas s'en inquiéter, ne pas tenter de l'insérer manuellement.
> 2. **`sudo` ne fonctionne PAS en session `domibus`** (compte applicatif sans privilèges). Chaque commande indique clairement l'utilisateur : 🟦 `domi` (avec sudo) ou 🟩 `domibus` (sans sudo).
> 3. **Le bloc MySQL ne doit être ajouté qu'UNE fois** dans `domibus.properties` (ajout idempotent intégré).
> 4. **`JAVA_HOME` contient le numéro de version exact** (`jdk-21.0.11-oracle-x64`). On utilise une détection automatique pour ne jamais se tromper.
> 5. **`apache2-utils` (htpasswd) doit être installé AVANT** de générer le hash du mot de passe admin, sinon le hash est vide et le login échoue.
>
> **Convention d'utilisateur dans tout ce guide :**
> - 🟦 **`domi`** = ton compte admin avec droits sudo (celui de connexion SSH)
> - 🟩 **`domibus`** = compte applicatif (lancé via `sudo su - domibus`), JAMAIS de sudo dedans

---

## Table des matières

1. [Architecture & ordre des opérations](#1-architecture--ordre-des-opérations)
2. [Étape 1 — Préparation système](#2-étape-1--préparation-système-)
3. [Étape 2 — Oracle JDK 21](#3-étape-2--oracle-jdk-21-)
4. [Étape 3 — MySQL 8 + configuration](#4-étape-3--mysql-8--configuration-)
5. [Étape 4 — Base de données Domibus](#5-étape-4--base-de-données-domibus-)
6. [Étape 5 — Téléchargement Domibus & scripts](#6-étape-5--téléchargement-domibus--scripts-)
7. [Étape 6 — Initialisation du schéma SQL](#7-étape-6--initialisation-du-schéma-sql-)
8. [Étape 7 — Driver JDBC MySQL](#8-étape-7--driver-jdbc-mysql-)
9. [Étape 8 — Configuration domibus.properties](#9-étape-8--configuration-domibusproperties-)
10. [Étape 9 — Configuration JVM (setenv.sh)](#10-étape-9--configuration-jvm-setenvsh-)
11. [Étape 10 — Premier démarrage (crée l'admin)](#11-étape-10--premier-démarrage-crée-ladmin-)
12. [Étape 11 — Mot de passe admin DomibusAmoaman2026](#12-étape-11--mot-de-passe-admin-)
13. [Étape 12 — Service systemd](#13-étape-12--service-systemd-)
14. [Étape 13 — Accès à la console](#14-étape-13--accès-à-la-console-)
15. [Annexe A — Récapitulatif des chemins](#15-annexe-a--récapitulatif-des-chemins)
16. [Annexe B — Diagnostic](#16-annexe-b--diagnostic)
17. [Annexe C — Sécurité](#17-annexe-c--sécurité)
18. [Annexe D — Prochaines étapes H2H BOA](#18-annexe-d--prochaines-étapes-h2h-boa)

---

## 1. Architecture & ordre des opérations

```text
┌──────────────────────────────────────────────────────────────┐
│                  VPS Ubuntu 24.04 LTS                         │
│  ┌────────────────────────┐      ┌─────────────────────────┐ │
│  │ Domibus 5.2 (JEE10)    │      │      MySQL 8            │ │
│  │ Tomcat 10.1 embarqué   │◄────►│  domibus_schema        │ │
│  │ /opt/domibus/tomcat/   │ JDBC │  utf8mb4_bin           │ │
│  │   domibus/             │      │  user: edelivery_user  │ │
│  │ port 8080 (HTTP)       │      │  port 3306             │ │
│  └────────────────────────┘      └─────────────────────────┘ │
│              ▲ Oracle JDK 21                                  │
└──────────────────────────────────────────────────────────────┘
```

> **⚠️ Changement d'ordre important vs versions précédentes** : la définition du mot de passe admin se fait **APRÈS le premier démarrage de Domibus** (étape 11), car c'est Domibus qui crée l'utilisateur `admin` à son initialisation. Ne pas chercher l'admin en base avant cela.

| # | Étape | Utilisateur | Durée |
|---|-------|-------------|-------|
| 1 | Préparation OS | 🟦 domi | 10 min |
| 2 | Oracle JDK 21 | 🟦 domi | 10 min |
| 3 | MySQL 8 + config | 🟦 domi | 15 min |
| 4 | Base de données | 🟦 domi | 5 min |
| 5 | Téléchargement Domibus | 🟩 domibus | 10 min |
| 6 | Schéma SQL | 🟩 domibus | 5 min |
| 7 | Driver JDBC | 🟦 domi | 5 min |
| 8 | domibus.properties | 🟩 domibus | 10 min |
| 9 | JVM setenv.sh | 🟩 domibus | 5 min |
| 10 | 1er démarrage (crée admin) | 🟩 domibus | 5 min |
| 11 | Mot de passe admin | 🟦 domi | 5 min |
| 12 | Service systemd | 🟦 domi | 10 min |
| 13 | Console | navigateur | 5 min |

---

## 2. Étape 1 — Préparation système 🟦

Toutes ces commandes en tant que **`domi`** (compte sudo).

### 2.1 Mise à jour et paquets

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y unzip curl wget openssl net-tools lsof vim nano ca-certificates gnupg lsb-release apache2-utils
```

> 💡 **CORRECTION #5** : on installe `apache2-utils` (qui fournit `htpasswd`) **dès maintenant**, pour qu'il soit disponible à l'étape 11. C'est ce qui manquait et faisait échouer le hash du mot de passe.

### 2.2 Fuseau horaire UTC

```bash
sudo timedatectl set-timezone UTC
timedatectl
```

### 2.3 Utilisateur applicatif `domibus`

```bash
sudo useradd -r -m -d /opt/domibus -s /bin/bash domibus
```

> 💡 **CORRECTION #2** : cet utilisateur n'a **volontairement pas** les droits sudo. Toute commande à exécuter en tant que `domibus` ne devra JAMAIS contenir `sudo`.

### 2.4 Arborescence

```bash
sudo mkdir -p /opt/domibus/{tomcat,scripts,backups,payloads,fs-plugin,conf,plugins}
sudo mkdir -p /opt/domibus/fs-plugin/{OUT,IN}
sudo chown -R domibus:domibus /opt/domibus
sudo chmod 750 /opt/domibus
```

### 2.5 Limites système

```bash
sudo tee -a /etc/security/limits.conf > /dev/null <<EOF
domibus  soft  nofile  65536
domibus  hard  nofile  65536
domibus  soft  nproc   4096
domibus  hard  nproc   4096
EOF
```

> ✅ **Contrôle** : `ls -la /opt/domibus/` montre les dossiers appartenant à `domibus:domibus`.

---

## 3. Étape 2 — Oracle JDK 21 🟦

### 3.1 Téléchargement et installation

```bash
cd /tmp
wget https://download.oracle.com/java/21/latest/jdk-21_linux-x64_bin.deb -O jdk-21_linux-x64.deb
sudo dpkg -i jdk-21_linux-x64.deb
sudo apt install -f -y
```

### 3.2 Vérification + détection auto du chemin

```bash
java -version
JAVA_REAL=$(dirname $(dirname $(readlink -f $(which java))))
echo "JAVA_HOME détecté : $JAVA_REAL"
```

> 💡 **CORRECTION #4** : le chemin réel est du type `/usr/lib/jvm/jdk-21.0.11-oracle-x64` (avec le numéro de patch). **Ne JAMAIS coder en dur `jdk-21-oracle-x64`** — toujours utiliser cette détection `readlink`. On la réutilisera partout.

### 3.3 Variable globale

```bash
sudo tee /etc/profile.d/java.sh > /dev/null <<EOF
export JAVA_HOME=$JAVA_REAL
export PATH=\$JAVA_HOME/bin:\$PATH
EOF
sudo chmod 644 /etc/profile.d/java.sh
source /etc/profile.d/java.sh
echo $JAVA_HOME
```

> ✅ **Contrôle** : `java -version` affiche `21.0.x`, `echo $JAVA_HOME` renvoie le chemin avec le numéro de version.

---

## 4. Étape 3 — MySQL 8 + configuration 🟦

### 4.1 Installation

```bash
sudo apt install -y mysql-server
sudo systemctl enable --now mysql
sudo systemctl status mysql --no-pager
```

### 4.2 Sécurisation

```bash
sudo mysql_secure_installation
```

| Question | Réponse |
|----------|---------|
| VALIDATE PASSWORD component | `y` puis `2` (STRONG) |
| Set root password | **Sur Ubuntu, root utilise auth_socket : il est normal qu'aucun mot de passe ne soit demandé / que ce soit ignoré.** |
| Remove anonymous users | `y` |
| Disallow root login remotely | `y` |
| Remove test database | `y` |
| Reload privilege tables | `y` |

> 💡 **Important** : sur Ubuntu 24.04, root MySQL s'authentifie via **auth_socket**. On se connecte avec `sudo mysql` (sans `-p`, sans mot de passe). Ne jamais utiliser `mysql -uroot -p` — ça donnerait `Access denied`.

### 4.3 Configuration critique ⚠️

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Sous `[mysqld]`, ajouter :

```ini
[mysqld]
# === Collation OBLIGATOIRE pour Domibus (sinon "Illegal mix of collations") ===
character-set-server         = utf8mb4
collation-server             = utf8mb4_bin

# === Autorise fonctions/triggers des scripts Domibus (sinon "SUPER privilege") ===
log_bin_trust_function_creators = 1

# === Fuseau horaire ===
default-time-zone            = '+00:00'

# === Dimensionnement (adapter à la RAM du VPS) ===
max_connections              = 300
max_allowed_packet           = 64M
innodb_buffer_pool_size      = 2G
innodb_log_file_size         = 256M
innodb_flush_log_at_trx_commit = 2
```

### 4.4 Redémarrage

```bash
sudo systemctl restart mysql
sudo systemctl status mysql --no-pager
```

### 4.5 Vérification critique

```bash
sudo mysql -e "SHOW VARIABLES LIKE 'collation_server'; SHOW VARIABLES LIKE 'character_set_server'; SHOW VARIABLES LIKE 'log_bin_trust_function_creators';"
```

> ✅ **Contrôle BLOQUANT** — tu dois obtenir EXACTEMENT :
> - `collation_server = utf8mb4_bin`
> - `character_set_server = utf8mb4`
> - `log_bin_trust_function_creators = ON`
>
> Si l'une de ces valeurs est fausse, **NE CONTINUE PAS** : l'init du schéma échouera. Reprends le 4.3.

---

## 5. Étape 4 — Base de données Domibus 🟦

### 5.1 Génération d'un mot de passe conforme à la policy STRONG

```bash
# IMPORTANT : ce mot de passe contient OBLIGATOIREMENT un caractère spécial (-)
# pour satisfaire la policy STRONG de MySQL. Ne PAS retirer le "-Aa9".
EDELIVERY_PWD="Db$(openssl rand -hex 14)-Aa9"
echo "==> Mot de passe edelivery_user : $EDELIVERY_PWD"

echo "$EDELIVERY_PWD" | sudo tee /root/.edelivery_pwd > /dev/null
sudo chmod 600 /root/.edelivery_pwd
```

> 💡 **Piège évité** : un mot de passe uniquement alphanumérique (sans caractère spécial) est REJETÉ par la policy STRONG avec `ERROR 1819`. Le suffixe `-Aa9` garantit majuscule + minuscule + chiffre + spécial. Le `-` est sûr partout (shell, `.my.cnf`, JDBC, properties).

⚠️ **Copie ce mot de passe dans ton gestionnaire de secrets maintenant.**

### 5.2 Création schéma + utilisateur (auth_socket, sans -p)

```bash
sudo mysql <<EOF
CREATE DATABASE IF NOT EXISTS domibus_schema
  DEFAULT CHARACTER SET utf8mb4
  DEFAULT COLLATE utf8mb4_bin;

CREATE USER IF NOT EXISTS 'edelivery_user'@'localhost' IDENTIFIED BY '$EDELIVERY_PWD';
GRANT ALL PRIVILEGES ON domibus_schema.* TO 'edelivery_user'@'localhost';
FLUSH PRIVILEGES;

SHOW CREATE DATABASE domibus_schema;
EOF
```

> ✅ **Contrôle** : la sortie contient `DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_bin` et aucune erreur 1819.

### 5.3 Fichier `.my.cnf` (connexion sans mot de passe)

```bash
sudo tee /opt/domibus/.my.cnf > /dev/null <<EOF
[client]
user=edelivery_user
password="$EDELIVERY_PWD"

[mysql]
database=domibus_schema
EOF

sudo chown domibus:domibus /opt/domibus/.my.cnf
sudo chmod 600 /opt/domibus/.my.cnf
```

### 5.4 Test

```bash
sudo su - domibus -c "mysql -e 'SELECT CURRENT_USER();'"
```

> ✅ **Contrôle** : affiche `edelivery_user@localhost` sans demander de mot de passe.

---

## 6. Étape 5 — Téléchargement Domibus & scripts 🟩

On bascule en `domibus`.

```bash
sudo su - domibus
cd /opt/domibus
```

### 6.1 Domibus 5.2 (JEE10 Tomcat Full)

```bash
wget "https://ec.europa.eu/digital-building-blocks/artifact/repository/eDelivery/eu/domibus/domibus-msh-distribution/5.2-JEE10/domibus-msh-distribution-5.2-JEE10-tomcat-full.zip" \
  -O domibus-5.2-tomcat-full.zip
ls -lh domibus-5.2-tomcat-full.zip
```

Taille attendue : ~136 Mo.

### 6.2 Extraction (structure réelle)

> ⚠️ L'archive crée un sous-dossier `domibus/` qui EST le vrai Tomcat → il finit dans `/opt/domibus/tomcat/domibus/`.

```bash
cd /opt/domibus
unzip -q domibus-5.2-tomcat-full.zip -d ./tomcat-extract
mv ./tomcat-extract/* ./tomcat/
rm -rf ./tomcat-extract
ls /opt/domibus/tomcat/domibus/
```

> ✅ **Contrôle** : `/opt/domibus/tomcat/domibus/` contient `bin/ conf/ lib/ logs/ webapps/ temp/`.

### 6.3 Scripts SQL

```bash
cd /opt/domibus/scripts
wget "https://ec.europa.eu/digital-building-blocks/artifact/repository/eDelivery/eu/domibus/domibus-msh-sql-distribution/1.18/domibus-msh-sql-distribution-1.18.zip" \
  -O domibus-msh-sql-1.18.zip
unzip -q domibus-msh-sql-1.18.zip
ls -la /opt/domibus/scripts/sql-scripts/5.2/mysql/
```

> ✅ **Contrôle** : présence de `mysql-5.2.ddl` et `mysql-5.2-data.ddl`.

---

## 7. Étape 6 — Initialisation du schéma SQL 🟩

> ⚠️ Chaque script UNE SEULE FOIS, dans l'ordre, sur un schéma vide.

```bash
cd /opt/domibus/scripts/sql-scripts/5.2/mysql/
mysql domibus_schema < mysql-5.2.ddl
mysql domibus_schema < mysql-5.2-data.ddl
```

Les deux prompts reviennent **sans erreur**.

### Vérification (corrigée)

```bash
# Nombre de tables — DOIT être ~119
mysql domibus_schema -e "SELECT COUNT(*) AS nb_tables FROM information_schema.tables WHERE table_schema='domibus_schema';"

# Rôles MSH créés par le script
mysql domibus_schema -e "SELECT * FROM TB_D_MSH_ROLE;"
```

> ✅ **Contrôle** :
> - ~119 tables
> - `TB_D_MSH_ROLE` contient `SENDING` et `RECEIVING`
>
> 💡 **CORRECTION #1 — TRÈS IMPORTANT** : à ce stade, **`TB_USER` est VIDE, et c'est NORMAL.** L'utilisateur `admin` n'est PAS créé par les scripts SQL — il est créé automatiquement **par Domibus lui-même au premier démarrage** (étape 10). **N'essaie PAS d'insérer l'admin manuellement ici** : tu créerais des conflits. On le configurera proprement à l'étape 11, après le 1er démarrage.

---

## 8. Étape 7 — Driver JDBC MySQL 🟦

Le driver MySQL n'est PAS fourni dans l'archive Domibus — il faut l'ajouter.

### 8.1 Sortir de `domibus`, télécharger, installer

```bash
exit   # retour en domi

cd /tmp
wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.4.0/mysql-connector-j-8.4.0.jar
sudo mv mysql-connector-j-8.4.0.jar /opt/domibus/tomcat/domibus/lib/
sudo chown domibus:domibus /opt/domibus/tomcat/domibus/lib/mysql-connector-j-8.4.0.jar
sudo chmod 644 /opt/domibus/tomcat/domibus/lib/mysql-connector-j-8.4.0.jar
sudo ls -la /opt/domibus/tomcat/domibus/lib/mysql-connector-j-8.4.0.jar
```

> ✅ **Contrôle** : le `.jar` est présent dans `lib/`, propriétaire `domibus`.

---

## 9. Étape 8 — Configuration domibus.properties 🟩

```bash
sudo su - domibus
```

### 9.1 Sauvegarde

```bash
PROP=/opt/domibus/tomcat/domibus/conf/domibus/domibus.properties
cp $PROP ${PROP}.original
```

### 9.2 Commenter les anciennes lignes datasource (sans sudo !)

> 💡 **CORRECTION #2** : ces commandes sont lancées EN `domibus`, donc **PAS de `sudo`** (le fichier appartient déjà à `domibus`).

```bash
PROP=/opt/domibus/tomcat/domibus/conf/domibus/domibus.properties

sed -i -E 's|^(domibus\.datasource\.driverClassName=.*)|#\1|' $PROP
sed -i -E 's|^(domibus\.datasource\.url=.*)|#\1|' $PROP
sed -i -E 's|^(domibus\.datasource\.user=.*)|#\1|' $PROP
sed -i -E 's|^(domibus\.datasource\.password=.*)|#\1|' $PROP
sed -i -E 's|^(domibus\.entityManagerFactory\.jpaProperty\.hibernate\.dialect=.*)|#\1|' $PROP
sed -i -E 's|^(domibus\.database\.serverName=.*)|#\1|' $PROP
sed -i -E 's|^(domibus\.database\.port=.*)|#\1|' $PROP
sed -i -E 's|^(domibus\.database\.schema=.*)|#\1|' $PROP
```

### 9.3 Ajout idempotent du bloc MySQL

> 💡 **CORRECTION #3** : on vérifie d'abord si le bloc existe DÉJÀ avant de l'ajouter, pour éviter les doublons en cas de relance.

```bash
PROP=/opt/domibus/tomcat/domibus/conf/domibus/domibus.properties
EDELIVERY_PWD=$(cat /opt/domibus/.my.cnf | grep '^password=' | sed 's/^password=//' | tr -d '"')

if grep -q "# === BLOC MYSQL H2H BOA ===" "$PROP"; then
  echo "⚠️ Le bloc MySQL existe déjà — aucun ajout (idempotent)."
else
  tee -a "$PROP" > /dev/null <<EOF

# === BLOC MYSQL H2H BOA ===
# =====================================================================
# Configuration MySQL 8 — Projet H2H BOA (bloc authoritatif)
# =====================================================================
domibus.database.serverName=localhost
domibus.database.port=3306
domibus.database.schema=domibus_schema
domibus.datasource.driverClassName=com.mysql.cj.jdbc.Driver
domibus.datasource.url=jdbc:mysql://localhost:3306/domibus_schema?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8&allowPublicKeyRetrieval=true&connectionCollation=utf8mb4_bin
domibus.datasource.user=edelivery_user
domibus.datasource.password=$EDELIVERY_PWD
domibus.entityManagerFactory.jpaProperty.hibernate.dialect=org.hibernate.dialect.MySQLDialect
domibus.datasource.minPoolSize=5
domibus.datasource.maxPoolSize=100
domibus.attachment.storage.location=/opt/domibus/payloads
# SANDBOX UNIQUEMENT — réactiver en production :
domibus.certificate.revocation.check.strategies=
EOF
  echo "✅ Bloc MySQL ajouté."
fi

chmod 640 "$PROP"
```

> 💡 Le mot de passe est lu depuis `.my.cnf` (pas de `sudo cat /root/...` qui échouerait en session `domibus`).

### 9.4 Vérification

```bash
grep -E "^domibus\.(datasource|database\.|entityManager)" $PROP | grep -v "^#"
```

> ✅ **Contrôle** : tu vois UNE SEULE occurrence de chaque ligne MySQL (pas de doublon), notamment :
> - `domibus.datasource.driverClassName=com.mysql.cj.jdbc.Driver`
> - `domibus.datasource.url=jdbc:mysql://...connectionCollation=utf8mb4_bin`
>
> Et `grep -E "^[^#].*[Oo]racle" $PROP` ne renvoie rien.

---

## 10. Étape 9 — Configuration JVM (setenv.sh) 🟩

> 💡 **CORRECTION #4** : détection auto de `JAVA_HOME` intégrée dans le script.

```bash
JAVA_REAL=$(dirname $(dirname $(readlink -f $(which java))))
echo "JAVA_HOME utilisé : $JAVA_REAL"

cat > /opt/domibus/tomcat/domibus/bin/setenv.sh <<EOF
#!/bin/bash
# ============================================================
# Variables JVM Tomcat — Domibus 5.2 — Projet H2H BOA
# ============================================================
export JAVA_HOME=$JAVA_REAL
export CATALINA_HOME=/opt/domibus/tomcat/domibus
export CATALINA_BASE=/opt/domibus/tomcat/domibus

export CATALINA_OPTS="\$CATALINA_OPTS \\
  -Xms2g \\
  -Xmx4g \\
  -XX:MetaspaceSize=256m \\
  -XX:MaxMetaspaceSize=512m \\
  -XX:+UseG1GC \\
  -XX:+HeapDumpOnOutOfMemoryError \\
  -XX:HeapDumpPath=/opt/domibus/tomcat/domibus/logs \\
  -Ddomibus.config.location=/opt/domibus/tomcat/domibus/conf/domibus \\
  -Dfile.encoding=UTF-8 \\
  -Djava.awt.headless=true"

export CATALINA_OUT=/opt/domibus/tomcat/domibus/logs/catalina.out
EOF

chmod +x /opt/domibus/tomcat/domibus/bin/setenv.sh
grep JAVA_HOME /opt/domibus/tomcat/domibus/bin/setenv.sh
```

> ✅ **Contrôle** : `JAVA_HOME` affiche le chemin AVEC le numéro de version (ex. `jdk-21.0.11-oracle-x64`), pas `jdk-21-oracle-x64`.

---

## 11. Étape 10 — Premier démarrage (crée l'admin) 🟩

> 💡 **CORRECTION #1** : c'est CE démarrage qui crée l'utilisateur `admin`, le PMode par défaut et les certificats de test. Il faut donc démarrer Domibus AVANT de configurer le mot de passe admin.

### 11.1 Démarrer

```bash
/opt/domibus/tomcat/domibus/bin/startup.sh
```

### 11.2 Suivre les logs

```bash
tail -f /opt/domibus/tomcat/domibus/logs/catalina.out
```

Le démarrage prend **30 à 90 secondes**. Indicateurs de succès :

- ✅ `domibus-MSH Version [5.2-JEE10]`
- ✅ `A user with role [ROLE_ADMIN] already exists` OU création de l'admin
- ✅ `PMode Configuration successfully updated`
- ✅ `Server startup in [xxxxx] ms`
- ❌ AUCUNE `ClassNotFoundException: oracle.jdbc.driver.OracleDriver`
- ❌ AUCUNE `Cannot create PoolableConnectionFactory`

Quitter le `tail` avec `Ctrl+C` (ça n'arrête pas Domibus).

### 11.3 Test HTTP

```bash
curl -I http://localhost:8080/domibus/
```

> ✅ **Contrôle** : `HTTP/1.1 302` avec `Location: /domibus/console/index.html` (et non 404).

### 11.4 Vérifier que l'admin existe maintenant

```bash
mysql domibus_schema -e "SELECT USER_NAME, USER_ENABLED FROM TB_USER;"
```

> ✅ **Contrôle** : tu vois maintenant `admin` (créé par Domibus au démarrage). C'est ce qu'on attendait. On va lui poser notre mot de passe à l'étape suivante.

---

## 12. Étape 11 — Mot de passe admin 🟦

> 💡 **CORRECTIONS #2 + #5** : on sort en `domi` (pour utiliser sudo/htpasswd), et `htpasswd` est déjà installé (étape 2.1).

### 12.1 Sortir en `domi`

```bash
exit   # retour en domi
```

### 12.2 Vérifier que htpasswd est dispo

```bash
which htpasswd || sudo apt install -y apache2-utils
```

### 12.3 Générer le VRAI hash et l'appliquer

```bash
HASH=$(htpasswd -bnBC 10 "" "DomibusAmoaman2026" | tr -d ':\n' | sed 's/^\$2y/\$2a/')
echo "Hash généré : $HASH"
```

> ⚠️ **VÉRIFIE que `$HASH` n'est PAS vide et commence par `$2a$10$`.** Si vide → htpasswd n'est pas installé, refais 12.2.

```bash
sudo -u domibus mysql --defaults-file=/opt/domibus/.my.cnf domibus_schema -e \
"UPDATE TB_USER SET USER_PASSWORD='$HASH', DEFAULT_PASSWORD=0, ATTEMPT_COUNT=0, USER_ENABLED=1 WHERE USER_NAME='admin';"
```

### 12.4 Vérification

```bash
sudo -u domibus mysql --defaults-file=/opt/domibus/.my.cnf domibus_schema -e \
"SELECT USER_NAME, LEFT(USER_PASSWORD,7) AS prefix, DEFAULT_PASSWORD, USER_ENABLED FROM TB_USER WHERE USER_NAME='admin';"
```

> ✅ **Contrôle** : `prefix` = `$2a$10$`, `DEFAULT_PASSWORD` = `0x00`, `USER_ENABLED` = `0x01`.
> Pas besoin de redémarrer Tomcat (le hash est relu à chaque login).

---

## 13. Étape 12 — Service systemd 🟦

### 13.1 Arrêter le démarrage manuel

```bash
sudo -u domibus /opt/domibus/tomcat/domibus/bin/shutdown.sh
sleep 15
ps -ef | grep -i tomcat | grep -v grep   # ne doit rien afficher
```

### 13.2 Créer le service (JAVA_HOME auto-détecté)

```bash
JAVA_REAL=$(dirname $(dirname $(readlink -f $(which java))))

sudo tee /etc/systemd/system/domibus.service > /dev/null <<EOF
[Unit]
Description=Domibus 5.2 AS4 Access Point (Tomcat)
After=network.target mysql.service
Requires=mysql.service

[Service]
Type=forking
User=domibus
Group=domibus

Environment="JAVA_HOME=$JAVA_REAL"
Environment="CATALINA_PID=/opt/domibus/tomcat/domibus/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/domibus/tomcat/domibus"
Environment="CATALINA_BASE=/opt/domibus/tomcat/domibus"

ExecStart=/opt/domibus/tomcat/domibus/bin/startup.sh
ExecStop=/opt/domibus/tomcat/domibus/bin/shutdown.sh

LimitNOFILE=65536
LimitNPROC=4096
Restart=on-failure
RestartSec=30
TimeoutStartSec=180
TimeoutStopSec=60
PrivateTmp=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF
```

### 13.3 Activer et démarrer

```bash
sudo systemctl daemon-reload
sudo systemctl enable domibus
sudo systemctl start domibus
sudo systemctl status domibus --no-pager
```

> ✅ **Contrôle** : `active (running)`. Suivre : `sudo journalctl -u domibus -f`.

### 13.4 Gestion

| Action | Commande |
|--------|----------|
| Démarrer | `sudo systemctl start domibus` |
| Arrêter | `sudo systemctl stop domibus` |
| Redémarrer | `sudo systemctl restart domibus` |
| Statut | `sudo systemctl status domibus` |
| Logs | `sudo journalctl -u domibus -f` |

---

## 14. Étape 13 — Accès à la console 🌐

### 14.1 IP de la VM

```bash
ip -4 addr show | grep inet | grep -v 127.0.0.1
```

### 14.2 Connexion

```text
http://192.168.30.28:8080/domibus/
```

| Champ | Valeur |
|-------|--------|
| Utilisateur | `admin` |
| Mot de passe | `DomibusAmoaman2026` |

> ✅ **Contrôle final** : tu arrives sur le tableau de bord Domibus. **Installation terminée.** 🎉

### 14.3 Nettoyage des secrets temporaires

```bash
sudo shred -u /root/.edelivery_pwd
rm -f /opt/domibus/domibus-5.2-tomcat-full.zip
rm -f /opt/domibus/scripts/domibus-msh-sql-1.18.zip
```

(Le mot de passe `edelivery_user` reste dans `.my.cnf` et `domibus.properties`, c'est suffisant.)

---

## 15. Annexe A — Récapitulatif des chemins

| Élément | Chemin |
|---------|--------|
| CATALINA_HOME | `/opt/domibus/tomcat/domibus` |
| `domibus.properties` | `/opt/domibus/tomcat/domibus/conf/domibus/domibus.properties` |
| `setenv.sh` | `/opt/domibus/tomcat/domibus/bin/setenv.sh` |
| Driver JDBC | `/opt/domibus/tomcat/domibus/lib/mysql-connector-j-*.jar` |
| `catalina.out` | `/opt/domibus/tomcat/domibus/logs/catalina.out` |
| `domibus.log` | `/opt/domibus/tomcat/domibus/logs/domibus.log` |
| Scripts SQL | `/opt/domibus/scripts/sql-scripts/5.2/mysql/` |
| Payloads | `/opt/domibus/payloads/` |
| `.my.cnf` | `/opt/domibus/.my.cnf` |
| Service systemd | `/etc/systemd/system/domibus.service` |

---

## 16. Annexe B — Diagnostic

```bash
# État
sudo systemctl status domibus
sudo ss -tlnp | grep 8080

# Logs
sudo tail -f /opt/domibus/tomcat/domibus/logs/catalina.out
sudo tail -f /opt/domibus/tomcat/domibus/logs/domibus.log
sudo grep -i error /opt/domibus/tomcat/domibus/logs/domibus.log | tail -50

# Base (en domi, via sudo -u domibus)
sudo -u domibus mysql --defaults-file=/opt/domibus/.my.cnf domibus_schema -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='domibus_schema';"
```

### Problèmes connus et solutions

| Symptôme | Cause | Solution |
|----------|-------|----------|
| `ERROR 1819 password policy` | mot de passe sans caractère spécial | utiliser le format `Db...-Aa9` (étape 5.1) |
| `Illegal mix of collations` | schéma pas en utf8mb4_bin | recréer la base avec la bonne collation (étape 5.2) |
| `SUPER privilege / binary logging` | `log_bin_trust_function_creators` désactivé | étape 4.3 |
| `ClassNotFoundException oracle.jdbc` | driver MySQL absent ou properties Oracle actives | étapes 7 + 8 |
| `TB_USER` vide après scripts SQL | **NORMAL** — admin créé au 1er démarrage | ne rien faire, voir étape 10/11 |
| `domibus is not in the sudoers file` | sudo en session domibus | sortir avec `exit`, repasser en `domi` |
| `JAVA_HOME ... not defined correctly` | chemin codé en dur sans numéro de version | utiliser `readlink` (étapes 9/13) |
| Login admin échoue | hash vide (htpasswd absent) | installer apache2-utils, régénérer (étape 11) |
| 404 sur `/domibus/` | WAR pas déployée (datasource KO) | vérifier logs + properties MySQL |

---

## 17. Annexe C — Sécurité

- **Ne jamais coller un mot de passe en clair dans le terminal.** Purger si besoin : `truncate -s 0 ~/.bash_history`.
- `domibus.properties` et `.my.cnf` en `chmod 640`/`600`.
- En **production** : réactiver `domibus.certificate.revocation.check.strategies`, restreindre MySQL, reverse proxy HTTPS en DMZ, rotation des certificats.

### Firewall (UFW)

```bash
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow from <IP_ADMIN> to any port 8080 proto tcp comment 'Console Domibus'
sudo ufw enable
```

---

## 18. Annexe D — Prochaines étapes H2H BOA

1. **CSR & certificats** — générer les CSR (CN `BOA:H2H:{RACINE_CLIENT}`), transmettre à BOA.
2. **Keystore** — importer le `.p12` client signé (*Certificates → Keystore*).
3. **Truststore** — importer le certificat BOA `BOAH2H000001.pem` (alias `BOA_H2H_SANDBOX`) + le client.
4. **PMode** — adapter `PMODE_SAMPLE.xml` (PartyId, raison sociale, endpoint BOA), importer via *PMode → Current*.
5. **Plugin User** — créer l'utilisateur technique ERP (*Plugin Users*).
6. **Plugin FS ou WS** — FS recommandé pour démarrer.
7. **Réseau** — flux sortant 443 vers BOA, whitelisting IP.
8. **Test E2E** — premier échange AS4, vérifier le `Receipt ebMS`.

---

**Fin du guide V3 OPTIMISÉ — Domibus 5.2 / Ubuntu 24.04**
*Intègre les 5 corrections terrain : admin créé au démarrage · pas de sudo en domibus · ajout idempotent · JAVA_HOME auto · htpasswd préinstallé.*
*Procédure validée de bout en bout.*
