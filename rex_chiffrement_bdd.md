# Chiffrement de base de données



## Ancienne manière de chiffrer

Le chiffrement des données est fait en utilisant le bundle (bibliothèque) Symfony [nzo/url-encryptor-bundle](https://packagist.org/packages/nzo/url-encryptor-bundle) en version 3 qui s'appuie sur la bibliothèque de cryptologie [Mcrypt](https://en.wikipedia.org/wiki/Mcrypt) et donc l'extension [php-mcrypt](http://php.net/manual/fr/intro.mcrypt.php). Un des soucis majeurs concernant l'utilisation de Mcrypt est que cette bibliothèque n'est plus maintenue et comporte [des failles de sécurité](https://www.cvedetails.com/vulnerability-list/vendor_id-1643/Mcrypt.html). La communauté PHP a décidé d'arrêter le support de Mcrypt après PHP7.1


### Recherche dans les données chiffrées

La rechercher dans les données chiffrées est faites de la manière suivante :

* Récupération de toutes les données concernant l'utilisateur.
* Toutes ces données sont ensuite déchiffrées.
* Le filtrage est ensuite fais en utilisant des boucles et des expression régulières.


### Première Analyse

php-mcrypt est dépréciée depuis PHP7.1 hors la version courante de PHP est 7.3.

La dépendance vers nzo/encryptor-url-bundle qui n'est pas fait pour faire du chiffrement de base de données.

La manière de rechercher dans les données chiffrées qui n'est pas optimale et viable sur le long terme, car plus l'utilisateur aura de données plus cela prendra du temps de filtrer sa recherche.



## Pistes d'amélioration



### Chiffrer avec la base de données

À premier vu l'idée de réaliser le chiffrement du cote serveur peut sembler étrange puisque MariaDB possede des fonctions native pour chiffrer les donnees, [AES_ENCRYPT](https://mariadb.com/kb/en/library/aes_encrypt/) et [AES_DECRYPT](https://mariadb.com/kb/en/library/aes_decrypt/). Cette solution impose de faire le chiffrement au niveau des requêtes SQL :
```
INSERT INTO t VALUES (AES_ENCRYPT('text',SHA2('password',512)));
```
Cette manière, de procéder est contraignante à mettre en place, puisque le projet utilise Doctrine qui génère les requêtes en base de donnees or nativement il n'y a aucun moyen de dire a Doctrine d'utiliser le chiffrement natif SQL. Une solution envisagée était de mettre en place [des types personnalisés doctrine](https://www.doctrine-project.org/projects/doctrine-orm/en/2.6/cookbook/custom-mapping-types.html).

Cette solution semblait viable jusqu'à la découverte de comment faire la recherche qui ressemble presque à la façon de faire décrite précédemment a la différence que tout ce passe dans la base de données. En substance, la base déchiffre toutes les données de la colonne concernées et fait la recherche.


### Chiffrer les fichiers de base de données

Une piste explorée et qui a été adoptée est le chiffrement des fichiers de base de données. Cette méthode consiste a chiffrer les fichiers disque ou sont stocker les données, car si une personne s'introduit sur un système elle n'a nullement besoin d'avoir accès a la base pour voler les données il lui suffit d'avoir accès aux fichiers de base de donnes et avec quelques utilitaire Linux.

Une traduction française de la documentation de MariaDB pour [chiffrer les fichiers](http://www.christophe-meneses.fr/article/chiffrer-une-base-de-donnees-mariadb)


### Bundle doctrine

Il y a une multitude de bundle Symfony qui permettent de chiffrer des données assez simplement mais avec toutes les mêmes défaut une incapacité à faire des recherches full text.
Après plusieurs recherches, aucune solution n'a été retenue.



## La solution

Les recherches nous ont vers un article parlant de [recherche par bindIndex](https://paragonie.com/blog/2017/05/building-searchable-encrypted-databases-with-php-and-sql)

Le principe derrière est assez simple. Les données sont chiffrées avec des algorithmes de chiffrement complexe type [CHACHA20 et POLY1305](https://www.bortzmeyer.org/7539.html), l'intérêt d'utiliser ces algorithmes se retrouve lorsque l'on doit chiffrées une deuxième fois la même données le résultat est diffèrent.

De ce fait la recherche full text ou même classique devient compliquée, les blind index sont en fait des colonnes indexées qui contiennes les 8 derniers caractères du HMAC des valeurs chiffres. Comme ce HMAC ne change pas, lors recherche classique, il suffit de recherche dans la colonne contenant le bon HMAC.


### Recherche full text

Le cas porte sur la recherche de personne les noms et prénom sont chiffrées.
L'utilisateur doit pouvoir rechercher comme il veut la personne soit nom prénom soit prénom nom.
Pour ce faire une table dédiée a la recherche a été créé, contenant un lien vers la personne, un filtre et la valeur du filtre chiffrées, ces deux colonnes sont indexées dans la base de données pour plus de performances.
Pour alimenter cette table, le nom et prénom de l'utilisateur sont concaténer ie (jean-michelexemple) dans les deux sens (nomprenom et prenomnom).
Puis la chaîne est nettoyée, c'est à dire les accents et autres caractère spéciaux sont remplacés par le caractère non accentué, les espaces et tiret sont supprimes.
Ensuite, pour les deux chaînes, à partir du troisième caractère, on calcule les valeurs chiffrées a stocker, puis on avance d'un caractère et ainsi de suite jusqu'à la fin de la chaîne.

En base, nous avons les données suivantes :

```
jea
jean
jeanm
```

Lors de la recherche l'utilisateur entre **pren** le HMAC de cette valeur est calculer puis insérer dans la requêtes de recherche.