Model
*****

In trackintel, **tracking data** is split into several classes. It is not generally 
assumed that data is already available in all these classes, instead, trackintel
provides functionality to generate everything starting from the raw GPS positionfix data 
(consisting of at least ``(user_id, tracked_at, longitude, latitude)`` tuples).

* **positionfixes**: Raw GPS data.
* **staypoints**: Locations where a user spent a minimal time.
* **triplegs**: Segments covered with one mode of transport.
* **locations**: Clustered staypoints.
* **trips**: Segments between consecutive activity staypoints (special staypoints that are not just waiting points).
* **tours**: Sequences of trips which start and end at the same location (if the column 'journey' 
  is True, this location is *home*).

An example plot showing the hierarchy of the trackintel data model can be found below:

.. image:: /assets/hierarchy.png
	:align: center
	:height: 390px
	:width: 492px
	:alt: model image

The image below explicitly shows the definition of locations as clustered staypoints, generated by one or several users.

.. image:: /assets/locations_with_pfs.png
	:align: center
	:height: 405px
	:width: 720px 
	:scale: 70
	

A detailed (and SQL-specific) explanation of the different classes can be found under 
:ref:`data_model`.

GeoPandas Implementation
========================

.. highlight:: python

In trackintel we provide 5 classes that subclass DataFrames or GeoDataFrames, depending if there is a
geometry or not. That means all trackintel classes work like (Geo)DataFrames with some additional features, 
like: data validation and new methods for analyzing mobility data.
For example::

    df = trackintel.read_positionfixes_csv('data.csv')
    df.generate_staypoints()

This will read a CSV into a `Positionfixes` trackintel class and validate the input data corresponding to
our model. This allows access to the trackintel methods such as ``generate_staypoints()``.

Trackintel Classes
===================

The following classes are implemented within *trackintel*.

Positionfixes
---------------------

.. autoclass:: trackintel.Positionfixes
	:members:

Staypoints
------------------

.. autoclass:: trackintel.Staypoints
	:members:

Triplegs
----------------

.. autoclass:: trackintel.Triplegs
	:members:

Locations
-----------------

.. autoclass:: trackintel.Locations
	:members:

Trips
-------------

.. autoclass:: trackintel.Trips
	:members:

Tours
-------------

.. autoclass:: trackintel.Tours
	:members:



Data Model (SQL)
================


For a general description of the data model, please refer to the 
:doc:`/modules/model`. You can download the 
complete SQL script `here <https://github.com/mie-lab/trackintel/blob/master/sql/create_tables_pg.sql>`__
in case you want to quickly set up a database. Also take a look at the `example on github 
<https://github.com/mie-lab/trackintel/blob/master/examples/setup_example_database.py>`_.

.. highlight:: sql

The **positionfixes** table contains all positionfixes (i.e., all individual GPS trackpoints, 
consisting of longitude, latitude and timestamp) of all users. They are not only linked to 
a user, but also (potentially, if this link has already been made) to a tripleg or a staypoint::

    CREATE TABLE positionfixes (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id bigint NOT NULL,

        -- References to foreign tables.
        tripleg_id bigint,
        staypoint_id bigint,

        -- Temporal attributes.
        tracked_at timestamp with time zone NOT NULL,

        -- Spatial attributes.
        elevation double precision,
        geom geometry(Point, 4326),

        -- Constraints.
        CONSTRAINT positionfixes_pkey PRIMARY KEY (id)
    );

The **staypoints** table contains all stay points, i.e., points where a user stayed
for a certain amount of time. They are linked to a user, as well as (potentially) to a trip
and location. Depending on the purpose and time spent, a staypoint can be an *activity*,
i.e., a meaningful destination of movement::

    CREATE TABLE staypoints (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id bigint NOT NULL,

        -- References to foreign tables.
        trip_id bigint,
        location_id bigint,

        -- Temporal attributes.
        started_at timestamp with time zone NOT NULL,
        finished_at timestamp with time zone NOT NULL,
        
        -- Attributes related to the activity performed at the staypoint.
        purpose_detected character varying,
        purpose_validated character varying,
        validated boolean,
        validated_at timestamp with time zone,
        activity boolean,

        -- Spatial attributes.
        elevation double precision,
        geom geometry(Point, 4326),

        -- Constraints.
        CONSTRAINT staypoints_pkey PRIMARY KEY (id)
    );

The **triplegs** table contains all triplegs, i.e., journeys that have been taken 
with a single mode of transport. They are linked to both a user, as well as a trip 
and if applicable, a public transport case::

    CREATE TABLE triplegs (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id bigint NOT NULL,

        -- References to foreign tables.
        trip_id bigint,

        -- Temporal attributes.
        started_at timestamp with time zone NOT NULL,
        finished_at timestamp with time zone NOT NULL,

        -- Attributes related to the transport mode used for this tripleg.
        mode_detected character varying,
        mode_validated character varying,
        validated boolean,
        validated_at timestamp with time zone,

        -- Spatial attributes.
        -- The raw geometry is unprocessed, directly made up from the positionfixes. The column
        -- 'geom' contains processed (e.g., smoothened, map matched, etc.) data.
        geom_raw geometry(Linestring, 4326),
        geom geometry(Linestring, 4326),

        -- Constraints.
        CONSTRAINT triplegs_pkey PRIMARY KEY (id)
    );

The **locations** table contains all locations, i.e., somehow created (e.g., from clustering
staypoints) meaningful locations::

    CREATE TABLE locations (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id bigint,
        
        -- Spatial attributes.
        elevation double precision,
        extent geometry(Polygon, 4326),
        center geometry(Point, 4326),

        -- Constraints.
        CONSTRAINT locations_pkey PRIMARY KEY (id)
    );

The **trips** table contains all trips, i.e., collection of trip legs going from one 
activity (staypoint with ``activity==True``) to another. They are simply linked to a user::

    CREATE TABLE trips (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id integer NOT NULL,

        -- References to foreign tables.
        origin_staypoint_id bigint,
        destination_staypoint_id bigint,

        -- Temporal attributes.
        started_at timestamp with time zone NOT NULL,
        finished_at timestamp with time zone NOT NULL,

        -- Constraints.
        CONSTRAINT trips_pkey PRIMARY KEY (id)
    );

The **tours** table contains all tours, i.e., sequence of trips which start and end 
at the same location (in case of ``journey==True`` this location is *home*). 
They are linked to a user::

    CREATE TABLE tours (
        -- Common to all tables.
        id bigint NOT NULL,
        user_id integer NOT NULL,

        -- References to foreign tables.
        location_id bigint,

        -- Temporal attributes.
        started_at timestamp with time zone NOT NULL,
        finished_at timestamp with time zone NOT NULL,
        
        -- Specific attributes.
        journey bool,

        -- Constraints.
        CONSTRAINT tours_pkey PRIMARY KEY (id)
    );
