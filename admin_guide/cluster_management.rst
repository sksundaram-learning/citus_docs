.. _cluster_management:

Cluster Management
$$$$$$$$$$$$$$$$$$

In this section, we discuss how you can add or remove nodes from your Citus cluster and how you can deal with node failures.

.. note::
  To make moving shards across nodes or re-replicating shards on failed nodes easier, Citus Enterprise comes with a shard rebalancer extension. We discuss briefly about the functions provided by the shard rebalancer as and when relevant in the sections below. You can learn more about these functions, their arguments and usage, in the :ref:`cluster_management_functions` reference section.

.. _scaling_out_cluster:

Scaling out your cluster
########################

Citus’s logical sharding based architecture allows you to scale out your cluster without any down time. This section describes how you can add more nodes to your Citus cluster in order to improve query performance / scalability.

.. _adding_worker_node:

Adding a worker
----------------------

Citus stores all the data for distributed tables on the worker nodes. Hence, if you want to scale out your cluster by adding more computing power, you can do so by adding a worker.

To add a new node to the cluster, you first need to add the DNS name or IP address of that node and port (on which PostgreSQL is running) in the pg_dist_node catalog table. You can do so using the :ref:`master_add_node` UDF. Example:

::

   SELECT * from master_add_node('node-name', 5432);

In addition to the above, if you want to move existing shards to the newly added worker, Citus Enterprise provides an additional rebalance_table_shards function to make this easier. This function will move the shards of the given table to make them evenly distributed among the workers.

::

	select rebalance_table_shards('github_events');


Adding a master
----------------------

The Citus master only stores metadata about the table shards and does not store any data. This means that all the computation is pushed down to the workers and the master does only final aggregations on the result of the workers. Therefore, it is not very likely that the master becomes a bottleneck for read performance. Also, it is easy to boost up the master by shifting to a more powerful machine.

However, in some write heavy use cases where the master becomes a performance bottleneck, users can add another master. As the metadata tables are small (typically a few MBs in size), it is possible to copy over the metadata onto another node and sync it regularly. Once this is done, users can send their queries to any master and scale out performance. If your setup requires you to use multiple masters, please `contact us <https://www.citusdata.com/about/contact_us>`_.

.. _dealing_with_node_failures:

Dealing With Node Failures
##########################

In this sub-section, we discuss how you can deal with node failures without incurring any downtime on your Citus cluster. We first discuss how Citus handles worker failures automatically by maintaining multiple replicas of the data. We also briefly describe how users can replicate their shards to bring them to the desired replication factor in case a node is down for a long time. Lastly, we discuss how you can setup redundancy and failure handling mechanisms for the master.

.. _worker_node_failures:

Worker Node Failures
--------------------

Citus can easily tolerate worker node failures because of its logical sharding-based architecture. While loading data, Citus allows you to specify the replication factor to provide desired availability for your data. In face of worker node failures, Citus automatically switches to these replicas to serve your queries. It also issues warnings like below on the master so that users can take note of node failures and take actions accordingly.

::

    WARNING:  could not connect to node localhost:9700

On seeing such warnings, the first step would be to log into the failed node and
inspect the cause of the failure.

In the meanwhile, Citus will automatically re-route the work to the healthy workers. Also, if Citus is not able to connect to a worker, it will assign that task to another node having a copy of that shard. If the failure occurs mid-query, Citus does not re-run the whole query but assigns only the failed query fragments leading to faster responses in face of failures.

Once the node is brought back up, Citus will automatically continue connecting to it and
using the data. 

If the node suffers a permanent failure then you may wish to retain the same
level of replication so that your application can tolerate more failures. To
make this simpler, Citus enterprise provides a replicate_table_shards UDF which
can be called after. This function copies the shards of a table across the
healthy nodes so they all reach the configured replication factor.

To remove a permanently failed node from the list of workers, you should first
mark all shard placements on that node as invalid (if they are not already so)
using the following query:

::

   UPDATE pg_dist_shard_placement set shardstate = 3 where nodename = 'bad-node-name' and nodeport = 5432;

Then, you can remove the node using master_remove_node, as shown below:

::
   
   select master_remove_node('bad-node-name', 5432);

If you want to add a new node to the cluster to replace the
failed node, you can follow the instructions described in the
:ref:`adding_worker_node` section. Finally, you can use the function provided in
Citus Enterprise to replicate the invalid shards and maintain the replication factor.

::

    select replicate_table_shards('github_events');


.. _master_node_failures:

Master Node Failures
--------------------

The Citus master maintains metadata tables to track all of the cluster nodes and the locations of the database shards on those nodes. The metadata tables are small (typically a few MBs in size) and do not change very often. This means that they can be replicated and quickly restored if the node ever experiences a failure. There are several options on how users can deal with master failures.

1. **Use PostgreSQL streaming replication:** You can use PostgreSQL's streaming
replication feature to create a hot standby of the master. Then, if the primary
master node fails, the standby can be promoted to the primary automatically to
serve queries to your cluster. For details on setting this up, please refer to the `PostgreSQL wiki <https://wiki.postgresql.org/wiki/Streaming_Replication>`_.

2. Since the metadata tables are small, users can use EBS volumes, or `PostgreSQL
backup tools <http://www.postgresql.org/docs/9.6/static/backup.html>`_ to backup the metadata. Then, they can easily
copy over that metadata to new nodes to resume operation.

3. Citus's metadata tables are simple and mostly contain text columns which
are easy to understand. So, in case there is no failure handling mechanism in
place for the master node, users can dynamically reconstruct this metadata from
shard information available on the worker nodes. To learn more about the metadata
tables and their schema, you can visit the :ref:`metadata_tables` section of our documentation.

