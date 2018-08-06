# Kafka Connect Oracle

kafka-connect-oracle is a Kafka source connector for capturing all row based DML changes and streaming these changes through Kafka. Change data capture logic is based on Oracle LogMiner solution.

Only committed changes are pulled from Oracle which are Insert,Update,Delete operations. All streamed messages have related full "sql_redo" statement and parsed fields with values of sql statements. Parsed fields and values are kept in proper field type in schemas.

Messages have old and new values of row fields for DML operations.Insert operation has only new values of row tagged as "data",update operation has new data tagged as "data" as insert operation and also contains old values tagged "before".Delete operation only contains old data tagged as "before".

# Setting Up

The database must be in archivelog mode and supplemental logging must be enabled.

On database server

    sqlplus / as sysdba    
    SQL>shutdown immediate
    SQL>startup mount
    SQL>alter database archivelog;
    SQL>alter database open;

Enable supplemental logging

    sqlplus / as sysdba    
    SQL>alter database add supplemental log data (all) columns;

In order to execute connector successfully, connector must be started with privileged Oracle user.If given user has DBA role , this step can be skipped else following scripts will be executed to create privileged user.

    create role logmnr_role;
    grant create session to logmnr_role;
    grant  execute_catalog_role,select any transaction ,select any dictionary to logmnr_role;
    create user kminer identified by kminerpass;
    grant  logmnr_role to kminer;
    alter user kminer quota unlimited on users;


# Configuration

## Configuration Properties

|Name|Type|Description|
|---|---|---|
|name|String|Connector name|
|connector.class|String|The name of the java class for this connector.|
|db.name.alias|String|Identifier name for database like Test,Dev,Prod or specific name to identify the database.This name will be used as header of topics and schema names.|
|tasks.max|Integer|Maximum number of tasks to create.This connector uses a single task.|
|topic|String|Name of the topic that the messages will be written to.If it is set a value all messages will be written into this declared topic , if it is not set,  for each database table a topic will be created dynamically.|
|db.name|String|Service name  or sid of the database to connect.Mostly database service name is used.|
|db.hostname|String|Ip address or hostname of Oracle database server.|
|db.port|Integer|Port number of Oracle database server.|
|db.user|String |Name of database user which is used to connect to database to start and execute logminer. This           user must provide necessary privileges mentioned above.|
|db.user.password|String|Password of database user.|
|db.fetch.size|Integer|This config property sets Oracle row fetch size value.|
|table.whitelist|String|A comma separated list of database schema or table names which will be captured.<br />For all schema capture **<SCHEMA_NAME>.*** <br /> For table capture **<SCHEMA_NAME>.<TABLE_NAME>** must be specified.|
|parse.dml.data|Boolean|If it is true , captured sql DML statement will be parsed into fields and values.If it is false only sql DML statement is published.
|reset.offset|Boolean|If it is true , offset value will be set to current SCN of database when connector started.If it is false connector will start from last offset value.
|||



## Example Config

    name=oracle-logminer-connector
    connector.class=com.ecer.kafka.connect.oracle.OracleSourceConnector
    db.name.alias=test
    tasks.max=1
    topic=cdctest
    db.name=testdb
    db.hostname=10.1.X.X
    db.port=1521
    db.user=kminer
    db.user.password=kminerpass
    db.fetch.size=1
    table.whitelist=TEST.*,TEST2.TABLE2
    parse.dml.data=true
    reset.offset=false

## Building and Running

    mvn clean package

    Copy kafka-connect-oracle-1.0.jar and lib/ojdbc7.jar to KAFKA_HOME/lib folder. If CONFLUENT platform is using for kafka cluster , copy ojdbc7.jar and  kafka-connect-oracle-1.0.jar to $CONFLUENT_HOME/share/java/kafka-connect-jdbc folder.

    Copy config/OracleSourceConnector.properties file to $KAFKA_HOME/config . For CONFLUENT copy properties file to $CONFLUENT_HOME/etc/kafka-connect-jdbc

    To start connector

    cd $KAFKA_HOME
    ./bin/connect-standalone.sh ./config/connect-standalone.properties ./config/OracleSourceConnector.properties

    In order to start connector with Avro serialization 
    
    cd $CONFLUENT_HOME

    ./bin/connect-standalone ./etc/schema-registry/connect-avro-standalone.properties ./etc/kafka-connect-jdbc/OracleSourceConnector.properties 

    Do not forget to populate connect-standalone.properties  or connect-avro-standalone.properties with the appropriate hostnames and ports.

# Todo:

    1.Implementation for DDL operations
    2.Support for other Oracle specific data types
    3.New implementations
    4.Performance Tuning
    5.Initial data load
    6.Bug fix