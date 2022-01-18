# Nettoyage des réseaux

Sommaire :
  - [Nettoyage de la base de données Geotrek](#nettoyage-de-la-base-de-données-geotrek)
  - [Nettoyage des données à intégrer](#nettoyage-des-données-à-intégrer)


L'objectif de cette partie est de présenter les principes et quelques techniques permettant d'avoir des données de qualité au sens topologique. Si vos données sont parfaites ce chapitre ne vous concerne pas.


## Nettoyage de la base de données Geotrek

Le but est de nettoyer sa base de données Geotrek de tous les artéfacts créés au fil des ans, des incohérences topologiques, etc. Pour cela, plusieurs requêtes SQL peuvent nous aider (ne pas oublier de créer un index spatial sur la colonne `geom` de la table `public.core_path` afin d'accélérer les requêtes) :

Identifier les tronçons qui en croisent d'autres sans être découpés à l'intersection :
``` sql
-- tronçons se croisant
SELECT DISTINCT cp1.id,
       'st_crosses' AS erreur
  FROM core_path cp1
       INNER JOIN core_path cp2
       ON ST_Crosses(cp1.geom, cp2.geom)
          AND cp1.id != cp2.id
```

Identifier les tronçons qui en chevauchent d'autres :
``` sql
-- tronçons se chevauchant
SELECT DISTINCT cp1.id,
       'st_overlaps or st_contains/within' AS erreur
  FROM core_path cp1
       INNER JOIN core_path cp2
       ON (ST_Overlaps(cp1.geom, cp2.geom)
           OR ST_Contains(cp1.geom, cp2.geom)
           OR ST_Within(cp1.geom, cp2.geom))
          AND cp1.id != cp2.id
```

Identifier les tronçons dont la géométrie n'est pas valide :
``` sql
-- tronçons NOT Valid
SELECT DISTINCT id,
       'st_isvalid' AS erreur
  FROM core_path
 WHERE NOT ST_IsValid(geom)
```

Identifier les tronçons qui s'auto-intersectent :
``` sql
-- tronçons NOT Simple
SELECT DISTINCT id,
       'st_issimple' AS erreur
  FROM core_path
 WHERE NOT ST_IsSimple(geom)
```

Identifier les tronçons qui ne sont pas une LineString :
``` sql
-- tronçons NOT LineString
SELECT DISTINCT id,
       'st_geometrytype' AS erreur
  FROM core_path
 WHERE NOT ST_GeometryType(geom) = 'ST_LineString'
```


Toutes ces requêtes devraient renvoyer extrêmement peu de tronçons, car Geotrek-admin effectue déjà un certain nombre de ces vérifications au moment de la création d'un tronçon. Cependant, il est toujours utile d'en avoir le coeur net.

Cherchons maintenant d'autres types de situations que nous voudrions corriger :

Identifier les tronçons en doublon :
``` sql
-- tronçons en doublon
SELECT cp1.id,
       cp2.id,
       'st_equals' AS erreur
  FROM core_path cp1
       INNER JOIN core_path cp2
       ON ST_Equals(cp1.geom, cp2.geom)
          AND cp1.id != cp2.id
```

Identifier les tronçons isolés du reste du réseau :
``` sql
-- tronçons isolés
SELECT cp1.id
  FROM core_path cp1
 WHERE NOT EXISTS
        (SELECT 1
           FROM core_path cp2
          WHERE cp1.id != cp2.id
            AND ST_INTERSECTS(cp1.geom, cp2.geom));
```


Et enfin, les tronçons qui se touchent presque, ce qui pourrait signifier qu'il leur manque un peu de longueur pour rentrer en contact et créer une intersection existant réellement :
``` sql
-- tronçons se touchant presque (régler la tolérance selon le besoin)
SELECT cp1.id,
       cp2.id
  FROM core_path cp1
       INNER JOIN core_path cp2
       ON ST_DWithin(ST_Boundary(cp1.geom), cp2.geom, 1)
          AND NOT ST_Intersects(cp1.geom, cp2.geom);
```
Ici la tolérance est placée à 1 mètre, mais elle peut être modifiée à souhait.

Le nettoyage peut ensuite se faire directement en base de données, ou bien via un SIG comme QGIS en créant la colonne `erreur` et en appliquant une symbologie catégorisée sur celle-ci.

Si le choix de la méthode base de données est fait, on peut utiliser l'extension `topology` de PostGIS pour automatiser la correction d'un certain nombre des erreurs décrites. Voir ce billet de blog de Mathieu Leplâtre pour la méthode : [http://blog.mathieu-leplatre.info/use-postgis-topologies-to-clean-up-road-networks.html]().
Attention, les résultats de ces opérations sont donc non maîtrisés, au contraire de la méthode manuelle qui est plus longue mais vous assure un nettoyage tel qu'il correspond à votre vision et vos besoins.


## Nettoyage des données à intégrer

Les données reçues devraient dans l'idéal déjà se soumettre à des principes topologiques fondamentaux :
- deux lignes ne peuvent pas se croiser, seulement se toucher aux extrémités => Si ce n'est pas le cas il faut découper chaque ligne de part et d'autre de l'intersection ;
- une ligne ne peut s'auto-croiser => Si ce n'est pas le cas il faut la découper en plusieurs lignes ;
- une géométrie ou un tronçon doit être de type LineString (et pas MultiLineString, Point, GeometryCollection...).

Seule exception au nettoyage manuel: une ligne touchée par l'extrémité d'une autre ligne n'a pas besoin d'être découpée, les triggers de Geotrek-admin s'en chargeront.

De manière plus subjective, pour garantir une qualité optimale du réseau, il est intéressant que :
- une intersection sur le terrain ne soit représentée que par une seule intersection dans le réseau => éviter de multiplier les intersections à quelques dizaines de centimètres d'écart (Cas des carrefours);
- des rues ou chemins qui sont connectées sur le terrain le soient aussi dans le réseau => éviter les "trous" de quelques centimètres qui donnent visuellement l'impression que les deux tronçons sont connectés jusqu'à ce qu'on zoome assez.

Pour cela, les mêmes requêtes spatiales que précédemment sont applicables, ici rassemblées en une seule pour plus de praticité (après création d'un index spatial pour plus de rapidité) :

``` sql
WITH
a AS ( -- tronçons qui en croisent d'autres
     SELECT DISTINCT cp1.id,
            'st_crosses' AS erreur
       FROM core_path cp1
            INNER JOIN core_path cp2
            ON ST_Crosses(cp1.geom, cp2.geom)
               AND cp1.id != cp2.id),
b AS ( -- tronçons qui en chevauchent d'autres
     SELECT DISTINCT cp1.id,
            'st_overlaps or st_contains/within' AS erreur
       FROM core_path cp1
            INNER JOIN core_path cp2
            ON (
				ST_Overlaps(cp1.geom, cp2.geom)
                OR ST_Contains(cp1.geom, cp2.geom)
                OR ST_Within(cp1.geom, cp2.geom)
			)
               AND cp1.id != cp2.id),
c AS ( -- tronçons dont la géométrie n'est pas valide
     SELECT DISTINCT id,
            'st_isvalid' AS erreur
       FROM core_path
      WHERE NOT ST_IsValid(geom)),
d AS ( -- tronçons qui s'auto-intersectent (normalement déjà pris en compte par st_isvalid)
     SELECT DISTINCT id,
            'st_issimple' AS erreur
       FROM core_path
      WHERE NOT ST_IsSimple(geom)),
e AS ( -- tronçons multilinestring
     SELECT DISTINCT id,
            'st_geometrytype' AS erreur
       FROM core_path
      WHERE NOT ST_GeometryType(geom) = 'ST_LineString'),
f AS ( -- tronçons en doublon
     SELECT cp1.id,
            'st_equals' AS erreur
       FROM core_path cp1
            INNER JOIN core_path cp2
            ON ST_Equals(cp1.geom, cp2.geom)
               AND cp1.id != cp2.id),
g AS ( -- tronçons isolés
     SELECT cp1.id,
            'isoles' as erreur
       FROM core_path cp1
      WHERE NOT EXISTS
            (SELECT 1
               FROM core_path cp2
              WHERE cp1.id != cp2.id
                AND ST_INTERSECTS(cp1.geom, cp2.geom)))
SELECT * FROM a
 UNION
SELECT * FROM b
 UNION
SELECT * FROM c
 UNION
SELECT * FROM d
 UNION
SELECT * FROM e
 UNION
SELECT * FROM f
 UNION
SELECT * FROM g
```
Attention un `core_path` peut avoir plusieurs erreurs et donc se retrouver plusieurs fois dans le résultat de la requête.

La seule requête à lancer de manière isolée est la suivante, car les identifiants des deux tronçons concernés sont nécessaires, et qu'il ne s'agit pas à proprement parler d'une erreur :
``` sql
-- tronçons se touchant presque (régler la tolérance selon le besoin)
SELECT cp1.id,
       cp2.id
  FROM core_path cp1
       INNER JOIN core_path cp2
       ON ST_DWithin(ST_Boundary(cp1.geom), cp2.geom, 1)
          AND NOT ST_Intersects(cp1.geom, cp2.geom);
```