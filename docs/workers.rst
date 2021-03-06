Scaling synapse via workers
===========================

Synapse has experimental support for splitting out functionality into
multiple separate python processes, helping greatly with scalability.  These
processes are called 'workers', and are (eventually) intended to scale
horizontally independently.

All of the below is highly experimental and subject to change as Synapse evolves,
but documenting it here to help folks needing highly scalable Synapses similar
to the one running matrix.org!

All processes continue to share the same database instance, and as such, workers
only work with postgres based synapse deployments (sharing a single sqlite
across multiple processes is a recipe for disaster, plus you should be using
postgres anyway if you care about scalability).

The workers communicate with the master synapse process via a synapse-specific
TCP protocol called 'replication' - analogous to MySQL or Postgres style
database replication; feeding a stream of relevant data to the workers so they
can be kept in sync with the main synapse process and database state.

Configuration
-------------

To make effective use of the workers, you will need to configure an HTTP
reverse-proxy such as nginx or haproxy, which will direct incoming requests to
the correct worker, or to the main synapse instance. Note that this includes
requests made to the federation port. The caveats regarding running a
reverse-proxy on the federation port still apply (see
https://github.com/matrix-org/synapse/blob/master/README.rst#reverse-proxying-the-federation-port).

To enable workers, you need to add a replication listener to the master synapse, e.g.::

    listeners:
      - port: 9092
        bind_address: '127.0.0.1'
        type: replication

Under **no circumstances** should this replication API listener be exposed to the
public internet; it currently implements no authentication whatsoever and is
unencrypted.

You then create a set of configs for the various worker processes.  These
should be worker configuration files, and should be stored in a dedicated
subdirectory, to allow synctl to manipulate them.

Each worker configuration file inherits the configuration of the main homeserver
configuration file.  You can then override configuration specific to that worker,
e.g. the HTTP listener that it provides (if any); logging configuration; etc.
You should minimise the number of overrides though to maintain a usable config.

You must specify the type of worker application (``worker_app``). The currently
available worker applications are listed below. You must also specify the
replication endpoint that it's talking to on the main synapse process
(``worker_replication_host`` and ``worker_replication_port``).

For instance::

    worker_app: synapse.app.synchrotron

    # The replication listener on the synapse to talk to.
    worker_replication_host: 127.0.0.1
    worker_replication_port: 9092

    worker_listeners:
     - type: http
       port: 8083
       resources:
         - names:
           - client

    worker_daemonize: True
    worker_pid_file: /home/matrix/synapse/synchrotron.pid
    worker_log_config: /home/matrix/synapse/config/synchrotron_log_config.yaml

...is a full configuration for a synchrotron worker instance, which will expose a
plain HTTP ``/sync`` endpoint on port 8083 separately from the ``/sync`` endpoint provided
by the main synapse.

Obviously you should configure your reverse-proxy to route the relevant
endpoints to the worker (``localhost:8083`` in the above example).

Finally, to actually run your worker-based synapse, you must pass synctl the -a
commandline option to tell it to operate on all the worker configurations found
in the given directory, e.g.::

    synctl -a $CONFIG/workers start

Currently one should always restart all workers when restarting or upgrading
synapse, unless you explicitly know it's safe not to.  For instance, restarting
synapse without restarting all the synchrotrons may result in broken typing
notifications.

To manipulate a specific worker, you pass the -w option to synctl::

    synctl -w $CONFIG/workers/synchrotron.yaml restart


Available worker applications
-----------------------------

``synapse.app.pusher``
~~~~~~~~~~~~~~~~~~~~~~

Handles sending push notifications to sygnal and email. Doesn't handle any
REST endpoints itself, but you should set ``start_pushers: False`` in the
shared configuration file to stop the main synapse sending these notifications.

Note this worker cannot be load-balanced: only one instance should be active.

``synapse.app.synchrotron``
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The synchrotron handles ``sync`` requests from clients. In particular, it can
handle REST endpoints matching the following regular expressions::

    ^/_matrix/client/(v2_alpha|r0)/sync$
    ^/_matrix/client/(api/v1|v2_alpha|r0)/events$
    ^/_matrix/client/(api/v1|r0)/initialSync$
    ^/_matrix/client/(api/v1|r0)/rooms/[^/]+/initialSync$

The above endpoints should all be routed to the synchrotron worker by the
reverse-proxy configuration.

It is possible to run multiple instances of the synchrotron to scale
horizontally. In this case the reverse-proxy should be configured to
load-balance across the instances, though it will be more efficient if all
requests from a particular user are routed to a single instance. Extracting
a userid from the access token is currently left as an exercise for the reader.

``synapse.app.appservice``
~~~~~~~~~~~~~~~~~~~~~~~~~~

Handles sending output traffic to Application Services. Doesn't handle any
REST endpoints itself, but you should set ``notify_appservices: False`` in the
shared configuration file to stop the main synapse sending these notifications.

Note this worker cannot be load-balanced: only one instance should be active.

``synapse.app.federation_reader``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Handles a subset of federation endpoints. In particular, it can handle REST
endpoints matching the following regular expressions::

    ^/_matrix/federation/v1/event/
    ^/_matrix/federation/v1/state/
    ^/_matrix/federation/v1/state_ids/
    ^/_matrix/federation/v1/backfill/
    ^/_matrix/federation/v1/get_missing_events/
    ^/_matrix/federation/v1/publicRooms

The above endpoints should all be routed to the federation_reader worker by the
reverse-proxy configuration.

``synapse.app.federation_sender``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Handles sending federation traffic to other servers. Doesn't handle any
REST endpoints itself, but you should set ``send_federation: False`` in the
shared configuration file to stop the main synapse sending this traffic.

Note this worker cannot be load-balanced: only one instance should be active.

``synapse.app.media_repository``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Handles the media repository. It can handle all endpoints starting with::

    /_matrix/media/

You should also set ``enable_media_repo: False`` in the shared configuration
file to stop the main synapse running background jobs related to managing the
media repository.

Note this worker cannot be load-balanced: only one instance should be active.

``synapse.app.client_reader``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Handles client API endpoints. It can handle REST endpoints matching the
following regular expressions::

    ^/_matrix/client/(api/v1|r0|unstable)/publicRooms$

``synapse.app.user_dir``
~~~~~~~~~~~~~~~~~~~~~~~~

Handles searches in the user directory. It can handle REST endpoints matching
the following regular expressions::

    ^/_matrix/client/(api/v1|r0|unstable)/user_directory/search$

``synapse.app.frontend_proxy``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Proxies some frequently-requested client endpoints to add caching and remove
load from the main synapse. It can handle REST endpoints matching the following
regular expressions::

    ^/_matrix/client/(api/v1|r0|unstable)/keys/upload

It will proxy any requests it cannot handle to the main synapse instance. It
must therefore be configured with the location of the main instance, via
the ``worker_main_http_uri`` setting in the frontend_proxy worker configuration
file. For example::

    worker_main_http_uri: http://127.0.0.1:8008
