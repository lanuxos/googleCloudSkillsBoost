# Create and Manage AlloyDB Instances
## AlloyDB - Database Fundamentals [GSP1083]
### create a cluster and instance
Databases  > AlloyDB for PostgreSQL > Clusters > Create cluster > Highly Available > Continue
Cluster ID : lab-cluster
Password : Change3Me
Network : peering-network
Continue
machine type : 2 vCPU, 16GB
Create Cluster
### create tables and insert data in your database
Compute Engine > VM instances > alloydb-client > Connect [SSH]
export ALLOYDB=Instance_IP
- launch PostgreSQL
psql -h $ALLOYDB -U postgres
- create table
CREATE TABLE regions (
    region_id bigint NOT NULL,
    region_name varchar(25)
) ;

ALTER TABLE regions ADD PRIMARY KEY (region_id);
- insert records into table
INSERT INTO regions VALUES ( 1, 'Europe' );

INSERT INTO regions VALUES ( 2, 'Americas' );

INSERT INTO regions VALUES ( 3, 'Asia' );

INSERT INTO regions VALUES ( 4, 'Middle East and Africa' );
- query
SELECT region_id, region_name from regions;
\q
### use the google cloud cli with alloyDB
- create cluster
gcloud beta alloydb clusters create gcloud-lab-cluster --password=Change3Me --network=peering-network --region=us-east4 --project=qwiklabs-gcp-04-b396c7c152f1
- create instance
gcloud beta alloydb instances create gcloud-lab-instance --instance-type=PRIMARY --cpu-count=2 --region=us-east4  --cluster=gcloud-lab-cluster --project=qwiklabs-gcp-04-b396c7c152f1
- list instances
gcloud beta alloydb clusters list
### deleting a cluster
gcloud beta alloydb clusters delete gcloud-lab-cluster --force --region=us-east4 --project=qwiklabs-gcp-04-b396c7c152f1
gcloud beta alloydb clusters list

## Migrating to AlloyDB from PostgreSQL Using Database Migration Service [GSP1084]
student-00-eb12a87b4f91@qwiklabs.net
jLGMXHFz7ySX
qwiklabs-gcp-03-eef20b05f8a4
us-east4
us-east4-b
### verify data in the source instance for migration
sudo -u postgres psql
\dt
select count (*) as countries_row_count from countries;
select count (*) as departments_row_count from departments;
select count (*) as employees_row_count from employees;
select count (*) as jobs_row_count from jobs;
select count (*) as locations_row_count from locations;
select count (*) as regions_row_count from regions;
\q
exit
### create a database migration service connection profile for a stand-alone postgresql database
Database Migration > Connection profiles.
Create Profile
Source database engine : PostgreSQL
Connection profile : pg14-source
IP
Port : 5432
Username : postgres
Password : Change3Me 
Region : us-east4
Create
### create and start a continuous migration job
- Create a new continuous migration job
Database Migration > Migration jobs
Create Migration Job
name : postgres-to-alloydb
Source database engine : PostgreSQL
Destination database engine : AlloyDB PostgreSQL
Destination region : us-east4
Migration job type: Continuous
Save & Continue
- Define the source instance
Source connection profile : pg14-source
Save & Continue
- Create the destination instance
Cluster Type : Highly available
Continue
Cluster ID : alloydb-target-cluster
Password : Change3Me
Network, select peering-network
Continue
Instance ID : alloydb-target-instance
Select 2 vCPU, 16 GB as your machine type
Save & Continue
Create Destination & Continue to continue.
Connectivity method : VPC peering
Configure & Continue
Configure & Continue
- Test and start the continuous migration job

### confirm data load in the alloyDB for postgresql instance
- ssh into destination instance [alloyDB]
psql -h $ALLOYDB -U postgres
\dt

select count (*) as countries_row_count from countries;
select count (*) as departments_row_count from departments;
select count (*) as employees_row_count from employees;
select count (*) as jobs_row_count from jobs;
select count (*) as locations_row_count from locations;
select count (*) as regions_row_count from regions;

