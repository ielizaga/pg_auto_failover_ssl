# Runbook: Setting up VMware Postgres with pg_auto_failover and cert authentication

## Pre-requirements

1. For a similar setup to the one described in this runbook, you will need three servers running RHEL7/CenOS7 (at the moment of writing, VMware Postgres is supported only on RHEL7 and CentOS7. These servers should be reachable from one another.

   1. Server 1. Will be used as the pg_auto_failover monitor, using the hostname `autofailover-monitor`

     1. Server 2. Will be used as the pg_auto_failover first postgres node, using the hostname `autofailover-1`
     2. Server 3. Will be used as the pg_auto_failover second postgres node, using the hostname `autofailover-2`

  2. Add /etc/hosts entries in all three servers to be able to resolve hostnames

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

  1. To make things faster and easier for this runbook, as root user, we will add postgres to the sudoers. Then we will switch to the postgres user.

     ```bash
     echo "postgres  ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/postgres
     su - postgres
     ```

  2. Create a key-pair for the root CA in the postgres home dir. Normally this would be provided by your IT organization but for testing purposes we will generate one.

     ```bash
     openssl req -new -x509 -days 365 -nodes -out ~postgres/root.crt -keyout ~postgres/root.key -subj "/CN=root-ca"
     ```

  3. Create the server certificate signing request (CSR) and key. The CN for the server key-pair should match the hostname, in this case `autofailover-m`. In this step we will also create the server certificates for the other two servers to save time and a postgres client certificate to be used by pg_autoctl when connecting to the monitor, but this could have been done at a later stage and it is not needed for the creation of the monitor itself. 

     ```bash
     openssl req -new -nodes -out server.csr -keyout server.key -subj "/CN=autofailover-m"
     openssl req -new -nodes -out autofailover1.csr -keyout autofailover1.key -subj "/CN=autofailover-1"
     openssl req -new -nodes -out autofailover2.csr -keyout autofailover2.key -subj "/CN=autofailover-2"
     openssl req -new -nodes -out postgresql.csr -keyout postgresql.key -subj "/CN=postgres"
     ```

  4. Sign the server csrs with the CA key and remove the signing requests as they are no longer needed

     ```bash
     openssl x509 -req -in server.csr -days 365 -CA root.crt -CAkey root.key -CAcreateserial -out server.crt
     openssl x509 -req -in autofailover1.csr -days 365 -CA root.crt -CAkey root.key -CAcreateserial -out autofailover1.crt
     openssl x509 -req -in autofailover2.csr -days 365 -CA root.crt -CAkey root.key -CAcreateserial -out autofailover1.crt
     openssl x509 -req -in postgresql.csr -days 365 -CA root.crt -CAkey root.key -CAcreateserial -out postgresql.crt
     rm server.csr autofailover1.csr autofailover2.csr postgresql.csr
     ```

  5. Ensure permissions are 600 or pg_autoctl may fail to create the monitor node

     ```bash
     chmod og-rwx root.key root.crt server.crt server.key autofailover1.crt autofailover1.key autofailover2.crt autofailover2.key postgresql.crt postgres.key
     ```

  6. Dir listing at this point:

     ```bash
     needs redo!
     ```

  7. Initialize the monitor. In this example the monitor will be placed under ~postgres/monitor. The server certificate, key and root CA certificate are passed to the pg_autoctl command as arguments and the auth type is set to cert.

     ```bash
     pg_autoctl create monitor --ssl-ca-file root.crt --server-cert server.crt --server-key server.key --auth=cert --pgdata ~postgres/monitor --ssl-mode verify-full
     ```

  4. Add the service to systemd to be able to start it easily

     ```bash
     pg_autoctl -q show systemd --pgdata ~postgres/monitor > pgautofailover.service
     sudo mv pgautofailover.service /etc/systemd/system
     sudo systemctl daemon-reload
     sudo systemctl enable pgautofailover
     sudo systemctl start pgautofailover
     ```

  5. Edit the monitor pg_ident.conf to add a user name map for pgautofailover:

     ```bash
     echo "pgautofailover   postgres               autoctl_node" >> ~postgres/monitor/pg_ident.conf
     ```

  6. Edit the monitor pg_hba.conf to add a rule for the pgautofailover map.

     ```bash
     echo "hostssl pg_auto_failover autoctl_node 0.0.0.0/0 cert map=pgautofailover" >> ~postgres/monitor/pg_hba.conf
     ```

## Set up postgres node (autofailover-1)

  1. To make things faster and easier for this runbook, as root user, we will add postgres to the sudoers. Then we will switch to the postgres user.

     ```bash
     echo "postgres  ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/postgres
     su - postgres
     ```

  2. Create the .postgres directory where the Client cert will be placed

     ````bash
     mkdir ~postgres/.postgres
     ````

  3. Need to clean up after this...

  4. SCP the autofailover-m ~postgres/root.crt ~postgres/autofailover1.crt and postgres/autofailover1.key into this server

  5. Create server certs and signed them with CA key

         1. `openssl req -new -nodes -out server.csr -keyout server.key -subj "/CN=autofailover-1"`
                2. scp server.csr over to monitor (had to do this to sign because of the -CAcreateserial but not sure if it can be done faster, I think so)
                       3. sign autofailover-1 server.csr with CA key in the monitor `openssl x509 -req -in server.csr -days 365 -CA ~postgres/root.crt -CAkey ~postgres/keys/ca.key -CAcreateserial -out server-autofailover-1.crt`
                              4. scp server-autofailover-1.crt back to autofailover-1:~postgres and rename to server.crt
                                     5. remove server.csr

  6. Create client cert to be able to initialize the db and connect to the monitor

         1. `cd ~postgres/.postgres`
                2. `openssl req -new -nodes -out postgresql.csr -keyout postgresql.key -subj /CN=postgres`
                       3. scp postgres.csr over to monitor (had to do this to sign because of the -CAcreateserial but not sure if it can be done faster, I think so)
                              4. sign autofailover-1 postgres.csr with CA key in the monitor `openssl x509 -req -in postgres.csr -days 365 -CA ~postgres/root.crt -CAkey ~postgres/keys/ca.key -CAcreateserial -out postgres-autofailover-1.crt`
                                     5. scp postgres-autofailover-1.crt back to autofailover-1:~postgres/.postgres and rename to postgres.crt

  7. Test connecting to the monitor to verify the client cert and SSL is working
     `psql postgres://autoctl_node@autofailover-m/pg_auto_failover?sslmode=verify-full`

  8. At this point I added an entry in /etc/hosts for autofailover-m that I then reference in the next step

  9. Initialize the node:

    pg_autoctl create postgres --ssl-ca-file ca.crt --server-cert server.crt --server-key server.key --ssl-mode verify-full --pgdata ha --auth=cert --username ha-admin --dbname appdb --hostname autofailover-1 --pgctl /bin/pg_ctl --monitor 'postgres://autoctl_node@autofailover-m/pg_auto_failover?sslmode=verify-full'

  References
  https://www.postgresql.org/docs/current/ssl-tcp.html#SSL-CERTIFICATE-CREATION
  https://pg-auto-failover.readthedocs.io/en/master/security.html#ssl-certificates-authentication  
