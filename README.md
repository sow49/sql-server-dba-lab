# 🗄️ SQL Server DBA Lab — Portfolio Pratique

> Lab complet de formation DBA SQL Server réalisé en environnement local.  
> **Auteur :** Adama Sow — [@sow49](https://github.com/sow49)  
> **Stack :** SQL Server 2019/2022 · SSIS · SSMA · MySQL · Windows Server

---

## 📋 Table des matières

- [Architecture du lab](#architecture-du-lab)
- [Projets réalisés](#projets-réalisés)
  - [1. ETL SSIS — Import Employés](#1-etl-ssis--import-employés)
  - [2. Migration MySQL → SQL Server avec SSMA](#2-migration-mysql--sql-server-avec-ssma)
  - [3. Infrastructure DBA Lab (VirtualBox)](#3-infrastructure-dba-lab-virtualbox)
- [Environnement technique](#environnement-technique)
- [Compétences démontrées](#compétences-démontrées)
- [Roadmap](#roadmap)

---

## 🏗️ Architecture du lab

```
┌─────────────────────────────────────────────────────┐
│              Machine hôte (Windows 10/11)            │
│                                                      │
│  ┌─────────────────┐    ┌────────────────────────┐  │
│  │  SQL Server      │    │  MySQL Server          │  │
│  │  Instance: DBA1  │◄───│  localhost:3306        │  │
│  │  DB: revision_db │    │  Bases: sakila, world  │  │
│  └─────────────────┘    └────────────────────────┘  │
│           ▲                        ▲                 │
│           │                        │                 │
│    ┌──────────────┐    ┌──────────────────────┐     │
│    │  SSMS + SSIS │    │  SSMA for MySQL      │     │
│    │  (ETL jobs)  │    │  (Migration tool)    │     │
│    └──────────────┘    └──────────────────────┘     │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │  VirtualBox — DBA Lab réseau isolé           │    │
│  │  VM: DC01 (Windows Server 2022)              │    │
│  │  Domaine: LABDBA.local                       │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

---

## 📁 Projets réalisés

### 1. ETL SSIS — Import Employés

**Objectif :** Concevoir un package SSIS complet qui importe des employés depuis un fichier CSV vers SQL Server avec gestion des erreurs.

**Architecture du flux de données :**
```
[Flat File Source CSV]
        │
        ▼
[Data Conversion] — (VARCHAR → NVARCHAR)
        │
        ▼
[Conditional Split]
    ├── Valides (Salaire >= 2000 ET Département non vide)
    │       └──► [OLE DB Destination] → dbo.Staging_Employes
    └── Invalides
            └──► [OLE DB Destination] → dbo.Staging_Employes_Erreurs
```

**Tables SQL Server créées :**

```sql
-- Table principale
CREATE TABLE dbo.Employes (
    ID          INT,
    Nom         NVARCHAR(100),
    Prenom      NVARCHAR(100),
    Email       NVARCHAR(150),
    Salaire     FLOAT,
    Departement NVARCHAR(100)
);

-- Table staging (validés)
CREATE TABLE dbo.Staging_Employes (
    ID          INT,
    Nom         NVARCHAR(100),
    Prenom      NVARCHAR(100),
    Email       NVARCHAR(150),
    Salaire     NVARCHAR(50),
    Departement NVARCHAR(100),
    Statut      NVARCHAR(100)
);

-- Table erreurs
CREATE TABLE dbo.Staging_Employes_Erreurs (
    ID          INT,
    Nom         NVARCHAR(100),
    -- ...
);
```

**Résultat :**
- ✅ 22 lignes valides → `dbo.Staging_Employes`
- ✅ 0 erreurs → `dbo.Staging_Employes_Erreurs`

📄 **[Voir le PDF détaillé](./docs/ETL_Employes_V2_SSIS.pdf)**

---

### 2. Migration MySQL → SQL Server avec SSMA

**Objectif :** Migrer deux bases de données MySQL (sakila et world) vers SQL Server en utilisant SSMA (SQL Server Migration Assistant).

#### 🔧 Outils installés

| Outil | Version |
|-------|---------|
| SSMA for MySQL | 10.6.26238 |
| SSMA Extension Pack | 10.6.26238 |
| MySQL ODBC Driver | 26.10.0 Unicode |
| MySQL Server | 26.10.0 |
| .NET Runtime | 8.x |

#### 📊 Assessment Report — sakila

| Métrique | Résultat |
|----------|----------|
| Objets analysés | 35 |
| Conversion automatique | 21 (60%) ✅ |
| Erreurs (manuel requis) | 14 (40%) ⚠️ |
| Temps manuel estimé | 20.5 heures |

**Principales erreurs identifiées :**

| Code SSMA | Problème | Solution T-SQL |
|-----------|----------|----------------|
| M2SS0103 | GROUP BY non-ANSI | Ajouter toutes les colonnes SELECT dans GROUP BY |
| M2SS0037 | CASCADE multiple sur FK | Remplacer par NO ACTION + trigger |
| M2SS0201 | GROUP_CONCAT() inexistant | Utiliser STRING_AGG() (SQL Server 2017+) |
| M2SS0035 | FULLTEXT index | Recréer avec Full-Text Search SQL Server |

#### 📊 Assessment Report — world

| Métrique | Résultat |
|----------|----------|
| Objets analysés | 3 |
| Erreurs | **0** — 100% automatique ✅ |
| Warnings | 0 |

#### 📈 Résultats de migration

**sakila** (15/16 tables migrées) :

| Table | Rows | Taux |
|-------|------|------|
| film | 1 000 | 100% ✅ |
| film_actor | 5 462 | 100% ✅ |
| payment | 16 044 | 100% ✅ |
| rental | 16 044 | 100% ✅ |
| inventory | 4 581 | 100% ✅ |
| address | 603 | 0% ⚠️ (erreur ODBC) |
| *12 autres tables* | *~5 090* | *100% ✅* |

**world** (3/3 tables migrées) :

| Table | Rows | Taux |
|-------|------|------|
| city | 4 079 | 100% ✅ |
| country | 239 | 100% ✅ |
| countrylanguage | 984 | 100% ✅ |

**Total migré : ~51 967 lignes** 🎉

#### 🔍 Validation SSMS

```sql
-- Vérification post-migration
SELECT COUNT(*) FROM [revision_db].[sakila].[film];       -- 1000
SELECT COUNT(*) FROM [revision_db].[sakila].[rental];     -- 16044
SELECT COUNT(*) FROM [revision_db].[world].[city];        -- 4079
SELECT COUNT(*) FROM [revision_db].[world].[country];     -- 239
```

📄 **[Voir le PDF détaillé](./docs/SSMA_MySQL_to_SQLServer_Migration.pdf)**

---

### 3. Infrastructure DBA Lab (VirtualBox)

**Objectif :** Monter un lab réseau isolé avec Active Directory pour pratiquer les configurations DBA en environnement d'entreprise.

#### Configuration réseau

| Paramètre | Valeur |
|-----------|--------|
| Réseau VirtualBox | NAT Network — labdba-net |
| Plage IP | 192.168.100.0/24 |
| VM DC01 IP | 192.168.100.10 (statique) |
| Domaine AD | LABDBA.local |

#### Promotion DC01 en contrôleur de domaine

```powershell
Install-ADDSForest `
    -DomainName "LABDBA.local" `
    -DomainNetbiosName "LABDBA" `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDns:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "****" -AsPlainText -Force) `
    -Force:$true
```

**Statut :** ⏸️ En pause — reprise prévue avec SSD externe (RAM et stockage insuffisants sur HDD actuel)

---

## ⚙️ Environnement technique

```
Poste de travail : Lenovo (Windows 10/11, 8 GB RAM, HDD)
SQL Server       : DESKTOP-4BM4LUV\DBA1 — Windows Authentication
Base principale  : revision_db
MySQL            : localhost:3306 (root)
SSMS             : SQL Server Management Studio 2019+
SSIS             : SQL Server Integration Services (Visual Studio)
SSMA             : SQL Server Migration Assistant for MySQL 10.6
VirtualBox       : 7.x — VM Windows Server 2022
```

---

## 🎯 Compétences démontrées

### SQL Server Administration
- [x] Installation et configuration d'instances SQL Server nommées
- [x] Création et gestion de bases de données
- [x] Authentification Windows et SQL Server
- [x] Gestion des schemas (dbo, sakila, world)

### ETL & Intégration de données
- [x] Conception de packages SSIS (Data Flow, Control Flow)
- [x] Connexions OLE DB, Flat File Source
- [x] Transformation : Data Conversion, Conditional Split
- [x] Gestion des erreurs et redirection de flux

### Migration de bases de données
- [x] Assessment de compatibilité MySQL → SQL Server
- [x] Conversion de schéma DDL (MySQL → T-SQL)
- [x] Migration de données via ODBC
- [x] Analyse et résolution des erreurs de compatibilité
- [x] Validation post-migration via SSMS

### Infrastructure & Réseau
- [x] Configuration réseau VirtualBox (NAT Network)
- [x] Installation Windows Server 2022
- [x] Configuration IP statique
- [x] Déploiement Active Directory (en cours)

---

## 🗺️ Roadmap

### Court terme
- [ ] Remigrrer `sakila.address` (603 lignes, erreur ODBC à corriger)
- [ ] Corriger les 14 objets sakila en erreur (vues GROUP BY, procédures)
- [ ] Finaliser DC01 → Contrôleur de domaine LABDBA.local

### Moyen terme
- [ ] **Always On Availability Groups** — Configuration WSFC + AG sur lab VirtualBox
- [ ] Migration Oracle → SQL Server avec SSMA (sur SSD externe)
- [ ] Script PowerShell d'automatisation SSMA (mode CLI)
- [ ] Monitoring SQL Server : Agent Jobs, Database Mail, alertes

### Long terme
- [ ] Certification **Microsoft DP-300** (Azure Database Administrator)
- [ ] Lab Azure SQL — déploiement et migration cloud

---

## 📚 Ressources utilisées

- [Documentation SSMA for MySQL — Microsoft](https://docs.microsoft.com/sql/ssma/mysql/)
- [Base sakila — MySQL Sample Databases](https://dev.mysql.com/doc/sakila/en/)
- [Base world — MySQL Sample Databases](https://dev.mysql.com/doc/world-setup/en/)
- [SSIS Documentation — Microsoft](https://docs.microsoft.com/sql/integration-services/)

---

## 📄 Documents du portfolio

| Fichier | Description |
|---------|-------------|
| `docs/ETL_Employes_V2_SSIS.pdf` | Conception et exécution du package SSIS ETL Employés |
| `docs/SSMA_MySQL_to_SQLServer_Migration.pdf` | Migration complète MySQL → SQL Server avec SSMA |

---

*Portfolio DBA SQL Server — Adama Sow | Île-de-France, France | Octobre 2026*
