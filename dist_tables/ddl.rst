.. _ddl:

Creating Distributed Tables (DDL)
#################################

.. note::
    The instructions below assume that the PostgreSQL installation is in your path. If not, you will need to add it to your PATH environment variable. For example:

    ::

        export PATH=/usr/lib/postgresql/9.6/:$PATH

We use the github events dataset to illustrate the commands below. You can download that dataset by running:

::

    wget http://examples.citusdata.com/github_archive/github_events-2015-01-01-{0..5}.csv.gz
    gzip -d github_events-2015-01-01-*.gz

Creating And Distributing Tables
--------------------------------

To create a distributed table, you need to first define the table schema. To do so, you can define a table using the `CREATE TABLE <http://www.postgresql.org/docs/9.6/static/sql-createtable.html>`_ statement in the same way as you would do with a regular PostgreSQL table.

::

    psql -h localhost -d postgres
    CREATE TABLE github_events
    (
    	event_id bigint,
    	event_type text,
    	event_public boolean,
    	repo_id bigint,
    	payload jsonb,
    	repo jsonb,
    	actor jsonb,
    	org jsonb,
    	created_at timestamp
    );

Next, you can use the create_distributed_table() function to specify the table
distribution column and create the worker shards.

::

    SELECT create_distributed_table('github_events', 'repo_id');

This function informs Citus that the github_events table should be distributed
on the repo_id column (by hashing the column value). The function also creates
shards on the worker nodes using the citus.shard_count and
citus.shard_replication_factor configuration values.

This example would create a total of citus.shard_count number of shards where each
shard owns a portion of a hash token space and gets replicated based on the
default citus.shard_replication_factor configuration value. The shard replicas
created on the worker have the same table schema, index, and constraint
definitions as the table on the master. Once the replicas are created, this
function saves all distributed metadata on the master.

Each created shard is assigned a unique shard id and all its replicas have the same shard id. Each shard is represented on the worker node as a regular PostgreSQL table with name 'tablename_shardid' where tablename is the name of the distributed table and shardid is the unique id assigned to that shard. You can connect to the worker postgres instances to view or run commands on individual shards.

You are now ready to insert data into the distributed table and run queries on it. You can also learn more about the UDF used in this section in the :ref:`user_defined_functions` of our documentation.

.. _colocation_groups:

Co-Location Groups
------------------

By default two tables are co-located when they are distributed by columns of the same type using the same distribution method and over the same number of shards. Another way to say this is that the tables are in the same *co-location group.*

It is important for performance to have full control over co-location groups. All members of a co-location group support router-executed JOIN queries between one another on their distribution column. They also support foreign key constraints for keys containing the distribution column. Additionally shard rebalancing preserves co-location in new shard placements.

To control co-location groups manually use the optional :code:`colocate_with` parameter of :code:`create_distributed_table`. Left unspecified it defaults to the value :code:`default` which uses the behavior described above to lump all tables having the same distribution column type into the same co-location group. These statements are equivalent:

.. code-block:: sql

  SELECT create_distributed_table('github_events', 'repo_id');

  --
  -- is equivalent to
  --

  SELECT create_distributed_table('github_events', 'repo_id', colocate_with => 'default');


To start a new group and add tables to it use the two other modes of :code:`colocate_with`: the reserved string :code:`none` and the name of another table.

.. code-block:: sql

  -- start a new group
  SELECT create_distributed_table('products', 'store_id', colocate_with => 'none');

  -- add to the same group as products
  SELECT create_distributed_table('orders', 'store_id', colocate_with => 'products');

There are a few reasons to create new co-location groups:

* To express that tables distributed by same type of columns using the same shard count should *not* be related. This makes the rebalancer more efficient because it has more flexibility in placing shards.
* To scale out sets of tables independently, with different choices of shard count.
* To ensures co-location happens even in situations where a worker failure prevents shard placement and makes two tables seem to have different shard counts.
* To express data modeling intentions explicitly.

Dropping Tables
---------------

You can use the standard PostgreSQL DROP TABLE command to remove your distributed tables. As with regular tables, DROP TABLE removes any indexes, rules, triggers, and constraints that exist for the target table. In addition, it also drops the shards on the worker nodes and cleans up their metadata.

::

    DROP TABLE github_events;
