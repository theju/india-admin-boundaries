# India admin boundaries
GIS information of Indian admin boundaries (includes disputed territories)

## Credits
The datameet google group. More specifically, Antariksh Tyagi for his reply at https://groups.google.com/forum/#!topic/datameet/X5kzViRMJKs

### Procedure
* Download the [file](http://mnre.gov.in/sec/INDIA.zip) from the [Ministry of Non-Renewable Energy](http://mnre.gov.in/sec/solar-assmnt.htm) website.
  The file (`INDIA.zip`) is available in this repo itself but may not be the latest. If you are worried about the license of this file, please check
  with the NRE Department, Government of India.
* Create a PostGIS enabled database:

  ```shell
  $ createdb india_db
  $ psql -d india_db -c "CREATE EXTENSION postgis;"
  $ psql -d india_db -c "CREATE EXTENSION postgis_topology;"
  $ unzip INDIA.zip -d INDIA
  $ cd INDIA
  $ shp2pgsql -s 4326 STATE.shp > state.sql
  $ shp2pgsql -s 32644:4326 district_Project.shp > district.sql
  $ psql -d india_db -f state.sql
  $ psql -d india_db -f district.sql
  $ psql -d india_db
  ```
* Since the data does not have Telangana's state boundaries (at the time of writing), we will have to generate them:

  ```sql
  UPDATE district_project SET state_name='TELANGANA' WHERE district IN ('ADILABAD', 'HYDERABAD (CAPITAL)', 'KARIMNAGAR', 'KHAMMAM', 'MAHBUBNAGAR', 'SANGAREDDI (MEDAK)', 'NALGONDA', 'NIZAMABAD', 'RANGAREDDI', 'WARANGAL');
  UPDATE state SET geom=(SELECT ST_MULTI(ST_UNION(geom)) FROM district_project WHERE state_name='ANDHRA PRADESH') WHERE state_name='ANDHRA PRADESH';
  INSERT INTO state (state_name, geom) VALUES ('TELANGANA', (SELECT ST_MULTI(ST_UNION(geom)) FROM district_project WHERE state_name='TELANGANA'));
  ```
* (Optional) If you want to update the area and perimeter of the states, run the following sql:

  ```sql
  UPDATE state SET area=ST_Area(geom), perimeter=ST_Perimeter(geom);
  ```
* To generate the boundaries GeoJSONs:

  ```sql
  -- State boundaries
  COPY (SELECT ST_AsGeoJSON(ST_Union(ST_Boundary(geom))) FROM state) TO '/tmp/state_boundary.geojson';
  -- District boundaries
  COPY (SELECT ST_AsGeoJSON(ST_Collect(ST_Boundary(geom))) FROM district_project) TO '/tmp/district_boundary.geojson';
  ```

