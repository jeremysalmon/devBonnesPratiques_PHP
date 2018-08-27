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
  * __login__ : pour toutes les requêtes d'authentification (login, recherche de mail ...) il faut utiliser __LOWER(champs)__ et également mettre en minuscule dans le language de programmation utilisé le paramètre. Celà permet de faire par exemple des pages de login qui fonctionne sans avoir à respecter la casse.
  
  
# PHP
  * __/libs/__ : le répertoire __libs/__ contient toutes les classes et fichiers de configuration de l'application
  * __.class.php__ : toutes les classes se suffixent __.class.php__ et sont nommées du nom de la table correspondante
  * __.inc.php__ : tous les fichiers qui contenant des define() et des fonctions se suffixent __.inc.php__
  * __1 classe = 1 table__ : la méthode mirroir de facilement se retrouver dans le code
  * __save()__ : une seule méthode a le droit d'écrire dans la base de données. __save()__ s'occupe des INSERT et des UPDATE. Aucune autre méthode ne doit insérer ou mettre à jour des enregistrements
  * __readRow()__ : doit toujours être appelée pour parser le retour des __SELECT__ dans la base de données. C'est donc le seul endroit qui lit les résultats retournés par la base de données.
```
function readRow($row = false){
   if($row === false)
      return false;
      if(trim($row['id']) !== ''){
         $obj = new MA_CLASSE();
         foreach($row as $key => $value){
            switch($key){
               case "customdata":
                  $obj->customdata = json_decode($value);
                  break;
               default:
                  $obj->$key = trim($value);
            }
         }

         return $obj;
       }
       return false;
   }
```
S'il y a des jointures, alors il faut appeler la fonction __readJoinString()__ avant de retourner l'$obj
```
$obj->dependencies = new stdClass();
$depend = MA_CLASSE_JOINTE::readJoinString($row);
if($depend !== false)
   $obj->dependencies->MA_CLASSE_JOINTE = $depend;
```

  * __returnResponse()__ : toutes les méthodes doivent retourner une réponse uniformisée. __returnResponse()__ (généralement dans error.inc.php) renvoit un objet avec :
    * __data__ : qui contient le payload de la réponse ou le message d'erreur en cas d'erreur
    * __error__ : true || false
  * __getJoinString()__ : permet à partir d'une classe de prendre toutes les propriétés et de construire une requête SQL de jointure. Cette fonction existe, car il est impossible de préfixer automatiquement les champs d'une requête PostgreSQL.
  Le paramètre $prefix (optionnel) permet de prefixer en cas de multiple jointures sur la même table.
  ATTENTION : il ne faut pas oublier le _ dans NOM_DE_CLASSE_.
  
     ```function getJoinString($prefix = ''){
            $classVars = get_class_vars('NOM_DE_CLASSE');
			         $joinString = '';
			         foreach($classVars as $name => $value) {
				           $joinString .= ', NOM_DE_CLASSE.'.$name.' as '.$prefix.'NOM_DE_CLASSE_'.$name;
			         }
			         return $joinString;
            }
     
  * __readJoinString()__ : a utiliser dans readRow(). Permet de voir si on a une jointure dans les résultats lus. Si oui on retourne un objet instancié.
  ```
  function readJoinString§($dbId = false, $prefix = ''){
     if($row === false){
        return false;
     }
     if(isset($row[$prefix.'hposte_id'])){
        $obj = new NOM_CLASSE();
        $vars = get_class_vars('NOM_CLASSE');
        foreach ($vars as $key => $value) {
           if(isset($row[$prefix.'NOM_CLASSE_'.strtolower($key)])){
              switch (strtolower($key)) {
                 case 'customdata':
                    $obj->$key = json_decode($row[$prefix.'NOM_CLASSE_'.strtolower($key)]);    
                    break;
                 default:
                    $obj->$key = trim($row[$prefix.'NOM_CLASSE_'.strtolower($key)]);
                    break;
               }
            }
        }
        return $obj;
      }
      return false;
}
  ```
  
  * __Fonctions pures__ : les fonctions doivent être sans état externe (cookies, session ou autre). Elles doivent toujours renvoyer les mêmes résultats quand on passe les mêmes paramètres
  
  * __Utilisation de get et set__ : le principe est d'avoir un point d'entree et de sortie des données d'une classe. Tous les tests de validité des données peuvent ainsi être effectués dans le set par exemple.
  ```PHP
  <?php
  
class testClass implements \JsonSerializable{
        private $id;
        private $name;

        function __construct(){
                $this->id = false;
                $this->name = 'Inconnu';
        }

        function __set($prop, $value){
                $this->$prop = trim($value);
        }

        function __get($prop){
                return strtoupper($this->$prop);
        }

	/* Par défaut l'accès au $this ne fait pas appel aux __get.
	   Il faut donc implicitement l'appeler
	*/
        public function jsonSerialize(){
                $myObj = new stdClass();
                $myObj->id = $this->id;
                $myObj->name = $this->__get('name');
                return $myObj;
        }
}

$myClass = new testClass();

echo("1. Acces a une fonction private\n");
echo($myClass->name."\n");

echo("2. Objet complet\n");
echo(json_encode($myClass));

?>
  ```
  * __JSON__ : tous les webservices doivent ecrire le bon header : ```header('Content-type: application/json');```
# Javascript
  * __AJAX__ : les requêtes AJAX pour les insertions, modifications et suppressions sont __INTERDITES__. Les requêtes AJAX sont utilisées uniquement pour vérification d'informations, remplissage de listes déroulantes ...
  * __Id & Name__ : les ID et NAME des input doivent être les mêmes que ceux de la base de données. Ceci permet de faire facilement du binding (une boucle qui parcours l'objet et qui renseigne correctement les champs)
  * 


# Github
  * tous les projets doivent être dans Github
  * le fichier README doit contenir toutes les informations permettant le déploiement du projet
  
# Backup
  * tous les projets doivent intégrer un script de backup

# Serveur Web
  * __Nginx-Naxsi__ : il est preferable d'utiliser nginx-naxsi (disponible dans les depots debian) qui est un WAF permettant une premiere protection contre les XSS et SQL Injection. Il faut tout de meme veiller aux faux positifs.
  * __.git & autres__ : il faut creer des regles dans le fichier de configuration de Nginx pour interdire la navigation et la recuperation de fichiers dans les repertoires sensibles (.git, README, changelog ...)
  * __fail2ban__ : pour la partie authentification ou detection d'activite suspecte, le moyen le plus simple est d'ecrire des ```error_log``` en PHP et de creer des regles fail2ban (TODO : mettre des exemples)
  * __headers HTTP__ : configurer Nginx pourqu'il mette au minimum les headers suivants :
    * TODO
  * __TLS__ : configurer uniquement TLSv1.2 / v1.3 et les ciphers suivants :
    * TODO
  * __DMZ__ : si le serveur n'est pas chez un hebergeur, le mettre dans un DMZ
  * __Ports__ : bien veiller (avec un nmap par exemple) que seuls les ports necessaires sont ouverts depuis l'exterieur sur le serveur
  * __SSH authorized_keys__ : ne configurer que les cles authorisees. ```ssh-copy-id -i id_rsa.pub root@IP_DISTANTE```
    
