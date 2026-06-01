# Guide de génération des CSR — Projet H2H BOA / AMOAMAN

**Projet** : Host-to-Host Bank of Africa — Access Point AS4 (Corner C2 client)
**Client** : AMOAMAN
**Racine client BOA** : `406539`
**Raison sociale** : `AMOAMAN`
**Environnement** : Sandbox BOA (`https://h2h-integration.out.of.africa`)
**Référence** : Guide BOA v1.0.0 — section 7.2

---

> **🎯 Objectif de cette étape**
>
> Générer la **CSR** (Certificate Signing Request) pour la **signature des messages AS4** (WS-Security XML Digital Signature).
>
> Le `.csr` est envoyé à BOA Groupe → BOA signe et renvoie un `.pem` → tu génères un `.p12` → tu importes dans le Keystore Domibus.
>
> **⚠️ La clé privée `.key` ne sort JAMAIS de ton serveur.** Seul le `.csr` est transmis à BOA.

---

## Table des matières

1. [Prérequis et arborescence](#1-prérequis-et-arborescence)
2. [Étape 1 — Fichier de configuration OpenSSL](#2-étape-1--fichier-de-configuration-openssl)
3. [Étape 2 — Génération de la clé privée RSA](#3-étape-2--génération-de-la-clé-privée-rsa)
4. [Étape 3 — Génération de la CSR](#4-étape-3--génération-de-la-csr)
5. [Étape 4 — Vérification de la CSR](#5-étape-4--vérification-de-la-csr)
6. [Étape 5 — Sécurisation et sauvegarde](#6-étape-5--sécurisation-et-sauvegarde)
7. [Étape 6 — Transmission à BOA](#7-étape-6--transmission-à-boa)
8. [Étape 7 — Après réception du .pem signé : génération du .p12](#8-étape-7--après-réception-du-pem-signé--génération-du-p12)
9. [Étape 8 — Import Keystore Domibus](#9-étape-8--import-keystore-domibus)
10. [Étape 9 — Import Truststore Domibus](#10-étape-9--import-truststore-domibus)
11. [Annexe A — Récapitulatif des fichiers](#11-annexe-a--récapitulatif-des-fichiers)
12. [Annexe B — Checklist e-mail à BOA](#12-annexe-b--checklist-e-mail-à-boa)
13. [Annexe C — Dépannage](#13-annexe-c--dépannage)

---

## 1. Prérequis et arborescence

### 1.1 Outils requis

Sur la VM Domibus (déjà installés normalement) :

```bash
which openssl   # /usr/bin/openssl
which keytool   # /usr/lib/jvm/jdk-21.0.11-oracle-x64/bin/keytool
openssl version # OpenSSL 3.x.x
```

> ✅ Si l'une de ces commandes manque : `sudo apt install -y openssl` (et le JDK 21 est déjà là grâce à l'étape 2 du guide d'installation).

### 1.2 Arborescence dédiée certificats

> 💡 **Bonne pratique de sécurité** : on isole tout dans un dossier dédié, avec des droits stricts. Tu n'auras à protéger qu'un seul dossier.

En 🟦 `domi` (sudo) :

```bash
# Création du dossier dédié certificats
sudo mkdir -p /opt/domibus/certs/{csr,keys,pem,p12,backup}
sudo chown -R domibus:domibus /opt/domibus/certs
sudo chmod 700 /opt/domibus/certs
sudo chmod 700 /opt/domibus/certs/keys     # clés privées : accès strict
sudo chmod 700 /opt/domibus/certs/p12      # .p12 = clés privées : accès strict
sudo chmod 755 /opt/domibus/certs/csr      # .csr peuvent être lus
sudo chmod 755 /opt/domibus/certs/pem      # .pem (certificats publics)

ls -la /opt/domibus/certs/
```

| Dossier | Contenu | Droits |
|---------|---------|--------|
| `keys/` | **Clés privées** `.key` (NE SORT JAMAIS) | 700 |
| `csr/`  | CSR à envoyer à BOA | 755 |
| `pem/`  | Certificats publics (BOA + ton .pem signé) | 755 |
| `p12/`  | **Bundle .p12 = clé privée + cert** | 700 |
| `backup/` | Sauvegardes chiffrées | 700 |

### 1.3 Bascule en utilisateur `domibus`

Toutes les opérations de génération se font en `domibus` (les fichiers appartiennent à l'application qui les consommera).

```bash
sudo su - domibus
cd /opt/domibus/certs
```

> 🟩 À partir d'ici, **tu es en `domibus`** (sans sudo).

---

## 2. Étape 1 — Fichier de configuration OpenSSL

> 💡 On utilise un fichier `.cnf` plutôt que les options en ligne pour garantir le **DN exact** demandé par BOA, sans risque d'erreur de frappe ou d'échappement shell.

### 2.1 Création du fichier `amoaman-h2h.cnf`

En 🟩 `domibus`, dans `/opt/domibus/certs/` :

```bash
cat > /opt/domibus/certs/amoaman-h2h.cnf <<'EOF'
# =====================================================================
# Configuration OpenSSL — CSR Signature AS4
# Projet  : H2H Bank of Africa
# Client  : AMOAMAN
# Racine  : 406539
# =====================================================================
[req]
default_md         = sha256
prompt             = no
distinguished_name = dn
string_mask        = utf8only

[dn]
CN = BOA:H2H:406539
O  = AMOAMAN
C  = BF
EOF

# Vérification
cat /opt/domibus/certs/amoaman-h2h.cnf
```

### 2.2 Détail des champs du DN

| Champ | Valeur | Origine | ⚠️ |
|-------|--------|---------|-----|
| **CN** (Common Name) | `BOA:H2H:406539` | Imposé par BOA (`BOA:H2H:{RACINE_CLIENT}`) | **Format exact obligatoire** — les `:` et les chiffres doivent matcher |
| **O** (Organization) | `AMOAMAN` | Raison sociale validée par BOA | Tel quel, sans accent, sans guillemet |
| **C** (Country) | `BF` | Code ISO Burkina Faso (corner C2 client) | 2 lettres majuscules |

> 🛑 **Ne modifie aucun de ces champs sans validation BOA.** Le PartyId et l'alias Keystore final seront dérivés de `CN`/`O`. Une erreur ici = CSR rejetée par BOA = nouveau round-trip.

> 💡 **Pas de SAN (Subject Alternative Name)** : c'est volontaire. Ce certificat sert à signer des messages AS4 (WS-Security), **pas** à sécuriser un endpoint TLS. Le SAN serait inutile ici. Le certificat TLS du reverse proxy (pour `https://domibus-test.amoaman.com`) sera un **certificat distinct** issu d'une CA publique (Let's Encrypt, etc.).

---

## 3. Étape 2 — Génération de la clé privée RSA

> 🔐 **Cette clé est le secret le plus critique du projet.** Si elle est compromise, l'attaquant peut signer des messages AS4 au nom d'AMOAMAN. Elle ne doit **jamais** sortir du serveur, jamais être envoyée par mail, jamais commitée dans Git.

### 3.1 Génération

En 🟩 `domibus`, dans `/opt/domibus/certs/` :

```bash
cd /opt/domibus/certs

openssl genpkey \
  -algorithm RSA \
  -pkeyopt rsa_keygen_bits:2048 \
  -out keys/amoaman-h2h-dsig.key

# Droits stricts immédiatement
chmod 600 keys/amoaman-h2h-dsig.key
ls -la keys/amoaman-h2h-dsig.key
```

> 💡 **Différence avec le PDF BOA** : le PDF contient une coquille `rsa_keygen_bites:2048` (avec `bites`). La bonne option OpenSSL est `rsa_keygen_bits:2048` (avec `bits`). C'est corrigé ci-dessus.

### 3.2 Vérification

```bash
# La clé doit faire ~1700 octets et commencer par "-----BEGIN PRIVATE KEY-----"
head -1 keys/amoaman-h2h-dsig.key
wc -c keys/amoaman-h2h-dsig.key

# Inspection détaillée (taille = 2048 bits)
openssl pkey -in keys/amoaman-h2h-dsig.key -text -noout | head -5
```

> ✅ **Contrôle** :
> - Premier ligne = `-----BEGIN PRIVATE KEY-----`
> - `openssl pkey ... -text` affiche `Private-Key: (2048 bit, 2 primes)`
> - Droits `-rw-------` (600) appartenant à `domibus:domibus`

---

## 4. Étape 3 — Génération de la CSR

### 4.1 Génération

```bash
cd /opt/domibus/certs

openssl req \
  -new \
  -key keys/amoaman-h2h-dsig.key \
  -config amoaman-h2h.cnf \
  -out csr/amoaman-h2h-dsig.csr

# Droits
chmod 644 csr/amoaman-h2h-dsig.csr
ls -la csr/amoaman-h2h-dsig.csr
```

### 4.2 Vérification visuelle

```bash
cat csr/amoaman-h2h-dsig.csr
```

> ✅ Doit afficher un bloc :
> ```
> -----BEGIN CERTIFICATE REQUEST-----
> MIIC...
> ...
> -----END CERTIFICATE REQUEST-----
> ```

---

## 5. Étape 4 — Vérification de la CSR

**Étape critique** : si le DN n'est pas exact, BOA rejettera la CSR.

### 5.1 Inspection détaillée

```bash
openssl req -in /opt/domibus/certs/csr/amoaman-h2h-dsig.csr -noout -text
```

> ✅ **Contrôle BLOQUANT** — vérifier ligne par ligne :
>
> ```
> Subject: CN = BOA:H2H:406539, O = AMOAMAN, C = BF
> Public Key Algorithm: rsaEncryption
>     RSA Public-Key: (2048 bit)
> Signature Algorithm: sha256WithRSAEncryption
> ```
>
> Si l'un de ces éléments est différent (faute de frappe dans le `.cnf`, mauvais algo, etc.) : **NE PAS ENVOYER À BOA**. Reprendre depuis l'étape 2.1 après correction du `.cnf` et regénération `.key` + `.csr`.

### 5.2 Vérification rapide format court

```bash
openssl req -in /opt/domibus/certs/csr/amoaman-h2h-dsig.csr -noout -subject
```

Doit afficher exactement : `subject=CN = BOA:H2H:406539, O = AMOAMAN, C = BF`

### 5.3 Vérification d'intégrité (cohérence clé ↔ CSR)

```bash
# Les deux hash MD5 ci-dessous doivent être IDENTIQUES
openssl req -in /opt/domibus/certs/csr/amoaman-h2h-dsig.csr -noout -pubkey | openssl md5
openssl pkey -in /opt/domibus/certs/keys/amoaman-h2h-dsig.key -pubout | openssl md5
```

> ✅ **Contrôle** : les deux empreintes MD5 sont identiques (= la clé privée correspond bien à la clé publique de la CSR).

---

## 6. Étape 5 — Sécurisation et sauvegarde

### 6.1 Sauvegarde chiffrée de la clé privée

> 🔐 La clé `.key` est le secret le plus précieux. On en fait une copie **chiffrée par mot de passe** stockée dans `backup/`, pour pouvoir restaurer en cas de perte.

```bash
cd /opt/domibus/certs

# Création d'une copie chiffrée AES-256 (un mot de passe sera demandé 2x)
openssl pkcs8 \
  -topk8 \
  -v2 aes-256-cbc \
  -in keys/amoaman-h2h-dsig.key \
  -out backup/amoaman-h2h-dsig.key.enc

chmod 600 backup/amoaman-h2h-dsig.key.enc
ls -la backup/
```

> ⚠️ **Mot de passe à conserver dans votre coffre-fort d'entreprise** (KeePass, Vaultwarden, HashiCorp Vault, etc.). Sans lui, la sauvegarde est inutilisable.

### 6.2 Sauvegarde du `.cnf` et de la `.csr`

```bash
cp /opt/domibus/certs/amoaman-h2h.cnf /opt/domibus/certs/backup/
cp /opt/domibus/certs/csr/amoaman-h2h-dsig.csr /opt/domibus/certs/backup/
ls -la /opt/domibus/certs/backup/
```

### 6.3 Sauvegarde externe (recommandé)

Récupère le dossier `backup/` vers un stockage hors-VM (NAS, coffre, machine d'administration) :

```bash
# Depuis ton poste admin
scp domi@<IP_VM>:/opt/domibus/certs/backup/amoaman-h2h-dsig.key.enc ./
scp domi@<IP_VM>:/opt/domibus/certs/backup/amoaman-h2h-dsig.csr ./
scp domi@<IP_VM>:/opt/domibus/certs/backup/amoaman-h2h.cnf ./
```

---

## 7. Étape 6 — Transmission à BOA

### 7.1 Ce qui doit être envoyé

| Fichier | À transmettre ? | Comment |
|---------|----------------|---------|
| `csr/amoaman-h2h-dsig.csr` | ✅ **OUI** | Pièce jointe e-mail ou portail BOA |
| `amoaman-h2h.cnf` | ❌ NON (interne) | — |
| `keys/amoaman-h2h-dsig.key` | 🛑 **JAMAIS** | Ne JAMAIS partager |
| `backup/amoaman-h2h-dsig.key.enc` | 🛑 **JAMAIS** | Idem (c'est la même clé chiffrée) |

### 7.2 Récupération de la CSR à transmettre

```bash
# Afficher le contenu pour pouvoir le copier-coller dans un e-mail si besoin
cat /opt/domibus/certs/csr/amoaman-h2h-dsig.csr

# Ou télécharger depuis ton poste admin
# scp domi@<IP_VM>:/opt/domibus/certs/csr/amoaman-h2h-dsig.csr ./
```

### 7.3 Empreinte de la CSR (à inclure dans le mail)

> 💡 Communiquer le SHA-256 de la CSR à BOA dans l'e-mail permet à BOA de vérifier qu'aucune altération n'a eu lieu pendant le transport.

```bash
sha256sum /opt/domibus/certs/csr/amoaman-h2h-dsig.csr
```

Note le résultat et inclus-le dans le mail à BOA (voir Annexe B).

---

## 8. Étape 7 — Après réception du `.pem` signé : génération du `.p12`

> Une fois que BOA t'a renvoyé `AMOAMAN-h2h-dsig.pem` (ton certificat signé par leur AC), tu **fusionnes ta clé privée locale** + **ce certificat signé** dans un bundle `.p12` à importer dans Domibus.

### 8.1 Réception et placement du `.pem`

```bash
# Depuis ton poste admin, après réception par e-mail / portail BOA
scp ./AMOAMAN-h2h-dsig.pem domi@<IP_VM>:/tmp/

# Sur la VM, en domi (sudo)
sudo mv /tmp/AMOAMAN-h2h-dsig.pem /opt/domibus/certs/pem/
sudo chown domibus:domibus /opt/domibus/certs/pem/AMOAMAN-h2h-dsig.pem
sudo chmod 644 /opt/domibus/certs/pem/AMOAMAN-h2h-dsig.pem
```

### 8.2 Vérification du `.pem` reçu

En 🟩 `domibus` (`sudo su - domibus`) :

```bash
cd /opt/domibus/certs

# Inspection du certificat
openssl x509 -in pem/AMOAMAN-h2h-dsig.pem -noout -text | head -25
```

> ✅ **Contrôles BLOQUANTS** :
> - `Subject: CN = BOA:H2H:406539, O = AMOAMAN, C = BF` → doit matcher EXACTEMENT le DN de la CSR
> - `Issuer: ...` → AC de BOA (à vérifier avec eux si plusieurs cas possibles)
> - `Not Before` / `Not After` → vérifier la date d'expiration
> - `Public Key Algorithm: rsaEncryption`, `2048 bit`

### 8.3 Cohérence clé privée ↔ certificat reçu

```bash
# Les trois empreintes MD5 ci-dessous doivent être IDENTIQUES
openssl x509 -in pem/AMOAMAN-h2h-dsig.pem -noout -pubkey | openssl md5
openssl pkey -in keys/amoaman-h2h-dsig.key -pubout | openssl md5
openssl req  -in csr/amoaman-h2h-dsig.csr -noout -pubkey | openssl md5
```

> ✅ Les trois MD5 doivent être strictement identiques. Si l'un diffère, **STOP** : le `.pem` reçu ne correspond pas à ta clé privée. Recontacter BOA.

### 8.4 Génération du `.p12`

> 💡 Le `.p12` (PKCS#12) regroupe **clé privée + certificat signé** dans un seul fichier protégé par mot de passe. C'est ce que Domibus attend pour son Keystore.

```bash
cd /opt/domibus/certs

# Génération (mot de passe demandé 2x — choisis un mot de passe FORT, à stocker dans le coffre)
openssl pkcs12 \
  -export \
  -name "AMOAMAN" \
  -inkey keys/amoaman-h2h-dsig.key \
  -in pem/AMOAMAN-h2h-dsig.pem \
  -out p12/AMOAMAN.p12

chmod 600 p12/AMOAMAN.p12
ls -la p12/AMOAMAN.p12
```

> 💡 **Naming convention** : l'option `-name "AMOAMAN"` définit l'**alias** dans le keystore (correspond au `Party name` à utiliser dans le PMode plus tard). Match obligatoire avec ta raison sociale.

### 8.5 Vérification du `.p12`

```bash
# Liste le contenu (mot de passe demandé)
keytool -list -v -keystore /opt/domibus/certs/p12/AMOAMAN.p12 -storetype PKCS12 | head -30
```

> ✅ **Contrôle** :
> - `Alias name: amoaman` (ou `AMOAMAN`, selon casse du `-name`)
> - `Entry type: PrivateKeyEntry`
> - `Owner: CN=BOA:H2H:406539, O=AMOAMAN, C=BF`
> - `Certificate chain length: 1` (ou plus, si BOA inclut sa chaîne)

### 8.6 Sauvegarde du `.p12`

```bash
cp /opt/domibus/certs/p12/AMOAMAN.p12 /opt/domibus/certs/backup/
chmod 600 /opt/domibus/certs/backup/AMOAMAN.p12
```

---

## 9. Étape 8 — Import Keystore Domibus

### 9.1 Via la console web (méthode recommandée)

1. Se connecter à la console Domibus : `https://domibus-test.amoaman.com/domibus/` (ou `http://<IP>:8080/domibus/` en local)
2. Login : `admin` / `DomibusAmoaman2026`
3. Menu **Certificates → Keystore**
4. Cliquer sur **Upload** (ou équivalent selon version 5.2)
5. Sélectionner le fichier `/opt/domibus/certs/p12/AMOAMAN.p12`
6. **Password** : le mot de passe utilisé à l'étape 8.4
7. Valider

> ✅ **Contrôle** : la table affiche une nouvelle entrée :
> - **Alias** : `amoaman` (ou `AMOAMAN`)
> - **Subject** : `CN=BOA:H2H:406539, O=AMOAMAN, C=BF`
> - **Type** : `PrivateKeyEntry`
> - **Valid until** : la date d'expiration du `.pem`

### 9.2 Alternative : commande keytool (si console indisponible)

> ⚠️ Cette manipulation est plus délicate — préférer la console web. À utiliser uniquement en dépannage.

```bash
# Localiser le keystore Domibus existant
ls -la /opt/domibus/tomcat/domibus/conf/domibus/keystores/

# Sauvegarde avant modification (TOUJOURS)
cp /opt/domibus/tomcat/domibus/conf/domibus/keystores/gateway_keystore.jks \
   /opt/domibus/certs/backup/gateway_keystore.jks.$(date +%Y%m%d-%H%M%S)

# Import (les mots de passe Domibus sont dans domibus.properties)
keytool -importkeystore \
  -srckeystore /opt/domibus/certs/p12/AMOAMAN.p12 \
  -srcstoretype PKCS12 \
  -destkeystore /opt/domibus/tomcat/domibus/conf/domibus/keystores/gateway_keystore.jks \
  -deststoretype JKS \
  -alias AMOAMAN
```

---

## 10. Étape 9 — Import Truststore Domibus

> 📌 Le **Truststore** contient les certificats de **confiance** : ce que ton Domibus accepte de croire venant de l'extérieur. Selon le guide BOA (§7.6), tu importes **deux** certificats :
> 1. Le certificat **BOA** : `BOAH2H000001.pem` (transmis par BOA)
> 2. **Ton propre certificat signé** `AMOAMAN-h2h-dsig.pem` (côté truststore : pour valider tes propres signatures lors d'un loopback / test)

### 10.1 Placer le `.pem` BOA sur la VM

Le fichier `BOAH2H000001.pem` t'est fourni par BOA (avec le mail évoqué).

```bash
# Depuis ton poste admin
scp ./BOAH2H000001.pem domi@<IP_VM>:/tmp/

# Sur la VM, en domi (sudo)
sudo mv /tmp/BOAH2H000001.pem /opt/domibus/certs/pem/
sudo chown domibus:domibus /opt/domibus/certs/pem/BOAH2H000001.pem
sudo chmod 644 /opt/domibus/certs/pem/BOAH2H000001.pem
```

### 10.2 Vérification du `.pem` BOA

En 🟩 `domibus` :

```bash
openssl x509 -in /opt/domibus/certs/pem/BOAH2H000001.pem -noout -subject -issuer -dates -fingerprint -sha256
```

> ✅ **Contrôle** :
> - `subject=...` → un identifiant BOA H2H (Sandbox)
> - `notAfter=...` → date d'expiration, à surveiller
> - L'empreinte SHA-256 (fingerprint) : à **vérifier** par téléphone/canal séparé avec BOA pour t'assurer que le fichier n'a pas été altéré en transit (man-in-the-middle).

### 10.3 Import des deux certificats dans le Truststore Domibus

Via la console web :

1. Menu **Certificates → Truststore**
2. Bouton **Upload Certificate**

**Premier import — BOA :**
- Fichier : `/opt/domibus/certs/pem/BOAH2H000001.pem`
- **Alias** : `BOA_H2H_SANDBOX` ← exactement, casse comprise (référence guide BOA §7.6)

**Deuxième import — AMOAMAN (ton propre certificat signé) :**
- Fichier : `/opt/domibus/certs/pem/AMOAMAN-h2h-dsig.pem`
- **Alias** : `AMOAMAN` ← exactement

> ✅ **Contrôle final** : la table Truststore affiche les **deux entrées** avec les bons alias :
> | Alias | Subject |
> |-------|---------|
> | `BOA_H2H_SANDBOX` | `CN=...BOA...` |
> | `AMOAMAN` | `CN=BOA:H2H:406539, O=AMOAMAN, C=BF` |

---

## 11. Annexe A — Récapitulatif des fichiers

### 11.1 Fichiers générés et leur rôle

| Fichier | Emplacement | Rôle | Sensibilité | À envoyer ? |
|---------|------------|------|-------------|-------------|
| `amoaman-h2h.cnf` | `/opt/domibus/certs/` | Config OpenSSL (DN) | Faible | Non |
| `amoaman-h2h-dsig.key` | `keys/` | **Clé privée RSA 2048** | 🔴 CRITIQUE | **JAMAIS** |
| `amoaman-h2h-dsig.csr` | `csr/` | Demande de signature | Moyenne | ✅ À BOA |
| `AMOAMAN-h2h-dsig.pem` | `pem/` | Certificat signé par BOA | Public | Reçu de BOA |
| `BOAH2H000001.pem` | `pem/` | Certificat BOA | Public | Reçu de BOA |
| `AMOAMAN.p12` | `p12/` | Bundle clé+cert | 🔴 CRITIQUE | Non |
| `amoaman-h2h-dsig.key.enc` | `backup/` | Sauvegarde chiffrée clé | Moyenne (avec mdp) | Non |

### 11.2 Mots de passe à conserver dans le coffre

| Secret | Usage | Quand demandé |
|--------|-------|---------------|
| Mot de passe de la sauvegarde chiffrée `.key.enc` | Restauration en cas de perte | Très rarement |
| Mot de passe du `.p12` `AMOAMAN.p12` | Import dans Keystore Domibus | À chaque import / migration |

> 💡 **Recommandation** : utiliser un gestionnaire de secrets d'entreprise (HashiCorp Vault, Bitwarden Business, etc.). Ne jamais stocker en clair dans un mail, un wiki, un Git, ou un fichier sur la VM.

### 11.3 Arborescence finale après import

```
/opt/domibus/certs/                       (700, domibus:domibus)
├── amoaman-h2h.cnf                       (644)
├── keys/                                 (700)
│   └── amoaman-h2h-dsig.key              (600) 🔴
├── csr/                                  (755)
│   └── amoaman-h2h-dsig.csr              (644)
├── pem/                                  (755)
│   ├── AMOAMAN-h2h-dsig.pem              (644) ← reçu de BOA
│   └── BOAH2H000001.pem                  (644) ← reçu de BOA
├── p12/                                  (700)
│   └── AMOAMAN.p12                       (600) 🔴
└── backup/                               (700)
    ├── amoaman-h2h-dsig.key.enc          (600)
    ├── amoaman-h2h-dsig.csr              (644)
    ├── amoaman-h2h.cnf                   (644)
    └── AMOAMAN.p12                       (600) 🔴
```

---

## 12. Annexe B — Checklist e-mail à BOA

### Objet suggéré
```
[H2H BOA - AMOAMAN] Transmission CSR signature AS4 - Racine 406539
```

### Corps suggéré

```
Bonjour [Équipe BOA Groupe],

Suite à vos instructions et conformément au guide d'intégration H2H BOA v1.0.0,
veuillez trouver ci-joint la Certificate Signing Request (CSR) pour la
signature des messages AS4 du projet AMOAMAN.

Paramètres :
  - Racine client     : 406539
  - Raison sociale    : AMOAMAN
  - Pays              : BF (Burkina Faso)
  - Subject DN        : CN=BOA:H2H:406539, O=AMOAMAN, C=BF
  - Algorithme        : RSA 2048 bits
  - Signature digest  : SHA-256
  - Environnement     : Sandbox

Empreinte SHA-256 du fichier transmis (à recouper avant signature) :
  [coller ici le résultat de la commande `sha256sum amoaman-h2h-dsig.csr`]

Pièce jointe :
  - amoaman-h2h-dsig.csr

Merci de bien vouloir signer cette CSR et nous retourner le certificat .pem
correspondant, ainsi que les éventuels certificats intermédiaires de votre
chaîne de confiance le cas échéant.

Pour mémoire, voici les actions encore en attente côté AMOAMAN après
réception de votre .pem :
  - Génération du .p12 et import dans le Keystore Domibus
  - Import du certificat BOAH2H000001.pem dans le Truststore (alias BOA_H2H_SANDBOX)
  - Paramétrage du PMode avec PartyId = 406539, endpoint BOA = h2h-integration.out.of.africa
  - Test de connectivité 443/TCP sortant vers 40.85.95.59 (en cours / fait)

Restant à votre disposition,
Cordialement,
[Signature]
```

### Pré-envoi : vérifications obligatoires

- [ ] La CSR ouvre bien avec `openssl req -in ... -text` et le Subject est `CN = BOA:H2H:406539, O = AMOAMAN, C = BF`
- [ ] La taille de clé est `2048 bit` et l'algo `sha256WithRSAEncryption`
- [ ] L'empreinte SHA-256 calculée est correctement reportée dans le mail
- [ ] **La clé `.key` n'est PAS en pièce jointe** (vérifier 2 fois)
- [ ] **Aucun mot de passe** n'est dans le mail
- [ ] Le mail est adressé au contact technique BOA validé

---

## 13. Annexe C — Dépannage

### 13.1 Problèmes connus

| Symptôme | Cause | Solution |
|----------|-------|----------|
| `openssl req` demande des infos interactivement | Option `prompt = no` mal positionnée ou typo dans `.cnf` | Re-vérifier `amoaman-h2h.cnf`, section `[req]` |
| `Error opening Private Key amoaman-h2h-dsig.key` | Mauvais chemin ou droits | `ls -la keys/` ; doit appartenir à `domibus`, droits 600 |
| `unable to load config info ... config = amoaman-h2h.cnf` | Mauvais chemin du `.cnf` | Toujours utiliser le chemin absolu : `/opt/domibus/certs/amoaman-h2h.cnf` |
| CSR contient `C = ??` ou `O =` vide | Caractères spéciaux dans `.cnf` (BOM, CRLF Windows) | Recréer le `.cnf` via `cat > ... <<'EOF'` sur la VM, pas via copier-coller |
| Les MD5 clé/CSR diffèrent | CSR générée avec une autre clé | Regénérer la CSR avec la **bonne** clé via étape 3 |
| `keytool: Keystore was tampered with` | Mauvais mot de passe `.p12` | Vérifier dans le coffre, refaire l'export `.p12` si perdu |
| BOA rejette la CSR pour DN invalide | Probable faute de frappe sur racine | Re-vérifier `406539` dans le `.cnf`, regénérer |
| `.pem` reçu de BOA ne matche pas la clé | Possible mix-up côté BOA | Recontacter BOA avec les empreintes MD5 |

### 13.2 Recommencer entièrement (en cas d'erreur grave)

```bash
# En domibus
cd /opt/domibus/certs

# Sauvegarder l'existant douteux (au cas où)
mkdir -p backup/$(date +%Y%m%d-%H%M%S)
mv keys/amoaman-h2h-dsig.key   backup/$(date +%Y%m%d-%H%M%S)/ 2>/dev/null
mv csr/amoaman-h2h-dsig.csr    backup/$(date +%Y%m%d-%H%M%S)/ 2>/dev/null

# Repartir de l'étape 2.1 (création du .cnf)
```

### 13.3 Vérifier l'horloge système

Si l'horloge de la VM est décalée, la date `Not Before` du `.pem` peut tomber dans le futur → signature rejetée par BOA / par ta propre validation.

```bash
date
timedatectl
# Doit être à l'UTC réelle (±10s)
```

Si décalage : `sudo timedatectl set-ntp true ; sudo systemctl restart systemd-timesyncd`

---

## ✅ Prochaines actions (rappel mail BOA)

1. ✅ **Génération CSR** — *présent guide, à exécuter*
2. ⏳ **Transmission CSR à BOA Groupe** — *Action AMOAMAN*
3. ⏳ **Réception .pem signé** — *Action BOA Groupe*
4. ⏳ **Génération .p12 + import Keystore + Truststore** — *Action AMOAMAN, §8-10 ci-dessus*
5. ⏳ **Paramétrage PMode** — *à faire ensuite (sur la base de `PMODE_SAMPLE.xml`)*
6. ⏳ **Exposer Domibus en HTTPS** — `https://domibus-test.amoaman.com` (cert TLS public distinct, type Let's Encrypt)
7. ⏳ **Ouverture flux sortant 443/TCP vers 40.85.95.59** — *Action AMOAMAN (firewall/infra)*
8. ⏳ **Atelier de tests E2E avec BOA**

---

**Fin du guide CSR — Projet H2H BOA / AMOAMAN — Racine 406539**
*À utiliser avec le guide d'installation Domibus 5.2 OPTIMISÉ V3 et le PDF BOA v1.0.0.*
