timeseries
==========

The timeseries extension implements a new data type, `timeseries`, as well as a number of utility
functions for creating, updating, selecting, and transforming them and the sample values which
comprise them.

Availability
------------

This extension is experimental and undergoing development. Because of this, the
software is unstable and source code quality is deemed unsuitable for general
release at this time. Contact the author to request access to the distribution
source code.

Installation
------------

Read the INSTALL file in the distribution for instructions on building the
extension. Once built, it is installed into a PostgreSQL database with the
following `psql` command:

    CREATE EXTENSION timeseries;

Dependencies
------------

The `timeseries` extension requires PostgreSQL 9.0 or higher.

Usage
-----

### Timeseries Data Types ###

#### timeseries ####

A `timeseries` is another data type like any of the pre-existing ones which come with PostgreSQL
and defines the type for a column in a table just like the others:

    CREATE TABLE samples (
        key varchar NOT NULL,
        ts timeseries NOT NULL
    );

A `timeseries` column is populated by calling the `create` function (described below) and has a
number of properties which are specified at creation time:

##### data_type #####

The underlying data type of the samples collected within the `timeseries`. It can be any of the
fixed-width PostgreSQL numeric types: `bigint`, `double precision`, `integer`, `real`, `smallint`.

##### period #####

A `timeseries` is composed of a collection of samples taken at a regular, repeated interval and this
property specifies this period in seconds.

##### epoch #####

Every `timeseries` is defined over a finite interval of time as specified by its period and extent
and this property specifies the start of the interval as a PostgreSQL `timestamp`.

##### extent #####

This property specifies the extent, or number of samples at the `timeseries` period which together
with the epoch determines the finite time interval over which it is defined.

##### measure_mode #####

This property specifies the interpretation of the sample values within the `timeseries`: it can be
either a `COUNTER`, which is a monotonically increasing count of some observed phenomenon (for
example, the number of clicks on a specific ad banner on a web page, or the number of packets
transmitted across a certain network interface), or a `GAUGE`, which is an instantaneous measurement
of some physical phenomenon (such as the current temperature at a given location).

##### rollover_mode #####

As `timeseries` are defined over a finite time interval as explained above, this property specifies
the behavior whenever new samples are recorded outside of the interval: it can either `GROW` beyond
its original interval (if this happens, the `extent` is increased), `WRAP` around and overwrite
existing samples, or `STOP` and silently discard the new samples being added.

##### gapfill_mode #####

A `timeseries` is a repeated sequence of observations of some phenomenon at a fixed frequency, but
it is often the case that an observation will fail to be made or recorded. When this happens, the
gaps between the last last recorded observation and the newest will be filled according to the value
of this property: either a `DEFAULT_VALUE` as specified by the `gap_value` property will be filled in
or the previously recorded value is `CARRY`'d forward.

##### interpolation_mode #####

Similar to the `gapfill_mode`, this property provides another method to fill gaps when there are
missing samples: if `NONE`, no interpolation is performed (and the `gapfill_mode` specifies how to
handle the gap), if `LINEAR`, simple linear interpolation is performed, and if `CUBIC`, cubic spline
interpolation is performed. Note that if not enough samples are present to perform the indicated
interpolation, a `NULL` value may be interpolated.

##### gap_value #####

This property determines the value to use when `gapfill_mode` is `DEFAULT_VALUE`. It may be `NULL`.

##### min_value #####

This property determines the minimum allowed value in the range of the sample data. If a sample
with a lower value than this is recorded, an error is raised and the update fails.

##### max_value #####

This property determines the maximum allowed value in the range of the sample data. If a sample
with a higher value than this is recorded, an error is raised and the update fails.

#### timeseries_slice ####

Another data type introduced by the extension is `timeseries_slice`. This is an intermediate
type used while processing time series data pipelines and cannot be directly instantiated by
database users, but it is mentioned here since it appears in some of the function signatures
below.


### Overview of Concepts and Data Flow ###

Time series exist both in the database's durable storage and in memory during processing and
transformation (corresponding to the `timeseries` and `timeseries_slice` data types mentioned
above). Processing and transformation require any input time series to be equivalent when used
together in any composable operation. This means:

* The `period` must be the same.
* The time interval covered by each time series must be the same.

Therefore, a typical work flow involves the following steps at minimum:

1. Select one or more `timeseries` from database tables using an ordinary SQL query.
2. Use the `select()` function to open the `timeseries` and prepare it for participating in processing.
3. Use the `resample()` function to ensure each selected `timeseries` has the same `period` (see below).
4. Use the `extend()` function to ensure each selected `timeseries` spans the same time interval (see below).
5. Perform some transformational or combinatorial operation on the `timeseries`.
6. Emit the results using `as_array()` (if a simple vector of the time series sample values is sufficient) or
   with `as_tuple()` if the results must be joined with other tables in a relational query.

#### Example ####

Given the following table of `timeseries` modeling the network bandwidth usage into and out of a hyptothetical
ISP (Internet Service Provider) for various types of traffic routing relationships with customers and other
ISPs:

    CREATE TABLE network_stats (
      name VARCHAR NOT NULL,
      peer_type VARCHAR NOT NULL,
      direction VARCHAR NOT NULL,
      ts timeseries NOT NULL
    );

    Each row has a `timeseries` for either ingress or egress bandwidth sampled at one minute granularity
    (every 60 seconds) over a 24-hour period for a specific customer or partner ISP:

    INSERT INTO network_stats VALUES ('CARRIER3-EAST', 'TRANSIT_PEER', 'INGRESS',
      create('FLOAT8', 60, '2015-03-01 00:00:00', 1440, 'GAUGE', 'WRAP', 'DEFAULT_VALUE', 'NONE'));
    INSERT INTO network_stats VALUES ('CARRIER3-EAST', 'TRANSIT_PEER', 'EGRESS',
      create('FLOAT8', 60, '2015-03-01 00:00:00', 1440, 'GAUGE', 'WRAP', 'DEFAULT_VALUE', 'NONE'));
    INSERT INTO network_stats VALUES ('CARRIER3-WEST', 'TRANSIT_PEER', 'INGRESS',
      create('FLOAT8', 60, '2015-03-01 00:00:00', 1440, 'GAUGE', 'WRAP', 'DEFAULT_VALUE', 'NONE'));
    INSERT INTO network_stats VALUES ('CARRIER3-WEST', 'TRANSIT_PEER', 'EGRESS',
      create('FLOAT8', 60, '2015-03-01 00:00:00', 1440, 'GAUGE', 'WRAP', 'DEFAULT_VALUE', 'NONE'));
    INSERT INTO network_stats VALUES ('SINGULARSYNTAX-NET-SFO', 'CUSTOMER_PEER', 'INGRESS',
      create('FLOAT8', 60, '2015-03-01 00:00:00', 1440, 'GAUGE', 'WRAP', 'DEFAULT_VALUE', 'NONE'));
    INSERT INTO network_stats VALUES ('SINGULARSYNTAX-NET-SFO', 'CUSTOMER_PEER', 'EGRESS',
      create('FLOAT8', 60, '2015-03-01 00:00:00', 1440, 'GAUGE', 'WRAP', 'DEFAULT_VALUE', 'NONE'));
    INSERT INTO network_stats VALUES ('SINGULARSYNTAX-NET-NYC', 'CUSTOMER_PEER', 'INGRESS',
      create('FLOAT8', 60, '2015-03-01 00:00:00', 1440, 'GAUGE', 'WRAP', 'DEFAULT_VALUE', 'NONE'));
    INSERT INTO network_stats VALUES ('SINGULARSYNTAX-NET-NYC', 'CUSTOMER_PEER', 'EGRESS',
      create('FLOAT8', 60, '2015-03-01 00:00:00', 1440, 'GAUGE', 'WRAP', 'DEFAULT_VALUE', 'NONE'));

    timeseries=# SELECT * FROM network_stats;
              name          |   peer_type   | direction |  ts   
    ------------------------+---------------+-----------+-------
     CARRIER3-EAST          | TRANSIT_PEER  | INGRESS   | 63620
     CARRIER3-EAST          | TRANSIT_PEER  | EGRESS    | 63621
     CARRIER3-WEST          | TRANSIT_PEER  | INGRESS   | 63622
     CARRIER3-WEST          | TRANSIT_PEER  | EGRESS    | 63623
     SINGULARSYNTAX-NET-SFO | CUSTOMER_PEER | INGRESS   | 63624
     SINGULARSYNTAX-NET-SFO | CUSTOMER_PEER | EGRESS    | 63625
     SINGULARSYNTAX-NET-NYC | CUSTOMER_PEER | INGRESS   | 63626
     SINGULARSYNTAX-NET-NYC | CUSTOMER_PEER | EGRESS    | 63627
    (8 rows)

