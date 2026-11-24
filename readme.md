# 🔍 Nessus : documentation d'utilisation

> Guide pratique pour installer, configurer et utiliser **Nessus** afin de détecter les vulnérabilités d'un système, dans un **laboratoire isolé**.

⚠️ **Avertissement légal** : ne scannez que des machines qui vous appartiennent ou pour lesquelles vous disposez d'une **autorisation écrite**. Un scan non autorisé est illégal (en France : articles 323-1 et suivants du Code pénal).

---

## Sommaire

1. [Qu'est-ce que Nessus ?](#1-quest-ce-que-nessus-)
2. [Les éditions de Nessus](#2-les-éditions-de-nessus)
3. [Comment fonctionne un scan ?](#3-comment-fonctionne-un-scan-)
4. [Architecture du laboratoire](#4-architecture-du-laboratoire)
5. [Installation sur Debian](#5-installation-sur-debian)
6. [Créer et lancer un scan](#6-créer-et-lancer-un-scan)
7. [Lire et analyser les résultats](#7-lire-et-analyser-les-résultats)
8. [Exporter un rapport](#8-exporter-un-rapport)
9. [Bonnes pratiques](#9-bonnes-pratiques)
10. [Dépannage](#10-dépannage)
11. [Ressources](#11-ressources)

---

## 1. Qu'est-ce que Nessus ?

**Nessus** est un **scanner de vulnérabilités** édité par **Tenable**. Il interroge les machines d'un réseau (ports ouverts, services, versions, configuration) et compare ce qu'il observe à une base de **plugins** (des tests de sécurité, chacun identifié par un *Plugin ID*) pour détecter :

- des **vulnérabilités connues** (identifiées par leur numéro **CVE**) ;
- des **erreurs de configuration** (mots de passe par défaut, services inutiles exposés, chiffrement faible) ;
- des **logiciels obsolètes ou non patchés** ;
- des **écarts de conformité** (audits de configuration).

> 💡 **À ne pas confondre** : *Nessus* est le scanner que l'on installe soi-même. *Tenable Vulnerability Management* (ex-Tenable.io) est la plateforme cloud de Tenable, qui centralise des scans.

Un scanner **détecte** mais n'**exploite pas** : il signale qu'une faille est probablement présente. C'est ensuite à l'analyste de confirmer (par exemple avec Metasploit, dans un cadre autorisé).

## 2. Les éditions de Nessus

| Édition | Public visé | Particularités |
| --- | --- | --- |
| **Nessus Essentials** | Étudiants, apprentissage, labo perso | Gratuite, **limitée à 16 adresses IP** par scanner |
| **Nessus Professional** | Consultants, pentesters, PME | Licence payante, nombre d'IP illimité, plus de modèles de scan |
| **Nessus Expert** | Équipes sécurité | Ajoute notamment le scan de surface d'attaque externe et de l'infrastructure cloud |

Pour un laboratoire de cours, **Nessus Essentials** suffit largement.

## 3. Comment fonctionne un scan ?

```
flowchart LR
    A[Découverte des hôtes] --> B[Scan des ports]
    B --> C[Identification des services et versions]
    C --> D[Exécution des plugins]
    D --> E[Corrélation avec les CVE]
    E --> F[Rapport + recommandations]
```

Il existe deux grands modes :

| Mode | Principe | Avantage | Limite |
| --- | --- | --- | --- |
| **Non authentifié** (« de l'extérieur ») | Nessus teste uniquement ce qui est visible depuis le réseau | Simule un attaquant sans accès | Moins précis, plus de faux positifs/négatifs |
| **Authentifié** (« avec identifiants ») | Nessus se connecte (SSH, SMB/Windows…) et inspecte le système de l'intérieur | Détecte les paquets non patchés, très fiable | Nécessite des comptes dédiés |

## 4. Architecture du laboratoire

```
flowchart LR
    subgraph RESEAU["Réseau isolé (host-only / réseau interne)"]
        N["VM Debian<br/>Nessus (port 8834)"]
        M["VM Metasploitable 2<br/>cible volontairement vulnérable"]
    end
    N -- "scan" --> M
    H["Hôte : navigateur web"] -- "https://IP-Nessus:8834" --> N
```

**Points essentiels :**

- Les deux VM doivent être sur le **même réseau virtuel** et pouvoir se joindre (test avec `ping`).
- Utilisez un réseau **host-only** ou **interne** (VirtualBox, VMware…). ❌ **Ne mettez jamais Metasploitable 2 en mode « bridge » ou accessible depuis Internet** : elle est conçue pour être piratée.
- Ressources conseillées pour la VM Nessus : **2 vCPU, 4 Go de RAM, 30 Go de disque**.

## 5. Installation sur Debian

### 5.1 Télécharger le paquet

Depuis le site officiel de Tenable (page *Downloads → Nessus*), choisir :

- **Version** : la dernière disponible ;
- **Plateforme** : *Linux – Debian – amd64* (fichier `.deb`).

### 5.2 Installer

```
# Installation du paquet (adapter le nom du fichier téléchargé)
sudo dpkg -i Nessus-<version>-debian10_amd64.deb

# Démarrer le service et l'activer au démarrage
sudo systemctl start nessusd
sudo systemctl enable nessusd

# Vérifier que le service tourne
sudo systemctl status nessusd
```

### 5.3 Configuration initiale (interface web)

1. Ouvrir `https://<IP-de-la-VM-Debian>:8834` dans un navigateur.
2. Accepter l'avertissement du certificat auto-signé (normal en local).
3. Choisir **Nessus Essentials** puis demander un **code d'activation** gratuit sur le site de Tenable (il est envoyé par e-mail).
4. Saisir le code, puis **créer le compte administrateur** (nom d'utilisateur + mot de passe).
5. Patienter pendant le **téléchargement et la compilation des plugins** : cela peut durer plusieurs minutes.

> 💡 Pour connaître l'IP de la VM : `ip a`.

### 5.4 Mettre à jour les plugins

Les plugins évoluent chaque jour. Pour forcer une mise à jour en ligne de commande :

```
sudo /opt/nessus/sbin/nessuscli update --all
sudo systemctl restart nessusd
```

## 6. Créer et lancer un scan

### 6.1 Choisir un modèle

Menu **Scans → New Scan**. Les modèles les plus utiles :

| Modèle | Usage |
| --- | --- |
| **Host Discovery** | Repérer les machines actives sur un réseau |
| **Basic Network Scan** | Scan général d'une machine ou d'un réseau (idéal pour débuter) |
| **Advanced Scan** | Même chose, avec tous les réglages détaillés |
| **Credentialed Patch Audit** | Scan authentifié des correctifs manquants |
| **Web Application Tests** | Tests sur des applications web |

### 6.2 Configurer le scan *Basic Network Scan*

| Champ | Valeur |
| --- | --- |
| **Name** | `Scan Metasploitable 2` |
| **Targets** | IP de la cible, par exemple `192.168.56.101` |
| **Schedule** (optionnel) | Planifier un scan récurrent |
| **Discovery / Assessment** | Laisser les valeurs par défaut pour un premier essai |
| **Credentials** (optionnel) | Ajouter un compte SSH pour un scan authentifié |

### 6.3 Lancer

Cliquer sur **Save**, puis sur le bouton ▶ **Launch**. Le statut passe à *Running*, puis à *Completed*. La durée dépend de la cible (quelques minutes en labo).

## 7. Lire et analyser les résultats

### 7.1 Les niveaux de sévérité

Nessus classe chaque constat selon son score **CVSS** :

| Sévérité | Score CVSS v3 | Interprétation |
| --- | --- | --- |
| 🔴 **Critical** | 9.0 – 10.0 | À corriger immédiatement |
| 🟠 **High** | 7.0 – 8.9 | À corriger en priorité |
| 🟡 **Medium** | 4.0 – 6.9 | À planifier |
| 🟢 **Low** | 0.1 – 3.9 | Risque faible |
| 🔵 **Info** | n/a | Information (service détecté, version, etc.) |

Nessus affiche aussi le **VPR** (*Vulnerability Priority Rating*), un score propre à Tenable qui tient compte de la menace réelle (exploit public disponible, activité observée…), ce qui aide à prioriser.

### 7.2 Les onglets d'un rapport

- **Hosts** : les machines scannées et leur nombre de vulnérabilités ;
- **Vulnerabilities** : la liste des failles, groupées par plugin ;
- **Remediations** : les actions correctives regroupées (par exemple « mettre à jour tel logiciel » corrige plusieurs failles à la fois) ;
- **History** : l'historique des exécutions, pratique pour comparer avant/après correction.

### 7.3 Anatomie d'une vulnérabilité

En cliquant sur un constat, on obtient notamment :

| Élément | Utilité |
| --- | --- |
| **Description** | Ce que la faille permet |
| **Solution** | Comment la corriger |
| **Plugin ID / CVE** | Identifiants pour rechercher la faille (par exemple dans Metasploit ou la base NVD) |
| **CVSS / VPR** | Gravité et priorité |
| **Plugin Output** | La preuve technique relevée par Nessus |
| **See Also** | Liens vers les avis officiels |

### 7.4 Méthode d'analyse conseillée

1. Trier par sévérité décroissante : traiter d'abord les **Critical** et **High**.
2. Regrouper par **type** : services obsolètes, mots de passe par défaut, services non chiffrés (Telnet, FTP), failles de protocole (SMB…), etc.
3. **Vérifier** les résultats : un scanner produit des faux positifs.
4. Noter pour chaque faille importante : le **CVE**, le **service/port**, l'**impact** et la **solution**.

## 8. Exporter un rapport

Dans le scan terminé : **Report** (ou *Export*), puis choisir le format :

| Format | Usage |
| --- | --- |
| **PDF / HTML** | Rapport lisible, à présenter |
| **CSV** | Exploitation dans un tableur |
| **Nessus (.nessus)** | Format XML natif, réimportable dans Nessus |

## 9. Bonnes pratiques

- ✅ Obtenir une **autorisation écrite** avant tout scan hors labo.
- ✅ **Mettre à jour les plugins** avant chaque scan.
- ✅ Préférer les **scans authentifiés** pour des résultats fiables, avec un compte dédié aux droits minimaux.
- ✅ Scanner **hors des heures de production** : un scan peut charger le réseau, voire perturber des équipements fragiles.
- ✅ **Isoler** le laboratoire du reste du réseau.
- ✅ **Rescanner après correction** pour confirmer que la faille a disparu.
- ✅ Scanner **régulièrement** : de nouvelles failles sont publiées chaque jour.
- ✅ **Protéger les rapports** : ils décrivent précisément les faiblesses de l'infrastructure.

## 10. Dépannage

| Problème | Piste de solution |
| --- | --- |
| Impossible d'ouvrir `https://IP:8834` | Vérifier `systemctl status nessusd`, l'IP de la VM et que le port 8834 n'est pas bloqué |
| Avertissement de certificat | Normal (certificat auto-signé) : accepter l'exception |
| Plugins bloqués en « compiling » | Patienter ; vérifier l'espace disque et la RAM de la VM |
| La cible n'est pas détectée | Tester `ping` entre les deux VM ; vérifier qu'elles sont sur le même réseau virtuel |
| Peu de résultats sur la cible | Vérifier que l'IP ciblée est la bonne ; essayer un scan authentifié |
| Message de limite d'IP | Nessus Essentials est limité à 16 IP |

## 11. Ressources

- [Documentation officielle de Nessus (Tenable)](https://docs.tenable.com/nessus/Content/GetStarted.htm)
- [Téléchargement de Nessus](https://www.tenable.com/downloads/nessus)
- [Base NVD (CVE)](https://nvd.nist.gov/)
- [Metasploitable 2 (documentation Rapid7)](https://docs.rapid7.com/metasploit/metasploitable-2/)

---

*Document réalisé dans le cadre du projet « Découverte et exploitation de vulnérabilités » (La Plateforme). Usage strictement pédagogique, en environnement contrôlé.*
