# TD 4 ORACLE – Gestion des transactions et verrous sur Oracle

## Exercice 1 Mettre en œuvre les différentes anomalies
(Utiliser les tables **PILOTE**, **AVION** et **VOL** contenues dans le script `airbase.sql`, voir le dossier du cours)

- **Mises à jour perdues** (*UPDATE LOSS*)  
- **Lectures impropres** (*DIRTY READ*)  
- **Lectures non reproductibles** (*INCONSISTENCY READ*)  
- **Lignes fantômes** (*GHOST UPDATE*)  

## Exercice 2 – Proposer des solutions aux anomalies constatées

Pour chaque anomalie constatée, proposez une solution en s’appuyant sur :  
- L’acquisition préalable des verrous pour chaque ressource  
- La modification du niveau d’isolation :  

```sql
SET TRANSACTION ISOLATION LEVEL [ ... ];
```

## Exercice 3 – Deadlocks

1. Mettre en œuvre un **deadlock** (organiser deux transactions avec les tables **PILOTE**, **AVION** et **VOL**).
2. Proposer **deux solutions différentes** pour éviter le problème de deadlock.


## Exercice 4 – Techniques de verrouillage

* Mettre en œuvre les techniques de verrouillage concurrentes avec Oracle / exécutez des requêtes provoquant un verrouillage
```sql
LOCK TABLE vol IN [...] MODE [NOWAIT]; 
```
* Visualiser les informations sur les verrous, leurs auteurs, l'état des transaction en utilisant les ressource suivantes, à chaque fois donnez votre résultat et l'interpretation que vous en avez

### Tables dynamiques utiles :

* `v$transaction`
* `v$lock`
* `dba_lock`
* `dba_dml_lock`
* `dba_ddl_lock`
* `V$LOCKED_OBJECT`
* `sys.obj$` (colonnes `obj#` et `name`)
  
### Requêtes utiles :
```sql
SELECT SUBSTR(TO_CHAR(session_id),1,5) "SID",
       SUBSTR(lock_type,1,15) "Lock Type",
       SUBSTR(mode_held,1,15) "Mode Held",
       SUBSTR(blocking_others,1,15) "Blocking?"
FROM dba_locks;
```
```sql
select * from dba_dml_locks;
```
```sql
SELECT * FROM dba_blockers;
```
```sql
SELECT * FROM dba_waiters;
```
```sql
SELECT
    -- Blocking Session Details
    l1.sid AS blocking_sid,
    s1.serial# AS blocking_serial,
    s1.username AS blocking_user,
    s1.osuser AS blocking_os_user,
    -- Waiting Session Details
    l2.sid AS waiting_sid,
    s2.serial# AS waiting_serial,
    s2.username AS waiting_user,
    s2.osuser AS waiting_os_user,
    -- Locked Object Details
    o.object_name AS locked_object,
    l1.type AS lock_type
FROM
    v$lock l1,
    v$lock l2,
    v$session s1,
    v$session s2,
    dba_objects o,
    v$locked_object lo
WHERE
    l1.block = 1
    AND l2.request > 0
    AND l1.id1 = l2.id1
    AND l1.id2 = l2.id2
    AND l1.sid = s1.sid
    AND l2.sid = s2.sid
    AND l1.sid = lo.session_id
    AND lo.object_id = o.object_id;
```

## Rendu
Effectuez les opérations demandées et commentez les opérations / résultats obtenus pour chaque étape, tracez tout dans un fichier ".sql" ou au format markdown que vous rendrez sur la boite de dépôt précisée sur Moodle.