The following query aggregates inbound bandwidth from partner ISPs over a day after resampling the data
to five minute granularity and converts to kilobytes per second (assuming it was sampled as kilobits/sec):

    SELECT
      as_tuple
        (lambda
          ('/', agg_lambda
            (resample
              (extend
                (ts.select(ts),
                  '2015-03-01 00:00:00', '2015-03-01 23:55:00'),
             300),
           '+'),
         8)) -- convert kbits to kbytes
    FROM network_stats WHERE peer_type = 'TRANSIT_PEER' AND direction = 'INGRESS';


### Constructor and Destructor Functions ###

#### create ####

##### Description #####

Creates a new `timeseries` instance.

##### Function #####

    create(
        data_type regtype,
        period integer,
        epoch timestamp,
        extent bigint,
        measure_mode measure_enum,
        rollover_mode rollover_enum,
        gapfill_mode gapfill_enum,
        interpolation_mode interpolate_enum,
        gap_value number,
        min_value number,
        max_value number
    )

##### Return Type #####

`timeseries`

##### Example #####

    INSERT INTO samples (key, ts) VALUES ('security_trading_price_per_minute', create(
        'double precision',
        60,
        '2015-03-01 00:00:00',
        1440,
        'GAUGE',
        'STOP',
        'DEFAULT_VALUE',
        'NONE',
        NULL,
        0,
        NULL
    ));

#### drop ####

##### Description #####

Drops a `timeseries` instance, releasing the resources and storage space it takes.

##### Function #####

    drop(timeseries)

##### Return Type #####

`timeseries`

##### Example #####

    SELECT drop(ts) FROM samples WHERE key = 'security_trading_price_per_minute';

#### materialize ####

##### Description #####

Creates a new `timeseries` instance from the result of transformation(s) on existing
ones.

##### Function #####

    materialize(timeseries_slice)

##### Return Type #####

`timeseries`

##### Example #####

    INSERT INTO samples (key, ts) VALUES ('security_trading_price_per_hour', materialize(
        resample(
            (SELECT select(ts) FROM samples WHERE key = 'security_trading_price_per_minute'), 3600
        )
    );

See `resample()` function below.


### Informational Functions ###

#### info ####

##### Description #####

Provides information on a `timeseries` instance as a table row showing its properties.

##### Function #####

    info(timeseries)

##### Return Type #####

RECORD

##### Example #####

    timeseries=# SELECT (info(ts)).* FROM samples WHERE key = 'security_trading_price_per_minute';
        data_type     | period |        epoch        | extent | measure_mode | rollover_mode | gapfill_mode  | interpolation_mode | gap_value | min_value | max_value | last_update 
    ------------------+--------+---------------------+--------+--------------+---------------+---------------+--------------------+-----------+-----------+-----------+-------------
     double precision |     60 | 2015-03-01 00:00:00 |   1440 | GAUGE        | STOP          | DEFAULT_VALUE | NONE               |           |         0 |           | 
    (1 row)


### Selection and Transformation Functions ###

TBD


Author
------

Stephen J. Scheck <singularsyntax@gmail.com>

Copyright and License
---------------------

Copyright (c) 2015, Stephen J. Scheck. All rights reserved.

This extension is released under the PostgreSQL License, a liberal Open Source
license, similar to the BSD or MIT licenses. See the LICENSE file included in
the distribution for further information.
