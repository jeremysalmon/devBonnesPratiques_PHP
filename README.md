# Base de données
  * __$$ escaping__ : dans PostgreSQL, toujours gérer les INSERT et UPDATE en __$$ escaping__ et non avec des quotes. Celà permet de ne pas avoir de problèmes lors de l'insertion de JSON par exemple
  * __UUID__ : la clé primaire de chaque table est idéalement un UUID. Pour que celà fonctionne il faut :
    ```PHP
    CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
    // Dans la création de la table
    id uuid NOT NULL DEFAULT uuid_generate_v1()
    ```
  * __uCreate ...__ : chaque table doit comporter :
    * __dCreate__ (timestamp), __uCreate__ (uuid) : représentent la date et l'utilisateur qui a créé l'enregistrement
    * __dUpdate__ (timestamp), __uUpdate__ (uuid) : réprésentent la date et l'utilisateur de la dernière modification
    * Trigger pour les modifications de dUpdate : 
    ```SQL
    CREATE FUNCTION update_dUpdate() RETURNS TRIGGER AS $$
    BEGIN
    UPDATE TG_TABLE_NAME SET dupdate=now() WHERE id=NEW.id;
    RETURN NEW;
    END; $$
    LANGUAGE plpgsql;
    
    CREATE TRIGGER nom_de_table_trigger_update
    BEFORE INSERT ON nom_de_table
    FOR EACH ROW EXECUTE update_dUpdate;
    ```
  * __customdata__ : toutes les tables contiennent un champs JSONB customdata qui permet de stocker les données non prévues
  * __SLOW QUERIES__ : si il y a besoin de trouver des requêtes lentes
  ```
  Dans postgresql.cong : shared_preload_libraries = 'pg_stat_statements'
  CREATE EXTENSION pg_stat_statements;
  select pg_stat_statements.total_time,pg_stat_statements.rows, pg_stat_statements.query from pg_stat_statements;
  ```
  
# PHP
  * __/libs/__ : le répertoire __libs/__ contient toutes les classes et fichiers de configuration de l'application
  * __.class.php__ : toutes les classes se suffixent __.class.php__ et sont nommées du nom de la table correspondante
  * __.inc.php__ : tous les fichiers qui contenant des define() et des fonctions se suffixent __.inc.php__
  * __1 classe = 1 table__ : la méthode mirroir de facilement se retrouver dans le code
  * __save()__ : une seule méthode a le droit d'écrire dans la base de données. __save()__ s'occupe des INSERT et des UPDATE. Aucune autre méthode ne doit insérer ou mettre à jour des enregistrements
  * __readRow()__ : doit toujours être appelée pour parser le retour des __SELECT__ dans la base de données. C'est donc le seul endroit qui lit les résultats retournés par la base de données
  * __returnResponse()__ : toutes les méthodes doivent retourner une réponse uniformisée. __returnResponse()__ (généralement dans error.inc.php) renvoit un objet avec :
    * __data__ : qui contient le payload de la réponse ou le message d'erreur en cas d'erreur
    * __error__ : true || false
  * __getJoinString()__ : 
  * __readJoinString()__ : 
  * __Fonctions pures__ : les fonctions doivent être sans état externe (cookies, session ou autre). Elles doivent toujours renvoyer les mêmes résultats quand on passe les mêmes paramètres
  
# Github
  * tous les projets doivent être dans Github
  * le fichier README doit contenir toutes les informations permettant le déploiement du projet
  
# Backup
  * tous les projets doivent intégrer un script de backup
