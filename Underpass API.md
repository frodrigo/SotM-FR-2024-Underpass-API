---
title: Underpass API
css: 'style.css'
---

# Underpass API
## Overpass API sur base SQL

## SotM FR 2024

Frédéric Rodrigo - Teritorio

f.rodrigo@teritorio.fr

---

# Overpass
## Langage et API

- [Langage](https://wiki.openstreetmap.org/wiki/Overpass_API/Language_Guide) : requêtes spatialisées pour la structure de données d'OSM
- [API](https://wiki.openstreetmap.org/wiki/Overpass_API) : serveur d’exécution des requêtes

----

# Contexte

- Overpass un "standard"
- Alternative : MapCSS, CartoCSS
  - plus adapté à la sélection / CSS
  - style de carte
  - validation de données (JOSM / Osmose)

----

# Besoin

Réutiliser base de données SQL existant
- Sous projet de Clearance
  - Extracts OSM partiellements synchronisés
  - Une base SQL par extract
- Extraire des données OSM par API
  - Requêtes simples `[amenity=charging_station][bicycle=yes]`
  - Petit traitement (arrêts de bus avec ligne depuis la relation)

---

# Parser Overpass

Besoin d'une grammaire.

ANTLR4 : générateur de parseur depuis une grammaire.

----

## Anatomie d'une requête Overpass

```
[out:json][timeout:25];
area(3600166718)->.a;
(
  nwr[highway=bus_stop][name](area.a);
  nwr[public_transport=platform](area.a);
);
out center meta;
```

----

## Éléments de langage

```antlr
request: (metadata ';')? (query_object ';')+ (out ';')?;
```

```antlr
metadata: '[' 'out:json' ']' ( '[' 'timeout:' number ']' )?;
```

```antlr
query_object: type DOT_ID? selector* filter* asignation?;
selector: '[' ( (NOT? key) | (key OPERATOR value) ) ']';
filter: '(' (bbox|poly|osm_id|osm_ids|area|around) ')';
```

----

## La grammaire : Overpass.g4

https://github.com/teritorio/overpass_parser-rb/blob/master/Overpass.g4

----

## Partiel de grammaire

- Support partiel du langage Overpass
- Sortie JSON uniquement

---

# Overpass ⮕ SQL

- Overpass et SQL sont conceptuellement assez proches
  - peu de transformations nécessaires
- Produire un équivalemment SQL pour chaque élément de la grammaire

----

## Exemple (partiel)

```
node.a[a=b](1,2,3,4)
```

```sql
SELECT
  *
FROM
  _a
WHERE
  osm_type = 'n' AND
  (tags?'a' AND tags->>'a' = 'b') AND
  geom && ST_Envelope('LINESTRING(2.0 1.0, 4.0 3.0)'::geometry)
```

----

## (petit) Exemple (complet)

```
[out:json][timeout:25];
node(1)->.a;
out center meta;
```

```sql
SET statement_timeout = 25000;
WITH
_a AS (
  SELECT
    *
  FROM
    node
  WHERE
    osm_type = 'n' AND
    id = ANY (ARRAY[1])
)
SELECT
  jsonb_strip_nulls(jsonb_build_object(
    'type', CASE osm_type WHEN 'n' THEN 'node' WHEN 'w' THEN 'way' WHEN 'r' THEN 'relation' WHEN 'a' THEN 'area' END,
    'id', id,
    'lon', CASE osm_type WHEN 'n' THEN ST_X(geom)::numeric END,
    'lat', CASE osm_type WHEN 'n' THEN ST_Y(geom)::numeric END,
    'timestamp', created,
    'version', version,
    'changeset', changeset,
    'user', \"user\",
    'uid', uid,
    'center', CASE osm_type = 'w' OR osm_type = 'r'
      WHEN true THEN jsonb_build_object(
        'lon', ST_X(ST_PointOnSurface(geom))::numeric,
        'lat', ST_Y(ST_PointOnSurface(geom))::numeric
      )
    END,
    'nodes', nodes,
    'members', members,
    'tags', tags)) AS j
FROM
  _a
;
```

---

# SQL
## mais quel SQL ?

- Plusieurs dialectes SQL
- Plusieurs extensions spatiales
- Plusieurs schémas

----

## Dialectes SQL
## et extensions spatiales

- Postgres+PostGIS
- DuckDB+Spatial
  - BD orientée colonne et Cloud Optimized

⮕ Variantes pour des noms de fonctions, paramètres ou opérateurs

----

## Schémas

Besoin de convertir les données la nomenclature attendu par le code SQL généré.

"À la volée" par des vues : profiter du stockage et des index existants.

```sql
CREATE OR REPLACE TEMP VIEW node AS
SELECT
  id, version, tstamp, changeset_id, user_id, tags,
  NULL::bigint[] AS nodes,
  NULL::jsonb AS members,
  geom,
  'n' AS osm_type
FROM nodes;
```

----

## Adaptateurs existants

- DuckDB+Spatial / QuackOSM
- Postgres+PostGIS / Osmosis (Osmose)
- Postgres+PostGIS / OPE (Clearance)

---

# Et ça marche ?

Oui

----

## Plan d'exécution

Les moteurs de base de données réécrivent les requêtes.

Optimisation automatique :
- structure de données
- index
- statistiques des données

⮕ Ne pas introduire de blocage dans le code SQL généré pour permettre la réécriture.

----

## Les performances

```
[out:json][timeout:25];
(
  nwr[highway=bus_stop][name];
  nwr[public_transport=platform];
);
out center meta;
```

----

## Les performances

<table>
<tr><th>Backend</th><th>Setup</th><th>Query</th><th></th></tr>
<tr><td>Postgres+PostGIS / Osmosis</td><td>10m11s</td><td>5,7s</td><td></td></tr>
<tr><td>DuckDB+Spatial / QuackOSM 1</td><td>2m00s</td><td>2,4s</td><td></td></tr>
<tr><td>Overpass API 2</td><td>8m49s</td><td>3,1s</td><td></td></tr>
<tr><td>Overpass API overpass-api.de 3</td><td>-</td><td>5,9s</td><td></td></tr>
<tr></tr>
</table>

1. Without metadata.
2. Required converion from PBF to XML included ().
3. Query with polygon to limit the spatial extent.

---

## Underpass API

Sur Github
- [teritorio/Underpass-API](https://github.com/teritorio/Underpass-API)
- [teritorio/overpass_parser-rb](https://github.com/teritorio/overpass_parser-rb)

----

## Underpass++ API

- Améliorer le support du language Overpass
- Support d'autres DB (en contrib)
- Utiliser Osmose comme serveur Underpass API ?
