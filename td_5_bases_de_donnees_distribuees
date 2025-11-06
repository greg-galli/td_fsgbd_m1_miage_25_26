# TD – Bases de Données Distribuées sous Oracle

## Objectifs du TP

Ce TD a pour objectif de découvrir les **principes et mécanismes des bases de données distribuées** sous Oracle, notamment :
- la configuration de liens entre plusieurs bases ;
- l’exécution de transactions distribuées ;
- la gestion des verrous et des commits à deux phases ;
- la compréhension des pannes simulées et du recouvrement distribué.

---

## Préparation avant de commencer

Avant de démarrer, assurez-vous d’avoir réalisé les étapes suivantes :

1. Téléchargez **Oracle Instant Client** pour votre système :  
   [https://www.oracle.com/fr/database/technologies/instant-client/downloads.html](https://www.oracle.com/fr/database/technologies/instant-client/downloads.html)
2. Décompressez et installez l’archive sur votre poste.
3. Copiez le fichier `TNSNAMES.ORA` fourni sur Moodle dans le dossier *instant client*.
4. Le code associé à la lecture complète de cette section et à transmettre à l'enseignant discrètement est "Rhododendron".

---

## Exercice 8.6.1 – Distribution de données sous Oracle

### 8.6.1.1 Problème

Nous disposons de deux bases de données :

- **Base1** : `PDBM1MIA.sub12051533510.vcnmiageuca.oraclevcn.com` (IP : 129.151.234.184, Port : 1521)
- **Base2** : `PDBL3MIA.sub12051533510.vcnmiageuca.oraclevcn.com` (IP : 129.151.234.184, Port : 1521)

Nous voulons permettre aux utilisateurs d’effectuer des **transactions distribuées** entre ces deux bases.

Un utilisateur `&DRUSER` existe sur chacune des bases avec un **mot de passe identique**.

### 8.6.1.2 Travail à faire

1. Dessiner la **topologie** de vos bases de données distribuées.
2. Configurer et tester la **distribution** sur les deux bases existantes.

---

## Activité 1 – Connexion et définition des variables

### 1.1 Connexion sur la Base 1

Ouvrez une **Invite de commandes** et placez-vous dans le dossier contenant `sqlplus.exe` :
```bash
cd D:\FSGBDS\instantclient_19_6
```

Lancez SQL*Plus et définissez les variables suivantes :

```sql
sqlplus /nolog

define REMOTEDB = PDBL3MIA.sub12051533510.vcnmiageuca.oraclevcn.com
define REMOTEDBNAME = PDBL3MIA
define CURRENTDB = PDBM1MIA

define DRUSER = votreLoginOracleIci
define DRUSERPASS = votrePasswordOracleIci

define SCRIPTPATH = c:\SCRIPTS_EXERCICES\Scripts
```

### 1.2 Connexion sur la Base 2

Dans une **seconde fenêtre CMD**, répétez la procédure :

```sql
sqlplus /nolog

define REMOTEDB = PDBM1MIA.sub12051533510.vcnmiageuca.oraclevcn.com
define REMOTEDBNAME = PDBM1MIA
define CURRENTDB = PDBL3MIA

define DRUSER = votreLoginOracleIci
define DRUSERPASS = votrePasswordOracleIci

define SCRIPTPATH = c:\SCRIPTS_EXERCICES\Scripts
```

---

## Activité 2 – Chargement des données

### 2.1 Sur la Base 1

Chargez les tables de démonstration :

```sql
connect &DRUSER@&CURRENTDB/&DRUSERPASS
@&SCRIPTPATH\chap8_demobld.sql
```

### 2.2 Sur la Base 2

Chargez les tables client, produit et commande :

```sql
connect &DRUSER@&CURRENTDB/&DRUSERPASS
@&SCRIPTPATH\chap8_clientbld.sql
```

---

## Activité 3 – Création d’un Database Link

Créez un **database link public** permettant à l’utilisateur `&DRUSER` d’accéder à la base distante.  
> Remarque : dans ce TP, le database link a déjà été créé par l’administrateur et est contenu dans la variable `REMOTEDBNAME`.

---

## Activité 4 – Consultations et mises à jour distantes

Cette activité consiste à interroger et modifier des données sur la base distante.

### 4.1 Consultations distantes

Connectez-vous et observez la structure des tables distantes :

```sql
connect &DRUSER@&CURRENTDB/&DRUSERPASS

desc &DRUSER..produit@&REMOTEDBNAME
desc &DRUSER..commande@&REMOTEDBNAME
desc &DRUSER..client@&REMOTEDBNAME

select * from &DRUSER..commande@&REMOTEDBNAME;
select * from &DRUSER..client@&REMOTEDBNAME;
select * from &DRUSER..produit@&REMOTEDBNAME;
```

### 4.2 Mises à jour distantes

Modifiez des données distantes et observez l’impact sur les transactions et les verrous.

```sql
update &DRUSER..commande@&REMOTEDBNAME set empno = 7369;
commit;
```

Réalisez ensuite une insertion distante :
```sql
insert into &DRUSER..commande@&REMOTEDBNAME 
(pcomm#, cdate, pid#, cid#, pnbre, pprixunit, empno)
values (4, sysdate, 1000, 1, 9, 2, 7369);
commit;
```

---

## Activité 5 – Transactions distribuées

### 5.1 Consultation distribuée (jointure)

Effectuez une jointure entre tables locales et distantes :
```sql
select c.pcomm#, c.pid#, e.empno, c.pprixunit
from &DRUSER..commande@&REMOTEDBNAME c, 
     &DRUSER..produit@&REMOTEDBNAME p, 
     emp e
where c.pid# = p.pid# and c.empno = e.empno;
```

### 5.2 Mise à jour distribuée

Insérez une nouvelle commande et augmentez la commission d’un employé dans la même transaction :
```sql
begin
  insert into &DRUSER..commande@&REMOTEDBNAME 
  (pcomm#, cdate, pid#, cid#, pnbre, pprixunit, empno)
  values (6, sysdate, 1000, 1, 9, 2, 7369);

  update emp set comm = nvl(comm, 0) + 2 where empno = 7369;
end;
/
commit;
```

---

## Activité 6 – Transparence grâce aux synonymes

L’objectif est de masquer la localisation physique des données via les **synonymes**.

```sql
drop synonym commande;
create synonym commande for &DRUSER..commande@&REMOTEDBNAME;

begin
  insert into commande (pcomm#, cdate, pid#, cid#, pnbre, pprixunit, empno)
  values (7, sysdate, 1000, 1, 9, 2, 7369);

  update emp set comm = nvl(comm, 0) + 2 where empno = 7369;
  commit;
end;
/
```

Vérifiez les données :
```sql
select * from commande;
select * from emp where empno = 7369;
```

---

## Activité 7 – Déclencheur (trigger) sur la Base2

Créez un **trigger** qui met à jour la commission de l’employé lorsqu’une commande est insérée ou supprimée.

```sql
connect &DRUSER/&DRUSERPASS@&CURRENTDB

create or replace trigger update_employe_comm
after delete or insert on commande for each row
begin
  if inserting then
    update &DRUSER..emp@&REMOTEDBNAME e
    set e.comm = nvl(e.comm, 0) + 2
    where empno = :new.empno;
  elsif deleting then
    update &DRUSER..emp@&REMOTEDBNAME e
    set e.comm = nvl(e.comm, 0) - 2
    where empno = :old.empno;
  end if;
end;
/
```

---

## Activité 8 – Vérification du trigger

Testez le bon fonctionnement du trigger :

```sql
connect &DRUSER@&CURRENTDB/&DRUSERPASS

insert into commande (pcomm#, cdate, pid#, cid#, pnbre, pprixunit, empno)
values (8, sysdate, 1000, 1, 9, 2, 7369);

select * from commande;
select * from emp where empno = 7369;
```

---

## Activité 9 – Simulation de pannes

Cette partie vise à comprendre le comportement des transactions distribuées en cas de panne simulée.

### 9.1 Liste des types de pannes

Oracle propose la commande suivante :
```sql
commit comment 'ORA-2PC-CRASH-TEST-N';
```
où `N` est un code de 1 à 10 correspondant à différents types de pannes (avant ou après la phase de validation, sur le commit point site, etc.).

### 9.2 Vues utiles

Les vues suivantes permettent d’observer l’état des transactions douteuses :
- `DBA_2PC_PENDING` : transactions distribuées en attente de recouvrement ;
- `DBA_2PC_NEIGHBORS` : connexions impliquées dans des transactions douteuses.

### 9.3 Simulation d’une panne (ex. panne 6)

Exécutez une transaction distribuée et simulez une panne :
```sql
update emp set sal = sal * 1.1 where empno = 7369;
update &DRUSER..produit@&REMOTEDBNAME 
set pdescription = pdescription || 'BravoBravo'
where pid# = 1000;

commit comment 'ORA-2PC-CRASH-TEST-6';
```

Observez ensuite les transactions en attente :
```sql
select * from dba_2pc_pending;
select * from dba_2pc_neighbors;
```

### 9.4 Désactivation / Réactivation du recouvrement distribué (pas réalisable dans le cadre de ce TD)

L’administrateur peut désactiver puis réactiver le recouvrement :
```sql
connect system@AliasRootDB/passwordCompteSystem
alter system disable distributed recovery;

-- (simulation de panne)

alter system enable distributed recovery;
```

---

## Activité 10– Expérimentation avec une autre panne

Reprenez les étapes de l’activité 9 avec un autre code de panne (ex. `ORA-2PC-CRASH-TEST-3`).
Comparez les comportements observés et notez vos conclusions.

---

## Conclusion

À l’issue de ce TD, vous avez :  
- configuré et exploité une architecture de **bases de données distribuées Oracle** ;  
- observé le fonctionnement des **transactions à deux phases** ;  
- expérimenté la **transparence d’accès** grâce aux synonymes ;  
- étudié le **recouvrement en cas de panne** et le rôle des vues système associées.  

Ce travail vous permet de mieux comprendre la robustesse et la complexité des transactions distribuées dans un environnement Oracle professionnel.
