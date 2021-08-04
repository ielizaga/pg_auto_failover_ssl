# Setting up pg_auto_failover primary, secondary and monitor nodes with cert authentication in VMware Postgres 

## Pre-requirements

1. For a similar setup to the one described in this runbook, you will need three servers running RHEL7/CenOS7 (at the moment of writing, VMware Postgres is supported only on RHEL7 and CentOS7. These servers should be reachable from one another

   1. Server 1. Will be used as the pg_auto_failover monitor, using the hostname `autofailover-monitor`

     1. Server 2. Will be used as the pg_auto_failover first postgres node, using the hostname `autofailover-1`
     2. Server 3. Will be used as the pg_auto_failover second postgres node, using the hostname `autofailover-2`

  2. Add /etc/hosts entries in all three servers to be able to resolve hostnames.

     ```bash
     echo "x.x.x.x  autofailover-monitor" >> /etc/hosts
     echo "y.y.y.y  autofailover-1" >> /etc/hosts
     echo "z.z.z.z  autofailover-1" >> /etc/hosts
     ```

3. Download VMware Postgres server binaries in the virtual three servers from [Tanzu Network](https://network.pivotal.io/products/pivotal-postgres/#/releases/884740)

4. Install VMware Postgres in all servers following the instructions in the [official Documentation](https://postgres.docs.pivotal.io/13-3/installing.html). Do not run initdb, this will be taken care of by pg_auto_failover later

   ```bash
   yum install vmware-postgres-11.7.1.el7.x86_64.rpm
   ```

## Autofailover-m (pg_auto_failover monitor)

  1. Create a key-pair for the root CA in the postgres home dir. Normally this would be provided by your IT organization but for testing purposes we will generate one

     ```bash
     su - postgres
     openssl req -new -x509 -days 365 -nodes -out ~postgres/root.crt -keyout ~postgres/root.key -subj "/CN=root-ca"
     ```

  2. Create the server certificate signing request (CSR) and key. The CN for the server key-pair has to match its hostname, in this example `autofailover-monitor` for the monitor node. In this step, the server key-pair for the other two servers and the postgresql client key-pair are also created for the purposes of time, but this could have been done at a later stage and it is not needed for the creation of the monitor itself

     ```bash
     openssl req -new -nodes -out server.csr -keyout server.key -subj "/CN=autofailover-monitor"
     openssl req -new -nodes -out autofailover1.csr -keyout autofailover1.key -subj "/CN=autofailover-1"
     openssl req -new -nodes -out autofailover2.csr -keyout autofailover2.key -subj "/CN=autofailover-2"
     openssl req -new -nodes -out postgresql.csr -keyout postgresql.key -subj "/CN=postgres"
     ```

  3. Sign the server csrs with the CA

     ```bash
     openssl x509 -req -in server.csr -days 365 -CA root.crt -CAkey root.key -CAcreateserial -out server.crt
     openssl x509 -req -in autofailover1.csr -days 365 -CA root.crt -CAkey root.key -CAcreateserial -out autofailover1.crt
     openssl x509 -req -in autofailover2.csr -days 365 -CA root.crt -CAkey root.key -CAcreateserial -out autofailover2.crt
     openssl x509 -req -in postgresql.csr -days 365 -CA root.crt -CAkey root.key -CAcreateserial -out postgresql.crt
     
     # optional
     rm server.csr autofailover1.csr autofailover2.csr postgresql.csr
     ```

  4. Ensure permissions are 600 or pg_autoctl may fail to create the nodes

     ```bash
     chmod og-rwx root.key root.crt server.crt server.key autofailover1.crt autofailover1.key autofailover2.crt autofailover2.key postgresql.crt postgresql.key
     ```

  5. Create the .postgres directory where the client certificate will be placed and move a copy of the postgresql files and root certificate to it.
     Note: If not done, `pg_autoctl show uri`  will later on fail

     ```bash
     mkdir ~postgres/.postgresql
     cp postgresql.crt postgresql.key root.crt ~postgres/.postgresql
     ```

  6. Initialize the monitor with pg_autoctl

     ```bash
     pg_autoctl create monitor --ssl-ca-file root.crt --server-cert server.crt --server-key server.key --skip-pg-hba --pgdata ~postgres/monitor --ssl-mode verify-full
     ```

  7. Add the service to systemd and start it

     ```bash
     pg_autoctl -q show systemd --pgdata ~postgres/monitor > pgautofailover.service
     sudo mv pgautofailover.service /etc/systemd/system
     sudo systemctl daemon-reload
     sudo systemctl enable pgautofailover
     sudo systemctl start pgautofailover
     ```

  8. Edit the `pg_ident.conf` file and add a user name map for pgautofailover

     ```bash
     echo "pgautofailover   postgres               autoctl_node" >> ~postgres/monitor/pg_ident.conf
     ```

  9. Edit the `pg_hba.conf` file and add a rule for the pgautofailover map

     ```bash
     echo "hostssl pg_auto_failover autoctl_node 0.0.0.0/0 cert map=pgautofailover" >> ~postgres/monitor/pg_hba.conf
     ```

  10. Restart service

     ```bash
     sudo systemctl restart pgautofailover
     ```

## Set up postgres node (autofailover-1)

  1. Transfer the autofailover1 server certificate and key, the root certificate and the postgresql client certificate and key from autofailover-monitor to autofailover-1

     ```bash
     scp postgresql.crt postgresql.key autofailover1.crt autofailover1.key root.crt root@autofailover-1:/var/lib/pgsql
     ```

  2. In autofailover-1, create the .postgres directory where the client certificate will be placed and move the postgresql files and root certificate to it.
     Note: This is required by pg_auto_failover when auth=cert is used, as described in the [documentation](https://pg-auto-failover.readthedocs.io/en/master/security.html#ssl-certificates-authentication)

     ````bash
     # if you scp as root in (2), remember to change ownership of the certificates and keys to postgres
     sudo chown postgres:postgres autofailover1.crt autofailover1.key postgresql.crt postgresql.key root.crt
     
     su - postgres
     mkdir ~postgres/.postgresql
     mv postgresql.crt postgresql.key root.crt ~postgres/.postgresql
     ````

  3. Initialize the postgres node  with pg_autoctl

     ```bash
     pg_autoctl create postgres --ssl-ca-file ~postgres/.postgresql/root.crt --server-cert ~postgres/autofailover1.crt --server-key ~postgres/autofailover1.key --ssl-mode verify-full --pgdata ha --skip-pg-hba --username ha-admin --dbname appdb --hostname autofailover-1 --pgctl /bin/pg_ctl --monitor 'postgres://autoctl_node@autofailover-monitor/pg_auto_failover?sslmode=verify-full'
     ```

  4. Add the service to systemd and start it

     ```bash
     pg_autoctl -q show systemd --pgdata ~postgres/ha > pgautofailover.service
     sudo mv pgautofailover.service /etc/systemd/system
     sudo systemctl daemon-reload
     sudo systemctl enable pgautofailover
     sudo systemctl start pgautofailover
     ```

  5. Edit the `pg_ident.conf` file to allow the `postgres` user in the client certificate to connect as the `pgautofailover_replicator` database user

     ```bash
     echo "pgautofailover  postgres                pgautofailover_replicator" >> ~postgres/ha/pg_ident.conf
     ```

  6. Edit the `pg_hba.conf` file and add a rule for the pgautofailover replicator map

     ```bash
     echo "hostssl all pgautofailover_monitor 0.0.0.0/0 trust" >> ~postgres/ha/pg_hba.conf
     echo "hostssl replication pgautofailover_replicator 0.0.0.0/0 cert map=pgautofailover" >> ~postgres/ha/pg_hba.conf
     echo "hostssl postgres pgautofailover_replicator 0.0.0.0/0 cert map=pgautofailover" >> ~postgres/ha/pg_hba.conf
     echo "hostssl appdb pgautofailover_replicator 0.0.0.0/0 cert map=pgautofailover" >> ~postgres/ha/pg_hba.conf
     ```

## Set up postgres node (autofailover-2)

  1. Transfer the autofailover2 server certificate and key, the root certificate and the postgresql client certificate and key from autofailover-monitor to autofailover-2

     ```bash
     scp postgresql.crt postgresql.key autofailover2.crt autofailover2.key root.crt root@autofailover-2:/var/lib/pgsql
     ```

  2. In autofailover-2, create the .postgres directory where the client certificate will be placed and move the postgresql files and root certificate to it.
     Note: This is required by pg_auto_failover when auth=cert is used, as described in the [documentation](https://pg-auto-failover.readthedocs.io/en/master/security.html#ssl-certificates-authentication)

     ````bash
     # if you scp as root in (2), remember to change ownership of the certificates and keys to postgres
     sudo chown postgres:postgres autofailover2.crt autofailover2.key postgresql.crt postgresql.key root.crt
     
     su - postgres
     mkdir ~postgres/.postgresql
     mv postgresql.crt postgresql.key root.crt ~postgres/.postgresql
     ````

  3. Initialize the postgres node  with pg_autoctl

     ```bash
     pg_autoctl create postgres --ssl-ca-file ~postgres/.postgresql/root.crt --server-cert ~postgres/autofailover2.crt --server-key ~postgres/autofailover2.key --ssl-mode verify-full --pgdata ha --skip-pg-hba --username ha-admin --dbname appdb --hostname autofailover-2 --pgctl /bin/pg_ctl --monitor 'postgres://autoctl_node@autofailover-monitor/pg_auto_failover?sslmode=verify-full'
     ```

  4. Add the service to systemd and start it

     ```bash
     pg_autoctl -q show systemd --pgdata ~postgres/ha > pgautofailover.service
     sudo mv pgautofailover.service /etc/systemd/system
     sudo systemctl daemon-reload
     sudo systemctl enable pgautofailover
     sudo systemctl start pgautofailover
     ```

  5. Edit the `pg_ident.conf` file to allow the `postgres` user in the client certificate to connect as the `pgautofailover_replicator` database user

     ```bash
     echo "pgautofailover  postgres                pgautofailover_replicator" >> ~postgres/ha/pg_ident.conf
     ```

  6. Edit the `pg_hba.conf` file and add a rule for the pgautofailover replicator map

     ```bash
     echo "hostssl all pgautofailover_monitor 0.0.0.0/0 trust" >> ~postgres/ha/pg_hba.conf
     echo "hostssl replication pgautofailover_replicator 0.0.0.0/0 cert map=pgautofailover" >> ~postgres/ha/pg_hba.conf
     echo "hostssl postgres pgautofailover_replicator 0.0.0.0/0 cert map=pgautofailover" >> ~postgres/ha/pg_hba.conf
     echo "hostssl appdb pgautofailover_replicator 0.0.0.0/0 cert map=pgautofailover" >> ~postgres/ha/pg_hba.conf
     ```

  ## References
  - [Secure TCP/IP Connections with SSL](https://www.postgresql.org/docs/current/ssl-tcp.html#SSL-CERTIFICATE-CREATION)
  - [Security settings for pg_auto_failover](https://pg-auto-failover.readthedocs.io/en/master/security.html#ssl-certificates-authentication)