select region_id, region_name from regions;
### propagate a alive update to the alloyDB instance
- ssh into source instance and add record
sudo -u postgres psql
insert into regions values (5, 'Oceania');
select region_id, region_name from regions;
- check destination instance for changes
select region_id, region_name from regions;

## Migrating to Alloy DB from PostgreSQL Using PostgreSQL Tools [GSP1085]
### verify data in source instance for migration
- 
sudo -u postgres psql
\dt
select count (*) as countries_row_count from countries;
select count (*) as departments_row_count from departments;
select count (*) as employees_row_count from employees;
select count (*) as jobs_row_count from jobs;
select count (*) as locations_row_count from locations;
select count (*) as regions_row_count from regions;
\q
### create a database DMP file using pg_dump
- dump DMP file
sudo -u postgres pg_dump -Fc postgres > pg14_source.DMP
- upload DMP file to cloud storage
gsutil cp pg14_source.DMP gs://qwiklabs-gcp-03-c3b783548fa1/pg14_source.DMP
### import DMP file using pg_restore
- connect to destination alloyDB to check data table inside
export ALLOYDB=ALLOYDB_ADDRESS
echo $ALLOYDB  > alloydbip.txt 
psql -h $ALLOYDB -U postgres
\dt
- copy DMP source to instance
gsutil cp  gs://qwiklabs-gcp-03-c3b783548fa1/pg14_source.DMP pg14_source.DMP
- create TOC file that comments out all extension
pg_restore -l  pg14_source.DMP | sed -E 's/(.* EXTENSION )/; \1/g' >  pg14_source_toc.toc
- restore DMP to alloyDB
pg_restore -h $ALLOYDB -U postgres -d postgres -L pg14_source_toc.toc pg14_source.DMP
psql -h $ALLOYDB -U postgres
\dt
select count (*) as countries_row_count from countries;
select count (*) as departments_row_count from departments;
select count (*) as employees_row_count from employees;
select count (*) as jobs_row_count from jobs;
select count (*) as locations_row_count from locations;
select count (*) as regions_row_count from regions;

## Administering an AlloyDB Database [GSP1086]
### examine a database flag
- 
### setup a database extension
export ALLOYDB=ALLOYDB_ADDRESS
echo $ALLOYDB  > alloydbip.txt 
psql -h $ALLOYDB -U postgres
\c postgres
CREATE EXTENSION IF NOT EXISTS PGAUDIT;
select extname, extversion from pg_extension where extname = 'pgaudit';
\q
exit
### create a read pool instance for an existing cluster
AlloyDB > Add Read Pool Instance
Read pool instance ID: lab-instance-rp1
Node count : 2
2 vCPU, 16 GB
Create Read Pool
### setup backups
Databases > AlloyDB for PostgreSQL > Backups > Create backup
- more detail on backup
gcloud beta alloydb backups list
### examine monitoring in the alloyDB console
pgbench -h $ALLOYDB -U postgres -i -s 50 -F 90 -n postgres
pgbench -h $ALLOYDB -U postgres -c 50 -j 2 -P 30 -T 180 postgres

## Accelerating Analytical Queries using the AlloyDB Columnar Engine [GSP1087]
### create baseline dataset for testing the columnar engine
pgbench -h $ALLOYDB -U postgres -i -s 500 -F 90 -n postgres
psql -h $ALLOYDB -U postgres
select count (*) from pgbench_accounts;
### run a baseline test
\timing on
SELECT aid, bid, abalance FROM pgbench_accounts WHERE bid < 189  OR  abalance > 100 LIMIT 20;
EXPLAIN (ANALYZE,COSTS,SETTINGS,BUFFERS,TIMING,SUMMARY,WAL,VERBOSE)
SELECT count(*) FROM pgbench_accounts WHERE bid < 189  OR  abalance > 100;
### verify the database flag for the columnar engine
### set or verify a database extension for the columnar engine
\c postgres
\dx
CREATE EXTENSION IF NOT EXISTS google_columnar_engine;
SELECT google_columnar_engine_add('pgbench_accounts');
EXPLAIN (ANALYZE,COSTS,SETTINGS,BUFFERS,TIMING,SUMMARY,WAL,VERBOSE)
SELECT count(*) FROM pgbench_accounts WHERE bid < 189  OR  abalance > 100;
### testing the columnar engine
- 

