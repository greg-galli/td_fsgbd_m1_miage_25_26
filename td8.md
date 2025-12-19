# Sujet de TD : Mécanismes d'Indexation et Performance sous Oracle


## Objectifs Pédagogiques

1.  Comprendre la différence structurelle entre un balayage de table séquentiel (**Full Table Scan**) et une recherche par index (**Index Range Scan** / **Unique Scan**).
    
2.  Manipuler des volumes de données significatifs pour observer les limites des systèmes.
    
3.  Mesurer l'impact des **B-Trees** (B+ Trees sous Oracle) sur les opérations de lecture et d'écriture.
    
4.  Analyser des métriques de performance statistiques (Min, Max, Moyenne, Médiane).

## Partie 1 : Préparation de l'environnement

Nous allons créer une table `CLIENTS_TEST` qui ne possède **aucune clé primaire** (pour éviter la création automatique d'un index par Oracle au début).

**1. Création de la table** Exécutez le script suivant pour créer la structure :
```SQL
DROP TABLE CLIENTS_TEST;

CREATE TABLE CLIENTS_TEST (
    ID_CLIENT NUMBER,
    NOM VARCHAR2(100),
    PRENOM VARCHAR2(100),
    EMAIL VARCHAR2(150),
    DATE_INSCRIPTION DATE,
    COMMENTS VARCHAR2(4000) -- Colonne "lourde" pour augmenter la taille des blocs
);
```

## Partie 2 : Génération de données (Injection Massive)

Pour que les tests soient significatifs, nous devons générer un grand volume de données. Nous utiliserons un bloc PL/SQL pour insérer des données aléatoires.

**Consigne :** Vous allez effectuer des tests sur trois paliers de volume. Pour l'instant, nous commençons par le **Palier 1 : 50 000 lignes**.

Exécutez ce bloc PL/SQL (ajustez la variable `v_nb_rows` selon le palier) :

```SQL
DECLARE
    v_nb_rows NUMBER := 50000; -- Changez cette valeur pour les paliers suivants (ex: 100000, 500000)
BEGIN
    FOR i IN 1..v_nb_rows LOOP
        INSERT INTO CLIENTS_TEST (ID_CLIENT, NOM, PRENOM, EMAIL, DATE_INSCRIPTION, COMMENTS)
        VALUES (
            i,
            'Nom_' || DBMS_RANDOM.STRING('U', 5),
            'Prenom_' || DBMS_RANDOM.STRING('U', 5),
            'user_' || i || '@test.com', -- L'email est unique ici
            SYSDATE - DBMS_RANDOM.VALUE(0, 3650),
            RPAD('A', 200, 'A') -- Remplissage pour simuler du volume disque
        );
        
        -- Commit tous les 5000 enregistrements pour ne pas saturer les logs
        IF MOD(i, 5000) = 0 THEN
            COMMIT;
        END IF;
    END LOOP;
    COMMIT;
END;
/
```

## Partie 3 : Protocole de Test et Mesures (Sans Index)

Nous allons chercher un client spécifique par son `EMAIL`. Actuellement, la table n'a pas d'index.

**1. Analyse du Plan d'Exécution** Avant de mesurer le temps, demandez à Oracle comment il compte chercher la donnée.

-   Exécutez : `EXPLAIN PLAN FOR SELECT * FROM CLIENTS_TEST WHERE EMAIL = 'user_45000@test.com';`
    
-   Affichez le plan : `SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);`
    
-   **Question :** Que signifie "TABLE ACCESS FULL" ? Quel est le "Cost" estimé ?
    

**2. Script de mesure de performance** Pour obtenir des statistiques fiables (Min, Max, Moyenne, Médiane), nous ne pouvons pas nous fier à une seule exécution (effets de cache, latence réseau). Utilisez ce script de test automatisé qui exécute la recherche **100 fois** :

```SQL
SET SERVEROUTPUT ON;
DECLARE
    v_start TIMESTAMP;
    v_end TIMESTAMP;
    v_elapsed NUMBER; -- temps en millisecondes
    v_dummy VARCHAR2(150);
    
    v_times SYS.ODCINUMBERLIST := SYS.ODCINUMBERLIST();
    
    v_min NUMBER := 999999;
    v_max NUMBER := 0;
    v_avg NUMBER := 0;
    v_median NUMBER := 0;
    v_total NUMBER := 0;
    v_iterations NUMBER := 100; -- Nombre de répétitions
    
    v_target_email VARCHAR2(100);
    v_count NUMBER; -- Variable ajoutée pour stocker le compte
BEGIN
    -- 1. Récupération du volume total de données
    SELECT COUNT(*) INTO v_count FROM CLIENTS_TEST;

    -- 2. Sélection d'un email cible aléatoire
    SELECT 'user_' || ROUND(DBMS_RANDOM.VALUE(1, v_count)) || '@test.com'
    INTO v_target_email FROM DUAL;

    -- 3. Boucle de test
    FOR i IN 1..v_iterations LOOP
        v_start := SYSTIMESTAMP;
        
        -- La requête à tester
        SELECT EMAIL INTO v_dummy FROM CLIENTS_TEST WHERE EMAIL = v_target_email;
        
        v_end := SYSTIMESTAMP;
        
        -- Conversion en millisecondes
        v_elapsed := EXTRACT(SECOND FROM (v_end - v_start)) * 1000;
        
        v_times.EXTEND;
        v_times(i) := v_elapsed;
        v_total := v_total + v_elapsed;
        
        IF v_elapsed < v_min THEN v_min := v_elapsed; END IF;
        IF v_elapsed > v_max THEN v_max := v_elapsed; END IF;
    END LOOP;
    
    -- Calcul Moyenne
    v_avg := v_total / v_iterations;
    
    -- Calcul Médiane
    SELECT MEDIAN(column_value) INTO v_median FROM TABLE(v_times);
    
    -- Affichage des résultats
    DBMS_OUTPUT.PUT_LINE('--------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE('RÉSULTATS POUR : ' || v_target_email);
    DBMS_OUTPUT.PUT_LINE('Volume données : ' || v_count); 
    DBMS_OUTPUT.PUT_LINE('--------------------------------------------------');
    DBMS_OUTPUT.PUT_LINE('Temps MIN      : ' || ROUND(v_min, 4) || ' ms');
    DBMS_OUTPUT.PUT_LINE('Temps MAX      : ' || ROUND(v_max, 4) || ' ms');
    DBMS_OUTPUT.PUT_LINE('Temps MOYEN    : ' || ROUND(v_avg, 4) || ' ms');
    DBMS_OUTPUT.PUT_LINE('Temps MÉDIAN   : ' || ROUND(v_median, 4) || ' ms');
    DBMS_OUTPUT.PUT_LINE('--------------------------------------------------');
END;
/
```

**Notez vos résultats dans un tableau pour pouvoir les analyser plus tard**


## Partie 4 : Création de l'Index B-Tree

Nous allons maintenant structurer les données sous forme d'arbre pour accélérer la recherche.

**1. Création de l'index**

```SQL
CREATE INDEX IDX_CLIENT_EMAIL ON CLIENTS_TEST(EMAIL);
```

**2. Vérification du Plan d'Exécution** Refaites l'étape du `EXPLAIN PLAN`.

-   **Question :** Voyez-vous apparaître "INDEX RANGE SCAN" ou "INDEX UNIQUE SCAN" ? Comparez le "Cost" avec celui de la Partie 3.
    

**3. Mesure de performance (Avec Index)** Relancez exactement le **même script PL/SQL** qu'à la partie 3 (le script de mesure).

-   Observez la différence drastique sur les temps de réponse.
    
-   Notez les résultats.
    

## Partie 5 : Montée en Charge (Volume Variable)

Pour observer la courbe de complexité (O(N) vs O(log N)), vous devez répéter l'opération en augmentant le volume de données.

1.  Supprimez LA TABLE et RECREEZ LA §§
    
2.  Ajoutez des données pour atteindre **100 000** enregistrements (utilisez le script de la Partie 2).
    
3.  Faites les mesures **SANS** index.
    
4.  Recréez l'index.
    
5.  Faites les mesures **AVEC** index.
    
6.  Répétez toutes les étapes pour **500 000** enregistrements.
    

----------

## Partie 6 : Analyse et Restitution

**A. Visualisation** Tracez deux courbes sur un graphique (Axe X = Nombre d'enregistrements, Axe Y = Temps median de réponse en ms) :

1.  Courbe "Full Scan" (Sans index).
    
2.  Courbe "Index Scan" (Avec index).
    

**B. Questions de réflexion**

1.  **Linéarité :** Comment évolue le temps de recherche sans index quand le volume double ? Et avec l'index ?
    
2.  **Insertion :** Avez-vous remarqué si l'insertion des données (Partie 2) était plus lente quand l'index était présent ? Pourquoi ?
    
3.  **Sélectivité :** Si nous avions créé un index sur la colonne `PRENOM` (où il y a beaucoup de doublons) au lieu de `EMAIL` (unique), l'index serait-il aussi performant ?
    
4.  **Espace Disque :** Un index occupe de la place. Comment vérifier la taille de votre index ? (Indice : `USER_SEGMENTS`).

**C. Questions dans le fil du TD**

Dans le fil du TD, vous trouverez des questions ici et là, répondez y à la fin du document. Prenez bien en compte qu'une question est souvent valable pour les différents scénarios (avec ou sans index, avec 50k, 100k ou 500k enregistrements) et attend généralement 6 réponses pour couvrir tous les cas.

**D. Format du rendu** 

Vous rendrez un fichier PDF qui contiendra toutes vos conclusions et réponses dans la boite de dépôt accessible sur Moodle.

