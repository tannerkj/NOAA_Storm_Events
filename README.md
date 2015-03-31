## Visualizing Percentage of Storms Resulting in Injuries/Deaths by State (1996 - 2014)

Starting in 1996, [National Weather Service directive 10-1605](http://www.ncdc.noaa.gov/stormevents/pd01016005curr.pdf) mandated the reporting of 48 different types of weather events and their effects on the impacted community.  This tutorial aims to create a visualization focusing on the percent of storms resulting in injuries/deaths reported by state, and to identify which states 
have been more fortunate than others.  The data is provided by [NOAA](http://www.ncdc.noaa.gov/stormevents/ftp.jsp) in .csv files organized by individual year.

The first step is to create the tables in PostgreSQL and load the data.  A new database is created and the following code executed to enable postgis:

	CREATE EXTENSION postgis;

Next, the code to generate the table is executed:

	CREATE TABLE storm_events (
		state varchar(20),
		year smallint,
		state_fips smallint,	
		event_type varchar(50),
		injuries_direct smallint,
		injuries_indirect smallint,
		deaths_direct smallint,
		deaths_indirect smallint
	);
	
Included are the attributes we need for this application, along with additional fields we may opt to use in the future for further analysis.

To create a surrogate primary key called id for this table, the following code is executed:

	ALTER TABLE storm_events ADD COLUMN id SERIAL PRIMARY KEY;

To load the data into the table, several options are available.  One option, though not the most efficient, is outlined below:

1) a python script is written to read from each year's .csv file in our details_raw folder and creates newly formatted .csv files, excluding the unnecessary data.

	# data_loader.py
	# reads .csv files in ./details_raw
	# creates new .csv files in a separate directory
	
	import os, csv

	in_dir = "./details_raw"

	def filter_fields(f):
    	with open(root + "/" + f, "rb") as source:
        	reader = csv.DictReader(source)
        	outf = root + "/formatted/" + f
        	with open(outf, "wb") as result:
          		writer = csv.DictWriter(result, fieldnames = ['STATE', 'YEAR','STATE_FIPS', 
          		'EVENT_TYPE', 'INJURIES_DIRECT', 'INJURIES_INDIRECT', 'DEATHS_DIRECT', 'DEATHS_INDIRECT'])
            	print ("Writing", outf)
            	writer.writeheader()
            	for row in reader:
                	writer.writerow({'STATE' : row['STATE'],
                                 	'YEAR' : row['YEAR'],
                                 	'STATE_FIPS' : row['STATE_FIPS'],
                                 	'EVENT_TYPE' : row['EVENT_TYPE'],
                                 	'INJURIES_DIRECT' : row['INJURIES_DIRECT'],
                                 	'INJURIES_INDIRECT' : row['INJURIES_INDIRECT'],
                                 	'DEATHS_DIRECT' : row['DEATHS_DIRECT'],
                                 	'DEATHS_INDIRECT' : row['DEATHS_INDIRECT']})

	for root, dirs, files in os.walk(in_dir):
    	for f in files:
        	filter_fields(f)
	
2) Using postgres command line tools for each csv file:

	psql -U postgres
	
	\connect geodata
	\copy storm_events from '/Path to csv' delimiter ',' csv header
	...
	
Now, to create views that allow us to analyze the data you have several options but below are a few simple ones that isolate the info we are interested in.  They also allow room for growth should you choose to add aggregates of other attributes:

	CREATE VIEW injuries_deaths AS
	SELECT storm_events.state, storm_events.state_fips,
	count(*) AS count
	FROM storm_events
	WHERE storm_events.injuries_direct <> 0 OR
	storm_events.injuries_indirect 	<> 0 OR
	storm_events.deaths_direct <> 0 OR
	storm_events.deaths_indirect <> 0
	GROUP BY storm_events.state, state_fips
	ORDER BY storm_events.state;

	CREATE VIEW state_totals AS
	SELECT total.state, total.state_fips,
	count(*) AS count
	FROM (SELECT storm_events.state,
           storm_events.year,
           storm_events.state_fips,
           storm_events.event_type,
           storm_events.injuries_direct,
           storm_events.injuries_indirect,
           storm_events.deaths_direct,
           storm_events.deaths_indirect
          FROM storm_events) total
    GROUP BY total.state, total.state_fips
    ORDER BY total.state;

	CREATE VIEW map_storm_data AS
	(SELECT s.state, s.state_fips, id.count injuries_death_count, s.count         	
	total_storms_count, round(((id.count::float/s.count::float)*100)::numeric,2) percent
	FROM injuries_deaths id
	LEFT JOIN state_totals s ON id.state = s.state);
	
A simple query to validate the data and identify the states with the highest percentage of storms resulting in injury/death:

	SELECT *
	FROM map_storm_data
	ORDER BY percent DESC;
	
To generate a shapefile we can use to visualize our data,  download from TIGER the shapefile needed, and then use command line to convert it to the desired projection and import it into postgres.  In this case, I entered:

	shp2pgsql -s 4269:4326 tl_2014_us_state  public.states | psql -h 
	localhost -d geodata -U postgres
	
Now to create a single view that includes the desired calculations along with the associated state geometry data:

	CREATE VIEW state_storm_percent AS
	SELECT a.name, b.state_fips, b.injuries_death_count,
	b.total_storms_count, b.percent, ST_Simplify(a.geom, 0.005) as geom
	FROM states a
	LEFT JOIN map_storm_data b
	ON a.statefp::int = b.state_fips;
	
	
For constructing the application, we will use Leaflet and GeoJSON.  To do this, we will export the map as a shapefile, and use gdal to convert it to geojson.

The following code is executed in the command line for PostgreSQL to generate the shapefile and then convert to geojson:

	pgsql2shp -f storm_events -h localhost -p 5432 -u postgres -P password
	geodata public.state_storm_percent;
	
	ogr2ogr -f GeoJSON storm_events.geojson storm_events.shp
	
To create the map for our application, we will use the same process as described in this [Leaflet tutorial](http://leafletjs.com/examples/choropleth.html).  The end result is a thematic map with a legend and an info window describing the data as you select each state.

View application [here](http://tannerkj.github.io/NOAA_Storm_Events/index.html). 
	

	

	
