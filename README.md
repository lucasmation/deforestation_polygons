# deforestation_polygons
Describes how use POSTGIS to import and correct deforestation polygons provided by DETER/INPE


### Downloading the data: 
at Linux terminal, Create folder, cd into it, and use wget to download the data and unzip to unpack: 
```
mkdir ~/temp_deforestation_data
cd    ~/temp_deforestation_data
wget -r -nd -A *shp.zip -nc --read-timeout=60 http://www.dpi.inpe.br/prodesdigital//dadosn/mosaicos/2013/
unzip(*shp.zip)
``` 

### Importing into PostGis:
Doing for AC now. Change file paths to replicate to other states.
At the Linux terminal run:
```
shp2pgsql -c -s 4674:4326 -I -W LATIN1 PDigital2013_AC_pol  public.PDigital2013_AC_pol   | psql -d deforestation
```
obs: -s 4674:4326 reproject from SAD69 to WGS84



from this point on run the SQL code at PgAdmin (or psql) 

### find invalid polygons:

Count invalid polygons:
```
select count(gid), reason(ST_IsValidDetail(geom)) as razao 
  from pdigital2013_ac_pol
  where IsValid(geom)=false 
  group by reason(ST_IsValidDetail(geom)) 
```

### Solving the problem: 

Tryign to solve with ST_MakeValid().
```
select gid, geom,ST_MakeValid(geom) 
  from pdigital2013_ac_pol 
  where IsValid(geom)=false;
```
It solves most cases but the query bellow shows that two problematic polygons remain
```
select gid , IsValid(geom), IsValid(ST_MakeValid(geom)) 
  from pdigital2013_ac_pol 
  where IsValid(geom)=false ;
```
the problematic cases are due  to the corrected spatial unit being a 'GEOMETRYCOLLECTION', composed of a poligons and a line. In theese cases, bellow, we will use just the poligon part. 


The actual solution is:
```
ALTER TABLE pdigital2013_ac_pol
  ADD COLUMN geom_C geometry(MultiPolygon,4326);
update pdigital2013_ac_pol
  set geom_C=geom;
update pdigital2013_ac_pol 
	set geom_C=ST_MakeValid(geom) 
	where IsValid(geom)=false and GeometryType(ST_MakeValid(geom))='MULTIPOLYGON'
update pdigital2013_ac_pol 
	set geom_C=ST_CollectionExtract(ST_MakeValid(geom),3) 
	where IsValid(geom)=false and GeometryType(ST_MakeValid(geom))='GEOMETRYCOLLECTION'
```  

The corrected geometry is geom_C (C for "corrected").
We can confinrm gemp_C is valid: 
```
select count(gid) , IsValid(geom_C) 
  from pdigital2013_ac_pol 
  group by IsValid(geom_C)
```