## Create and Manage AlloyDB Instances: Challenge Lab [GSP395]
Challenge scenario
In your role as the corporate Database Administrator, you have been tasked with standing up a new AlloyDB for PostgreSQL database for your company's HR Operations group. You have been provided a list specifications for this database related to tables to create and data to load.
student-04-65a538539173@qwiklabs.net
CzSMcabqr4K4
qwiklabs-gcp-02-d22e09d45a28
us-west1
us-west1-c

### create a cluster and instance
- using cloud console to create a cluster and instance
Databases  > AlloyDB for PostgreSQL > Clusters > Create cluster > Highly Available > Continue
Cluster ID : lab-cluster
Password : Change3Me
Network : peering-network
Continue
machine type : 2 vCPU, 16GB
Create Cluster
IP 10.55.0.5
- using cli to create a cluster
gcloud beta alloydb clusters create lab-cluster --password=Change3Me --network=peering-network --region=us-west1 --project=qwiklabs-gcp-02-d22e09d45a28
- using cli to create an instance
gcloud beta alloydb instances create lab-instance --instance-type=PRIMARY --cpu-count=2 --region=us-west1 --cluster=lab-cluster --project=qwiklabs-gcp-02-d22e09d45a28
- cluster private IP
When you are on the Overview page for the new cluster you created, please make note of the Private IP address in the instances section. Copy the Private IP address to a text file so that you can paste the value in a later step
### create tables in your instance
Compute Engine
VM instances
alloydb-client
SSH
- set environment
export ALLOYDB=10.55.0.5
echo $ALLOYDB  > alloydbip.txt 
- launch psql
psql -h $ALLOYDB -U postgres
Change3Me
- create table
CREATE TABLE regions(
    region_id bigint NOT NULL,
    region_name varchar(25)
);
ALTER TABLE regions ADD PRIMARY KEY (region_id);
CREATE TABLE countries(
    countries_id char(2) NOT NULL,
    country_name varchar(40),
    region_id bigint
);
ALTER TABLE countries ADD PRIMARY KEY (countries_id);
CREATE TABLE departments(
    department_id smallint NOT NULL,
    department_name varchar(30),
    manager_id integer,
    location_id smallint
);
ALTER TABLE departments ADD PRIMARY KEY (department_id);
### load simple datasets into tables
INSERT INTO regions
VALUES
(1, 'Europe'),
(2, 'Americas'),
(3, 'Asia'),
(4, 'Middle East and Africa');
INSERT INTO countries
VALUES
('IT', 'Italy', 1),
('JP', 'Japan', 3),
('US', 'United States of America', 2),
('CA', 'Canada', 2),
('CN', 'China', 3),
('IN', 'India', 3),
('AU', 'Australia', 3),
('ZW', 'Zimbabwe', 4),
('SG', 'Singapore', 3);
INSERT INTO departments
VALUES
(10, 'Administration', 200, 1700),
(20, 'Marketing', 201, 1800),
(30, 'Purchasing', 114, 1700),
(40, 'Human Resources', 203, 2400),
(50, 'Shipping', 121, 1500),
(60, 'IT', 103, 1400);
### create a read pool instance
- using cloud console
Database > AlloyDB
Add Read Pool | Add Read Pool Instance [in your cluster section of the Overview]
Read pool instance ID = lab-instance-rp1
Node count = 2
2 vCPU, 16 GB as your machine type.
Create Read Pool
- using cli
gcloud beta alloydb instance create lab-instance-rp1 --instance-type=READ_POOL --cpu-count=2 --read-pool-node-count=2 --region=us-west1 --cluster=lab-cluster --project=qwiklabs-gcp-02-d22e09d45a28
### create a backup
- using cloud console
Databases 
AlloyDB for PostgreSQL 
Backups = lab-backup
- using cli
gcloud beta alloydb backups create lab-backup --cluster=lab-cluster --region=us-west1 --project=qwiklabs-gcp-02-d22e09d45a28
- checking backup list
gcloud beta alloydb backups list