# Database Ops Quick Reference Guide

## PostgreSQL Connections
```bash
psql -h <host> -p 5432 -U <user> -d <database>
psql "$DATABASE_URL"
psql -c "select version();"
psql -c "\l"
psql -c "\dt"
psql -c "\du"
```

## PostgreSQL Inspection
```sql
select now();
select current_database();
select current_user;
select * from pg_stat_activity;
select * from pg_locks;
select * from pg_stat_replication;
select datname, pg_size_pretty(pg_database_size(datname)) from pg_database;
```

## PostgreSQL Backup and Restore
```bash
pg_dump -h <host> -U <user> -d <database> -f backup.sql
pg_dump -h <host> -U <user> -d <database> -Fc -f backup.dump
pg_restore -h <host> -U <user> -d <database> backup.dump
psql -h <host> -U <user> -d <database> < backup.sql
createdb -h <host> -U <user> <database>
dropdb -h <host> -U <user> <database>
```

## PostgreSQL Slow Queries and Locks
```sql
select pid, now() - query_start as age, state, query
from pg_stat_activity
where state <> 'idle'
order by age desc;

select blocked_locks.pid as blocked_pid,
       blocking_locks.pid as blocking_pid,
       blocked_activity.query as blocked_query,
       blocking_activity.query as blocking_query
from pg_catalog.pg_locks blocked_locks
join pg_catalog.pg_stat_activity blocked_activity on blocked_activity.pid = blocked_locks.pid
join pg_catalog.pg_locks blocking_locks
  on blocking_locks.locktype = blocked_locks.locktype
 and blocking_locks.database is not distinct from blocked_locks.database
 and blocking_locks.relation is not distinct from blocked_locks.relation
 and blocking_locks.pid != blocked_locks.pid
join pg_catalog.pg_stat_activity blocking_activity on blocking_activity.pid = blocking_locks.pid
where not blocked_locks.granted;
```

## PostgreSQL Maintenance
```sql
vacuum analyze;
reindex database <database>;
select pg_reload_conf();
select pg_terminate_backend(<pid>);
```

## MySQL Connections
```bash
mysql -h <host> -P 3306 -u <user> -p
mysql -h <host> -u <user> -p <database>
mysql -e "select version();"
mysql -e "show databases;"
mysql -e "show processlist;"
```

## MySQL Inspection
```sql
show databases;
show tables;
show full processlist;
show grants for current_user();
show variables like 'max_connections';
show status like 'Threads_connected';
show engine innodb status\G
```

## MySQL Backup and Restore
```bash
mysqldump -h <host> -u <user> -p <database> > backup.sql
mysqldump -h <host> -u <user> -p --single-transaction <database> > backup.sql
mysql -h <host> -u <user> -p <database> < backup.sql
mysqladmin -h <host> -u <user> -p create <database>
mysqladmin -h <host> -u <user> -p drop <database>
```

## Kubernetes Database Access
```bash
kubectl get pods -A | grep postgres
kubectl get pods -A | grep mysql
kubectl port-forward svc/<database-service> 5432:5432
kubectl port-forward svc/<database-service> 3306:3306
kubectl exec -it <pod> -- psql -U <user> -d <database>
kubectl exec -it <pod> -- mysql -u <user> -p <database>
```

## AWS RDS
```bash
aws rds describe-db-instances
aws rds describe-db-clusters
aws rds describe-db-snapshots
aws rds create-db-snapshot --db-instance-identifier <db> --db-snapshot-identifier <snapshot>
aws rds describe-events --source-type db-instance --source-identifier <db>
aws rds describe-db-log-files --db-instance-identifier <db>
aws rds download-db-log-file-portion --db-instance-identifier <db> --log-file-name <log>
```

## Common Triage
```bash
nc -vz <host> 5432
nc -vz <host> 3306
dig <host>
df -h
free -h
top
```
