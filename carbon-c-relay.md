carbon-c-relay(1) -- graphite relay, aggregator and rewriter
============================================================
[![Build Status](https://travis-ci.org/artb1sh/carbon-c-relay.svg)](https://travis-ci.org/artb1sh/carbon-c-relay)

## SYNOPSIS

`carbon-c-relay` `-f` *config-file* `[ options` ... `]`

## DESCRIPTION

**carbon-c-relay** accepts, cleanses, matches, rewrites, forwards and
aggregates graphite metrics by listening for incoming connections and
relaying the messages to other servers defined in its configuration.
The core functionality is to route messages via flexible rules to the
desired destinations.

**carbon-c-relay** is a simple program that reads its routing information
from a file.  The command line arguments allow to set the location for
this file, as well as the amount of dispatchers (worker threads) to use
for reading the data from incoming connections and passing them onto the
right destination(s).  The route file supports two main constructs:
clusters and matches.  The first define groups of hosts data metrics can
be sent to, the latter define which metrics should be sent to which
cluster.  Aggregation rules are seen as matches.  Rewrites are actions
that directly affect the metric at the point in which they appear in the
configuration.

For every metric received by the relay, cleansing is performed.  The
following changes are performed before any match, aggregate or rewrite
rule sees the metric:

  - double dot elimination (necessary for correctly functioning
    consistent hash routing)
  - trailing/leading dot elimination
  - whitespace normalisation (this mostly affects output of the relay
    to other targets: metric, value and timestamp will be separated by
    a single space only, ever)
  - irregular char replacement with underscores (\_), currently
    irregular is defined as not being in `[0-9a-zA-Z-_:#]`, but can be
    overridden on the command line.
    Note that tags (when present and allowed) are not processed this
    way.

## OPTIONS

These options control the behaviour of **carbon-c-relay**.

  * `-v`:
    Print version string and exit.

  * `-d`:
    Enable debug mode, this prints statistics to stdout and prints extra
    messages about some situations encountered by the relay that normally
    would be too verbose to be enabled.  When combined with `-t`
    (test mode) this also prints stub routes and consistent-hash ring
    contents.

  * `-s`:
    Enable submission mode.  In this mode, internal statistics are not
    generated.  Instead, queue pressure and metrics drops are reported on
    stdout.  This mode is useful when used as submission relay which'
    job is just to forward to (a set of) main relays.  Statistics about
    the submission relays in this case are not needed, and could easily
    cause a non-desired flood of metrics e.g. when used on each and
    every host locally.

  * `-t`:
    Test mode.  This mode doesn't do any routing at all, but instead reads
    input from stdin and prints what actions would be taken given the loaded
    configuration.  This mode is very useful for testing relay routes for
    regular expression syntax etc.  It also allows to give insight on how
    routing is applied in complex configurations, for it shows rewrites and
    aggregates taking place as well.  When `-t` is repeated, the relay
    will only test the configuration for validity and exit immediately
    afterwards.  Any standard output is suppressed in this mode, making
    it ideal for start-scripts to test a (new) configuration.

  * `-f` *config-file*:
    Read configuration from *config-file*.  A configuration consists of
    clusters and routes.  See [CONFIGURATION SYNTAX](#configuration-syntax)
    for more information on the options and syntax of this file.

  * `-l` *log-file*:
    Use *log-file* for writing messages.  Without this option, the relay
    writes both to *stdout* and *stderr*.  When logging to file, all
    messages are prefixed with `MSG` when they were sent to *stdout*, and
    `ERR` when they were sent to *stderr*.

  * `-p` *port*:
    Listen for connections on port *port*.  The port number is used for
    both `TCP`, `UDP` and `UNIX sockets`.  In the latter case, the socket
    file contains the port number.  The port defaults to *2003*, which is
    also used by the original `carbon-cache.py`.  Note that this only
    applies to the defaults, when `listen` directives are in the config,
    this setting is ignored.

  * `-w` *workers*:
    Use *workers* number of threads.  The default number of workers is
    equal to the amount of detected CPU cores.  It makes sense to reduce
    this number on many-core machines, or when the traffic is low.

  * `-b` *batchsize*:
    Set the amount of metrics that sent to remote servers at once to
    *batchsize*.  When the relay sends metrics to servers, it will
    retrieve `batchsize` metrics from the pending queue of metrics waiting
    for that server and send those one by one.  The size of the batch will
    have minimal impact on sending performance, but it controls the amount
    of lock-contention on the queue.  The default is *2500*.

  * `-q` *queuesize*:
    Each server from the configuration where the relay will send metrics
    to, has a queue associated with it.  This queue allows for disruptions
    and bursts to be handled.  The size of this queue will be set to
    *queuesize* which allows for that amount of metrics to be stored in
    the queue before it overflows, and the relay starts dropping metrics.
    The larger the queue, more metrics can be absorbed, but also more
    memory will be used by the relay.  The default queue size is *25000*.

  * `-L` *stalls*:
    Sets the max mount of stalls to *stalls* before the relay starts
    dropping metrics for a server.  When a queue fills up, the relay uses
    a mechanism called stalling to signal the client (writing to the
    relay) of this event.  In particular when the client sends a large
    amount of metrics in very short time (burst), stalling can help to
    avoid dropping metrics, since the client just needs to slow down for a
    bit, which in many cases is possible (e.g. when catting a file with
    `nc`(1)).  However, this behaviour can also obstruct, artificially
    stalling writers which cannot stop that easily.  For this the stalls
    can be set from *0* to *15*, where each stall can take around 1 second
    on the client.  The default value is set to *4*, which is aimed at the
    occasional disruption scenario and max effort to not loose metrics
    with moderate slowing down of clients.

  * `-C` *CAcertpath*:
    Read CA certs (for use with TLS/SSL connections) from given path or
    file.  When not given, the default locations are used.  Strict
    verfication of the peer is performed, so when using self-signed
    certificates, be sure to include the CA cert in the default
    location, or provide the path to the cert using this option.

  * `-T` *timeout*:
    Specifies the IO timeout in milliseconds used for server connections.
    The default is *600* milliseconds, but may need increasing when WAN
    links are used for target servers.  A relatively low value for
    connection timeout allows the relay to quickly establish a server is
    unreachable, and as such failover strategies to kick in before the
    queue runs high.

  * `-c` *chars*:
    Defines the characters that are next to `[A-Za-z0-9]` allowed in
    metrics to *chars*.  Any character not in this list, is replaced by
    the relay with `_` (underscore).  The default list of allowed
    characters is *-_:#*.

  * `-m` *length*:
    Limits the metric names to be of at most *length* bytes long.  Any
    lines containing metric names larger than this will be discarded.

  * `-M` *length*
    Limits the input to lines of at most *length* bytes.  Any excess
    lines will be discarded.  Note that `-m` needs to be smaller than
    this value.

  * `-H` *hostname*:
    Override hostname determined by a call to `gethostname`(3) with
    *hostname*.  The hostname is used mainly in the statistics metrics
    `carbon.relays.<hostname>.<...>` sent by the relay.

  * `-B` *backlog*:
    Sets TCP connection listen backlog to *backlog* connections.  The
    default value is *32* but on servers which receive many concurrent
    connections, this setting likely needs to be increased to avoid
    connection refused errors on the clients.

  * `-U` *bufsize*:
    Sets the socket send/receive buffer sizes in bytes, for both TCP and UDP
    scenarios.  When unset, the OS default is used.  The maximum is also
    determined by the OS.  The sizes are set using setsockopt with the flags
    SO_RCVBUF and SO_SNDBUF.  Setting this size may be necessary for large
    volume scenarios, for which also `-B` might apply.  Checking the *Recv-Q*
    and the *receive errors* values from *netstat* gives a good hint
    about buffer usage.

  * `-E`:
    Disable disconnecting idle incoming connections.  By default the
    relay disconnects idle client connections after 10 minutes.  It does
    this to prevent resources clogging up when a faulty or malicious
    client keeps on opening connections without closing them.  It
    typically prevents running out of file descriptors.  For some
    scenarios, however, it is not desirable for idle connections to be
    disconnected, hence passing this flag will disable this behaviour.

  * `-D`:
    Deamonise into the background after startup.  This option requires
    `-l` and `-P` flags to be set as well.

  * `-P` *pidfile*:
    Write the pid of the relay process to a file called *pidfile*.  This
    is in particular useful when daemonised in combination with init
    managers.

  * `-O` *threshold*:
    The minimum number of rules to find before trying to optimise the
    ruleset.  The default is `50`, to disable the optimiser, use `-1`,
    to always run the optimiser use `0`.  The optimiser tries to group
    rules to avoid spending excessive time on matching expressions.

## CONFIGURATION SYNTAX

The config file supports the following syntax, where comments start with
a `#` character and can appear at any position on a line and suppress
input until the end of that line:

```
cluster <name>
    < <forward | any_of | failover> [useall] |
      <carbon_ch | fnv1a_ch | jump_fnv1a_ch> [replication <count>] [dynamic] >
        <host[:port][=instance] [proto <udp | tcp>]
                                [type linemode]
                                [transport <plain | gzip | lz4 | snappy>
                                           [ssl]]> ...
    ;

cluster <name>
    file [ip]
        </path/to/file> ...
    ;

match
        <* | expression ...>
    [validate <expression> else <log | drop>]
    send to <cluster ... | blackhole>
    [stop]
    ;

rewrite <expression>
    into <replacement>
    ;

aggregate
        <expression> ...
    every <interval> seconds
    expire after <expiration> seconds
    [timestamp at <start | middle | end> of bucket]
    compute <sum | count | max | min | average |
             median | percentile<%> | variance | stddev> write to
        <metric>
    [compute ...]
    [send to <cluster ...>]
    [stop]
    ;

send statistics to <cluster ...>
    [stop]
    ;
statistics
    [submit every <interval> seconds]
    [reset counters after interval]
    [prefix with <prefix>]
    [send to <cluster ...>]
    [stop]
    ;

listen
    type linemode [transport <plain | gzip | lz4 | snappy> [ssl <pemcert>]]
        <<interface[:port] | port> proto <udp | tcp>> ...
        </ptah/to/file proto unix> ...
    ;

include </path/to/file/or/glob>
    ;
```

### CLUSTERS
Multiple clusters can be defined, and need not to be referenced by a
match rule.   All clusters point to one or more hosts, except the `file`
cluster which writes to files in the local filesystem.  `host` may be an
IPv4 or IPv6 address, or a hostname.  Since host is followed by an
optional `:` and port, for IPv6 addresses not to be interpreted wrongly,
either a port must be given, or the IPv6 address surrounded by brackets,
e.g. `[::1]`.  Optional `transport` and `proto` clauses can be used to
wrap the connection in a compression or encryption later or specify the
use of UDP or TCP to connect to the remote server.  When omitted the
connection defaults to an unwrapped TCP connection.  `type` can only be
linemode at the moment.

DNS hostnames are resolved to a single address, according to the preference
rules in [RFC 3484](https://www.ietf.org/rfc/rfc3484.txt).  The
`any_of`, `failover` and `forward` clusters have an explicit `useall`
flag that enables expansion for hostnames resolving to multiple
addresses.  Using this option Each address returned becomes a cluster
destination.

There are two groups of cluster types, simple forwarding clusters and
consistent hashing clusters.

* `forward` and `file` clusters

  The `forward` and `file` clusters simply send everything they receive
  to the defined members (host addresses or files).  When a cluster has
  multiple members, all incoming metrics are sent to /all/ members,
  basically duplicating the input metric stream over all members.

* `any_of` cluster

  The `any_of` cluster is a small variant of the `forward` cluster, but
  instead of sending the input metrics to all defined members, it sends
  each incoming metric to only one of defined members.  The purpose of
  this is a load-balanced scenario where any of the members can receive
  any metric.  As `any_of` suggests, when any of the members become
  unreachable, the remaining available members will immediately receive
  the full input stream of metrics.  This specifically mean that when 4
  members are used, each will receive approximately 25% of the input
  metrics.  When one member becomes unavailable (e.g. network
  interruption, or a restart of the service), the remaining 3 members
  will each receive about 33% of the input.  When designing cluster
  capacity, one should take into account that in the most extreme case,
  the final remaining member will receive all input traffic.

  An `any_of` cluster can in particular be useful when the cluster
  points to other relays or caches.  When used with other relays, it
  effectively load-balances, and adapts immediately over inavailability
  of targets.  When used with caches, the behaviour of the `any_of`
  router to send the same metrics consistently to the same destination
  helps caches to have a high hitrate on their internal buffers for the
  same metrics (if they use them), but still allows for a
  rolling-restart of the caches when e.g. on the same machine.

* `failover` cluster

  The `failover` cluster is like the `any_of` cluster, but sticks to the
  order in which servers are defined.  This is to implement a pure
  failover scenario between servers.  All metrics are sent to at most 1
  member, so no hashing or balancing is taking place.  A `failover`
  cluster with two members will only send metrics to the second member
  if the first becomes unavailable.

* `carbon_ch` cluster

  The `carbon_ch` cluster sends the metrics to the member that is
  responsible according to the consistent hash algorithm, as used in the
  original carbon python relay, or multiple members if replication is
  set to more than 1.  When `dynamic` is set, failure of any of the
  servers does not result in metrics being dropped for that server, but
  instead the undeliverable metrics are sent to any other server in the
  cluster in order for the metrics not to get lost.  This is most useful
  when replication is 1.

  The calculation of the hashring, that defines the way in which metrics
  are distributed, is based on the server host (or IP address) and the
  optional instance of the member.  This means that using `carbon_ch`
  two targets on different ports but on the same host will map to the
  same hashkey, which means no distribution of metrics takes place.  The
  instance is used to remedy that situation.  An instance is appended to
  the memeber after the port, and separated by an equals sign, e.g.
  `127.0.0.1:2006=a` for instance `a`.

  Consistent hashes are consistent in the sense that removal of a member
  from the cluster should not result in a complete re-mapping of all
  metrics to members, but instead only add the metrics from the removed
  member to all remaining members, approximately each gets its fair
  share.  The other way around, when a member is added, each member
  should see a subset of its metrics now being addressed to the new
  member.  This is an important advantage over a normal hash, where each
  removal or addition of members (also via e.g. a change in their IP
  address or hostname) would cause a full re-mapping of all metrics over
  all available metrics.

* `fnv1a_ch` cluster

  The `fnv1a_ch` cluster is a identical in behaviour to `carbon_ch`, but
  it uses a different hash technique (FNV1a) which is faster but more
  importantly defined to get by the beforementioned limitation of
  `carbon_ch` to use both host and port from the members.  This is
  useful when multiple targets live on the same host just separated by
  port.

  Since the instance property is no longer necessary with `fnv1a_ch`
  this way, this cluster type uses it to completely override the string
  that the hashkey should be calculated off.  This allows for many
  things, including masquerading old IP addresses, but it basically can
  be used to make the hash key location agnostic of the (physical)
  location of that key.  For example, usage like
  `10.0.0.1:2003=4d79d13554fa1301476c1f9fe968b0ac` would allow to change
  port and/or ip address of the server that receives data for the
  instance key.  Obviously, this way migration of data can be dealt with
  much more conveniently.  Note that since the instance name is used as
  full hash input, instances as `a`, `b`, etc. will likely result in
  poor hash distribution, since their hashes have very little input.
  Consider using longer and mostly differing instance names such as
  random hashes for better hash distribution behaviour.

* `jump_fnv1a_ch` cluster

  The `jump_fnv1a_ch` cluster is also a consistent hash cluster like the
  previous two, but it does not take the member host, port or instance
  into account at all.  Whether this is useful to you depends on your
  scenario.  The jump hash has almost perfect balancing over the members
  defined in the cluster, at the expense of not being able to remove any
  member but the last in order as defined in the cluster.  What this
  means is that this hash is fine to use with ever growing clusters
  where older nodes are never removed.

  If you have a cluster where removal of old nodes takes place often,
  the jump hash is not suitable for you.  Jump hash works with servers
  in an ordered list without gaps.  To influence the ordering, the
  instance given to the server will be used as sorting key.  Without,
  the order will be as given in the file.  It is a good practice to fix
  the order of the servers with instances such that it is explicit what
  the right nodes for the jump hash are.

### MATCHES
Match rules are the way to direct incoming metrics to one or more
clusters.  Match rules are processed top to bottom as they are defined
in the file.  It is possible to define multiple matches in the same
rule.  Each match rule can send data to one or more clusters.  Since
match rules "fall through" unless the `stop` keyword is added,
carefully crafted match expression can be used to target
multiple clusters or aggregations.  This ability allows to replicate
metrics, as well as send certain metrics to alternative clusters with
careful ordering and usage of the `stop` keyword.  The special cluster
`blackhole` discards any metrics sent to it.  This can be useful for
weeding out unwanted metrics in certain cases.  Because throwing metrics
away is pointless if other matches would accept the same data, a match
with as destination the blackhole cluster, has an implicit `stop`.
The `validation` clause adds a check to the data (what comes after the
metric) in the form of a regular expression.  When this expression
matches, the match rule will execute as if no validation clause was
present.  However, if it fails, the match rule is aborted, and no
metrics will be sent to destinations, this is the `drop` behaviour.
When `log` is used, the metric is logged to stderr.  Care should be
taken with the latter to avoid log flooding.  When a validate clause is
present, destinations need not to be present, this allows for applying a
global validation rule.  Note that the cleansing rules are applied
before validation is done, thus the data will not have duplicate spaces.
The `route using` clause is used to perform a temporary modification to
the key used for input to the consistent hashing routines.  The primary
purpose is to route traffic so that appropriate data is sent to the
needed aggregation instances.

### REWRITES
Rewrite rules take a regular expression as input to match incoming
metrics, and transform them into the desired new metric name.  In the
replacement,
backreferences are allowed to match capture groups defined in the input
regular expression.  A match of `server\.(x|y|z)\.` allows to use e.g.
`role.\1.` in the substitution.  A few caveats apply to the current
implementation of rewrite rules.  First, their location in the config
file determines when the rewrite is performed.  The rewrite is done
in-place, as such a match rule before the rewrite would match the
original name, a match rule after the rewrite no longer matches the
original name.  Care should be taken with the ordering, as multiple
rewrite rules in succession can take place, e.g. `a` gets replaced by
`b` and `b` gets replaced by `c` in a succeeding rewrite rule.  The
second caveat with the current implementation, is that the rewritten
metric names are not cleansed, like newly incoming metrics are.  Thus,
double dots and potential dangerous characters can appear if the
replacement string is crafted to produce them.  It is the responsibility
of the writer to make sure the metrics are clean.  If this is an issue
for routing, one can consider to have a rewrite-only instance that
forwards all metrics to another instance that will do the routing.
Obviously the second instance will cleanse the metrics as they come in.
The backreference notation allows to lowercase and uppercase the
replacement string with the use of the underscore (`_`) and carret
(`^`) symbols following directly after the backslash.  For example,
`role.\_1.` as substitution will lowercase the contents of `\1`.  The
dot (`.`) can be used in a similar fashion, or followed after the
underscore or caret to replace dots with underscores in the
substitution.  This can be handy for some situations where metrics are
sent to graphite.

### AGGREGATIONS
The aggregations defined take one or more input metrics expressed by one
or more regular expresions, similar to the match rules.  Incoming
metrics are aggregated over a period of time defined by the interval in
seconds.  Since events may arrive a bit later in time, the expiration
time in seconds defines when the aggregations should be considered
final, as no new entries are allowed to be added any more.  On top of an
aggregation multiple aggregations can be computed.  They can be of the
same or different aggregation types, but should write to a unique new
metric.  The metric names can include back references like in rewrite
expressions, allowing for powerful single aggregation rules that yield
in many aggregations.  When no `send to` clause is given, produced
metrics are sent to the relay as if they were submitted from the
outside, hence match and aggregation rules apply to those.  Care should
be taken that loops are avoided this way.  For this reason, the use of
the `send to` clause is encouraged, to direct the output traffic where
possible.  Like for match rules, it is possible to define multiple
cluster targets.  Also, like match rules, the `stop` keyword applies to
control the flow of metrics in the matching process.

### STATISTICS
The `send statistics to` construct is deprecated and will be removed in
the next release.  Use the special `statistics` construct instead.

The `statistics` construct can control a couple of things about the
(internal) statistics produced by the relay.  The `send to` target can
be used to avoid router loops by sending the statistics to a certain
destination cluster(s).  By default the metrics are prefixed with
`carbon.relays.<hostname>`, where hostname is determinted on startup and
can be overridden using the `-H` argument.  This prefix can be set using
the `prefix with` clause similar to a rewrite rule target.  The input
match in this case is the pre-set regular expression
`^(([^.]+)(\..*)?)$` on the hostname.  As such, one can see that the
default prefix is set by `carbon.relays.\.1`.  Note that this uses the
replace-dot-with-underscore replacement feature from rewrite rules.
Given the input expression, the following match groups are available:
`\1` the entire hostname, `\2` the short hostname and `\3` the domainname
(with leading dot).  It may make sense to replace the default by
something like `carbon.relays.\_2` for certain scenarios, to always use
the lowercased short hostname, which following the expression doesn't
contain a dot.  By default, the metrics are submitted every 60 seconds,
this can be changed using the `submit every <interval> seconds` clause.  
To obtain a more compatible set of values to carbon-cache.py, use the
`reset counters after interval` clause to make values non-cumulative,
that is, they will report the change compared to the previous value.

### LISTENERS
The ports and protocols the relay should listen for incoming connections
can be specified using the `listen` directive.  Currently, all listeners
need to be of `linemode` type.  An optional compression or encryption
wrapping can be specified for the port and optional interface given by
ip address, or unix socket by file.  When interface is not specified,
the any interface on all available ip protocols is assumed.  If no
`listen` directive is present, the relay will use the default listeners
for port 2003 on tcp and udp, plus the unix socket
`/tmp/.s.carbon-c-relay.2003`.  This typically expands to 5 listeners on
an IPv6 enabled system.  The default matches the behaviour of versions
prior to v3.2.

### INCLUDES
In case configuration becomes very long, or is managed better in
separate files, the `include` directive can be used to read another
file.  The given file will be read in place and added to the router
configuration at the time of inclusion.  The end result is one big route
configuration.  Multiple `include` statements can be used throughout the
configuration file.  The positioning will influence the order of rules
as normal.  Beware that recursive inclusion (`include` from an included
file) is supported, and currently no safeguards exist for an inclusion
loop.  For what is worth, this feature likely is best used with simple
configuration files (e.g. not having `include` in them).

## EXAMPLES

**carbon-c-relay** evolved over time, growing features on demand as the
tool proved to be stable and fitting the job well.  Below follow some
annotated examples of constructs that can be used with the relay.

Clusters can be defined as much as necessary.  They receive data from
match rules, and their type defines which members of the cluster finally
get the metric data.  The simplest cluster form is a `forward` cluster:

```
cluster send-through
    forward
        10.1.0.1
    ;
```

Any metric sent to the `send-through` cluster would simply be forwarded to
the server at IPv4 address `10.1.0.1`.  If we define multiple servers,
all of those servers would get the same metric, thus:

```
cluster send-through
    forward
        10.1.0.1
        10.2.0.1
    ;
```

The above results in a duplication of metrics send to both machines.
This can be useful, but most of the time it is not.  The `any_of`
cluster type is like `forward`, but it sends each incoming metric to any
of the members.  The same example with such cluster would be:

```
cluster send-to-any-one
    any_of 10.1.0.1:2010 10.1.0.1:2011;
```

This would implement a multipath scenario, where two servers are used,
the load between them is spread, but should any of them fail, all
metrics are sent to the remaining one.  This typically works well for
upstream relays, or for balancing carbon-cache processes running on the
same machine.  Should any member become unavailable, for instance due to
a rolling restart, the other members receive the traffic.  If it is
necessary to have true fail-over, where the secondary server is only
used if the first is down, the following would implement that:

```
cluster try-first-then-second
    failover 10.1.0.1:2010 10.1.0.1:2011;
```

These types are different from the two consistent hash cluster types:

```
cluster graphite
    carbon_ch
        127.0.0.1:2006=a
        127.0.0.1:2007=b
        127.0.0.1:2008=c
    ;
```

If a member in this example fails, all metrics that would go to that
member are kept in the queue, waiting for the member to return.  This
is useful for clusters of carbon-cache machines where it is desirable
that the same
metric ends up on the same server always.  The `carbon_ch` cluster type
is compatible with carbon-relay consistent hash, and can be used for
existing clusters populated by carbon-relay.  For new clusters, however,
it is better to use the `fnv1a_ch` cluster type, for it is faster, and
allows to balance over the same address but different ports without an
instance number, in constrast to `carbon_ch`.

Because we can use multiple clusters, we can also replicate without the
use of the `forward` cluster type, in a more intelligent way:

```
cluster dc-old
    carbon_ch replication 2
        10.1.0.1
        10.1.0.2
        10.1.0.3
    ;
cluster dc-new1
    fnv1a_ch replication 2
        10.2.0.1
        10.2.0.2
        10.2.0.3
    ;
cluster dc-new2
    fnv1a_ch replication 2
        10.3.0.1
        10.3.0.2
        10.3.0.3
    ;

match *
    send to dc-old
    ;
match *
    send to
        dc-new1
        dc-new2
    stop
    ;
```

In this example all incoming metrics are first sent to `dc-old`, then
`dc-new1` and finally to `dc-new2`.  Note that the cluster type of
`dc-old` is different.  Each incoming metric will be send to 2 members
of all three clusters, thus replicating to in total 6 destinations.  For
each cluster the destination members are computed independently.
Failure of clusters or members does not affect the others, since all
have individual queues.  The above example could also be written using
three match rules for each dc, or one match rule for all three dcs.  The
difference is mainly in performance, the number of times the incoming
metric has to be matched against an expression.  The `stop` rule in
`dc-new` match rule is not strictly necessary in this example, because
there are no more following match rules.  However, if the match would
target a specific subset, e.g.  `^sys\.`, and more clusters would be
defined, this could be necessary, as for instance in the following
abbreviated example:

```
cluster dc1-sys ... ;
cluster dc2-sys ... ;

cluster dc1-misc ... ;
cluster dc2-misc ... ;

match ^sys\. send to dc1-sys;
match ^sys\. send to dc2-sys stop;

match * send to dc1-misc;
match * send to dc2-misc stop;
```

As can be seen, without the `stop` in dc2-sys' match rule, all metrics
starting with `sys.` would also be send to dc1-misc and dc2-misc.  It
can be that this is desired, of course, but in this example there is a
dedicated cluster for the `sys` metrics.

Suppose there would be some unwanted metric that unfortunately is
generated, let's assume some bad/old software.  We don't want to store
this metric.  The `blackhole` cluster is suitable for that, when it is
harder to actually whitelist all wanted metrics.  Consider the
following:

```
match
        some_legacy1$
        some_legacy2$
    send to blackhole
    stop;
```

This would throw away all metrics that end with `some_legacy`, that
would otherwise be hard to filter out.  Since the order matters, it
can be used in a construct like this:

```
cluster old ... ;
cluster new ... ;

match * send to old;

match unwanted send to blackhole stop;

match * send to new;
```

In this example the old cluster would receive the metric that's unwanted
for the new cluster.  So, the order in which the rules occur does
matter for the execution.

Validation can be used to ensure the data for metrics is as expected.  A
global validation for just number (no floating point) values could be:

```
match *
    validate ^[0-9]+\ [0-9]+$ else drop
    ;
```

(Note the escape with backslash `\` of the space, you might be able to
use `\s` or `[:space:]` instead, this depends on your configured regex
implementation.)

The validation clause can exist on every match rule, so in principle,
the following is valid:

```
match ^foo
    validate ^[0-9]+\ [0-9]+$ else drop
    send to integer-cluster
    ;
match ^foo
    validate ^[0-9.e+-]+\ [0-9.e+-]+$ else drop
    send to float-cluster
    stop;
```

Note that the behaviour is different in the previous two examples.  When
no `send to` clusters are specified, a validation error makes the match
behave like the `stop` keyword is present.  Likewise, when validation
passes, processing continues with the next rule.
When destination clusters are present, the `match` respects the `stop`
keyword as normal.  When specified, processing will always stop when
specified so.  However, if validation fails, the rule does not send
anything to the destination clusters, the metric will be dropped or
logged, but never sent.

The relay is capable of rewriting incoming metrics on the fly.  This
process is done based on regular expressions with capture groups that
allow to substitute parts in a replacement string.  Rewrite rules allow
to cleanup metrics from applications, or provide a migration path.  In
it's simplest form a rewrite rule looks like this:

```
rewrite ^server\.(.+)\.(.+)\.([a-zA-Z]+)([0-9]+)
    into server.\_1.\2.\3.\3\4
    ;
```

In this example a metric like `server.DC.role.name123` would be
transformed into `server.dc.role.name.name123`.
For rewrite rules hold the same as for matches, that their order
matters.  Hence to build on top of the old/new cluster example done
earlier, the following would store the original metric name in the old
cluster, and the new metric name in the new cluster:

```
match * send to old;

rewrite ... ;

match * send to new;
```

Note that after the rewrite, the original metric name is no longer
available, as the rewrite happens in-place.

Aggregations are probably the most complex part of carbon-c-relay.  Two
ways of specifying aggregates are supported by carbon-c-relay.  The
first, static rules, are handled by an optimiser which tries to fold
thousands of rules into groups to make the matching more efficient.  The
second, dynamic rules, are very powerful compact definitions with
possibly thousands of internal instantiations.  A typical static
aggregation looks like:

```
aggregate
        ^sys\.dc1\.somehost-[0-9]+\.somecluster\.mysql\.replication_delay
        ^sys\.dc2\.somehost-[0-9]+\.somecluster\.mysql\.replication_delay
    every 10 seconds
    expire after 35 seconds
    timestamp at end of bucket
    compute sum write to
        mysql.somecluster.total_replication_delay
    compute average write to
        mysql.somecluster.average_replication_delay
    compute max write to
        mysql.somecluster.max_replication_delay
    compute count write to
        mysql.somecluster.replication_delay_metric_count
    ;
```

In this example, four aggregations are produced from the incoming
matching metrics.  In this example we could have written the two matches
as one, but for demonstration purposes we did not.  Obviously they can
refer to different metrics, if that makes sense.  The `every 10 seconds`
clause specifies in what interval the aggregator can expect new metrics
to arrive.  This interval is used to produce the aggregations, thus each
10 seconds 4 new metrics are generated from the data received sofar.
Because data may be in transit for some reason, or generation stalled,
the `expire after` clause specifies how long the data should be kept
before considering a data bucket (which is aggregated) to be complete.
In the example, 35 was used, which means after 35 seconds the first
aggregates are produced.  It also means that metrics can arrive 35
seconds late, and still be taken into account.  The exact time at which
the aggregate metrics are produced is random between 0 and interval (10
in this case) seconds after the expiry time.  This is done to prevent
thundering herds of metrics for large aggregation sets.
The `timestamp` that is used for the aggregations can be specified to be
the `start`, `middle` or `end` of the bucket.  Original
carbon-aggregator.py uses `start`, while carbon-c-relay's default has
always been `end`.
The `compute` clauses demonstrate a single aggregation rule can produce
multiple aggregates, as often is the case.  Internally, this comes for
free, since all possible aggregates are always calculated, whether or
not they are used.  The produced new metrics are resubmitted to the
relay, hence matches defined before in the configuration can match
output of the aggregator.  It is important to avoid loops, that can be
generated this way.  In general, splitting aggregations to their own
carbon-c-relay instance, such that it is easy to forward the produced
metrics to another relay instance is a good practice.

The previous example could also be written as follows to be dynamic:

```
aggregate
        ^sys\.dc[0-9].(somehost-[0-9]+)\.([^.]+)\.mysql\.replication_delay
    every 10 seconds
    expire after 35 seconds
    compute sum write to
        mysql.host.\1.replication_delay
    compute sum write to
        mysql.host.all.replication_delay
    compute sum write to
        mysql.cluster.\2.replication_delay
    compute sum write to
        mysql.cluster.all.replication_delay
    ;
```

Here a single match, results in four aggregations, each of a different
scope.  In this example aggregation based on hostname and cluster are
being made, as well as the more general `all` targets, which in this
example have both identical values.  Note that with this single
aggregation rule, both per-cluster, per-host and total aggregations are
produced.  Obviously, the input metrics define which hosts and clusters
are produced.

With use of the `send to` clause, aggregations can be made more
intuitive and less error-prone.  Consider the below example:

```
cluster graphite fnv1a_ch ip1 ip2 ip3;

aggregate ^sys\.somemetric
    every 60 seconds
    expire after 75 seconds
    compute sum write to
        sys.somemetric
    send to graphite
    stop
    ;

match * send to graphite;
```

It sends all incoming metrics to the graphite cluster, except the
sys.somemetric ones, which it replaces with a sum of all the incoming
ones.  Without a `stop` in the aggregate, this causes a loop, and
without the `send to`, the metric name can't be kept its original name,
for the output now directly goes to the cluster.


When configuring cluster you might want to check how the metrics will be routed and hashed. That's what the `-t` flag is for. For the following configuration : 
```
cluster graphite_swarm_odd
    fnv1a_ch replication 1
        host01.dom:2003=31F7A65E315586AC198BD798B6629CE4903D089947
        host03.dom:2003=9124E29E0C92EB63B3834C1403BD2632AA7508B740
        host05.dom:2003=B653412CD96B13C797658D2C48D952AEC3EB667313
;

cluster graphite_swarm_even
    fnv1a_ch replication 1
        host02.dom:2003=31F7A65E315586AC198BD798B6629CE4903D089947
        host04.dom:2003=9124E29E0C92EB63B3834C1403BD2632AA7508B740
        host06.dom:2003=B653412CD96B13C797658D2C48D952AEC3EB667313

;

match *
    send to
        graphite_swarm_odd
        graphite_swarm_even
    stop
;
```
Running the command : `echo "my.super.metric" | carbon-c-relay -f config.conf  -t`, will result in : 
```
[...]
match
    * -> my.super.metric
    fnv1a_ch(graphite_swarm_odd)
        host03.dom:2003
    fnv1a_ch(graphite_swarm_even)
        host04.dom:2003
    stop

```
You now know that your metric `my.super.metric` will be hashed and arrive on the host03 and host04 machines.
Adding the `-d` flag will increase the amount of information by showing you the hashring

## STATISTICS

When **carbon-c-relay** is run without `-d` or `-s` arguments,
statistics will be produced.  By default they are sent to the relay
itself in the form of `carbon.relays.<hostname>.*`.  See the
`statistics` construct to override this prefix, sending interval and
values produced.  While many metrics have a similar name to what
carbon-cache.py would produce, their values are likely different.  By
default, most values are running counters which only increase over time.
The use of the nonNegativeDerivative() function from graphite is useful
with these.

The following metrics are produced under the `carbon.relays.<hostname>`
namespace:

* metricsReceived

  The number of metrics that were received by the relay.  Received here
  means that they were seen and processed by any of the dispatchers.

* metricsSent

  The number of metrics that were sent from the relay.  This is a total
  count for all servers combined.  When incoming metrics are duplicated
  by the cluster configuration, this counter will include all those
  duplications.  In other words, the amount of metrics that were
  successfully sent to other systems.  Note that metrics that are
  processed (received) but still in the sending queue (queued) are not
  included in this counter.

* metricsDiscarded

  The number of input lines that were not considered to be a valid
  metric.  Such lines can be empty, only containing whitespace, or
  hitting the limits given for max input length and/or max metric length
  (see `-m` and `-M` options).

* metricsQueued

  The total number of metrics that are currently in the queues for all
  the server targets.  This metric is not cumulative, for it is a sample
  of the queue size, which can (and should) go up and down.  Therefore
  you should not use the derivative function for this metric.

* metricsDropped

  The total number of metric that had to be dropped due to server queues
  overflowing.  A queue typically overflows when the server it tries to
  send its metrics to is not reachable, or too slow in ingesting the
  amount of metrics queued.  This can be network or resource related,
  and also greatly depends on the rate of metrics being sent to the
  particular server.

* metricsBlackholed

  The number of metrics that did not match any rule, or matched a rule
  with blackhole as target.  Depending on your configuration, a high
  value might be an indication of a misconfiguration somewhere.  These
  metrics were received by the relay, but never sent anywhere, thus they
  disappeared.

* metricStalls

  The number of times the relay had to stall a client to indicate that
  the downstream server cannot handle the stream of metrics.  A stall is
  only performed when the queue is full and the server is actually
  receptive of metrics, but just too slow at the moment.  Stalls
  typically happen during micro-bursts, where the client typically is
  unaware that it should stop sending more data, while it is able to.

* connections

  The number of connect requests handled.  This is an ever increasing
  number just counting how many connections were accepted.

* disconnects

  The number of disconnected clients.  A disconnect either happens
  because the client goes away, or due to an idle timeout in the relay.
  The difference between this metric and connections is the amount of
  connections actively held by the relay.  In normal situations this
  amount remains within reasonable bounds.  Many connections, but few
  disconnections typically indicate a possible connection leak in the
  client.  The idle connections disconnect in the relay here is to guard
  against resource drain in such scenarios.

* dispatch\_wallTime\_us

  The number of microseconds spent by the dispatchers to do their work.
  In particular on multi-core systems, this value can be confusing,
  however, it indicates how long the dispatchers were doing work
  handling clients.  It includes everything they do, from reading data
  from a socket, cleaning up the input metric, to adding the metric to
  the appropriate queues.  The larger the configuration, and more
  complex in terms of matches, the more time the dispatchers will spend
  on the cpu.  But also time they do /not/ spend on the cpu is included
  in this number.  It is the pure wallclock time the dispatcher was
  serving a client.

* dispatch\_sleepTime\_us

  The number of microseconds spent by the dispatchers sleeping waiting
  for work.  When this value gets small (or even zero) the dispatcher
  has so much work that it doesn't sleep any more, and likely can't
  process the work in a timely fashion any more.  This value plus the
  wallTime from above sort of sums up to the total uptime taken by this
  dispatcher.  Therefore, expressing the wallTime as percentage of this
  sum gives the busyness percentage draining all the way up to 100% if
  sleepTime goes to 0.

* server\_wallTime\_us

  The number of microseconds spent by the servers to send the metrics
  from their queues.  This value includes connection creation, reading
  from the queue, and sending metrics over the network.

* dispatcherX

  For each indivual dispatcher, the metrics received and blackholed plus
  the wall clock time.  The values are as described above.

* destinations.X

  For all known destinations, the number of dropped, queued and sent
  metrics plus the wall clock time spent.  The values are as described
  above.

* aggregators.metricsReceived

  The number of metrics that were matched an aggregator rule and were
  accepted by the aggregator.  When a metric matches multiple
  aggregators, this value will reflect that.  A metric is not counted
  when it is considered syntactically invalid, e.g. no value was found.

* aggregators.metricsDropped

  The number of metrics that were sent to an aggregator, but did not fit
  timewise.  This is either because the metric was too far in the past
  or future.  The expire after clause in aggregate statements controls
  how long in the past metric values are accepted.

* aggregators.metricsSent

  The number of metrics that were sent from the aggregators.  These
  metrics were produced and are the actual results of aggregations.

## BUGS

Please report them at:
<https://github.com/grobian/carbon-c-relay/issues>

## AUTHOR

Fabian Groffen &lt;grobian@gentoo.org&gt;

## SEE ALSO

All other utilities from the graphite stack.

This project aims to be a fast replacement of the original
[Carbon relay](http://graphite.readthedocs.org/en/1.0/carbon-daemons.html#carbon-relay-py).
**carbon-c-relay** aims to deliver performance and configurability.
Carbon is single threaded, and sending metrics to multiple
consistent-hash clusters requires chaining of relays.  This project
provides a multithreaded relay which can address multiple targets and
clusters for each and every metric based on pattern matches.

There are a couple more replacement projects out there, which
are [carbon-relay-ng](https://github.com/graphite-ng/carbon-relay-ng) and [graphite-relay](https://github.com/markchadwick/graphite-relay
).

Compared to carbon-relay-ng, this project does provide carbon's
consistent-hash routing.  graphite-relay, which does this, however
doesn't do metric-based matches to direct the traffic, which this
project does as well.  To date, carbon-c-relay can do aggregations,
failover targets and more.

## ACKNOWLEDGEMENTS

This program was originally developed for Booking.com, which approved
that the code was published and released as Open Source on GitHub, for
which the author would like to express his gratitude.
Development has continued since with the help of many contributors
suggesting features, reporting bugs, adding patches and more to make
carbon-c-relay into what it is today.
