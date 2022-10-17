pipeline {
    agent { 
      docker { 
        image 'ghcr.io/oracle/oraclelinux8-instantclient:21'
      }
    }

    environment {
        // Global Variables
        CDB_HOST  = "${CDB_HOST}"
        CDB_NAME  = "${CDB_NAME}"
        DB_DOMAIN = "${DB_DOMAIN}"

        // Project Secrets
        CDB_PASS  = credentials('CDB_PASS')

        // Build Specific - Indy PDB per executor to avoid clashes
        PDB_NAME  = "P${env.EXECUTOR_NUMBER}"
    }

    stages {
        stage('Create-PDB') {
            steps {
                sh '''\
                #!/bin/env bash
                sqlplus -S /nolog <<-EOF
                    conn SYS/$CDB_PASS@//$CDB_HOST:$CDB_PORT/$CDB_NAME$DB_DOMAIN AS SYSDBA
                    WHENEVER SQLERROR EXIT FAILURE
                    create pluggable database $PDB_NAME ADMIN USER PDBADMIN 
                        IDENTIFIED BY "$CDB_PASS" 
                        ROLES = (dba,resource);
                    alter pluggable database $PDB_NAME OPEN;
                    alter session set container=$PDB_NAME;
                    ADMINISTER KEY MANAGEMENT SET KEY FORCE KEYSTORE IDENTIFIED BY "$CDB_PASS" WITH BACKUP;
                    show pdbs;
                EOF
                echo "$PDB_NAME is Ready; testing resolution and connectivity"
                echo "$PDB_NAME = $PDB_CONN" > $TNS_ADMIN/tnsnames.ora
                sqlplus PDBADMIN/$CDB_PASS@$PDB_NAME <<-EOP
                    SELECT sys_context('USERENV', 'DB_NAME') FROM dual;
                    CREATE TABLESPACE PARSE DATAFILE SIZE 250M AUTOEXTEND ON NEXT 250M;
                    ALTER USER PDBADMIN DEFAULT TABLESPACE PARSE;
                    ALTER USER PDBADMIN QUOTA UNLIMITED ON PARSE;
                    ALTER TABLESPACE SYSAUX ADD DATAFILE SIZE 1G;
                    ALTER TABLESPACE UNDOTBS1 ADD DATAFILE SIZE 1G;
                EOP
                exit $?
                '''.stripIndent()
            }
        }
        stage('Run-DB-Test') {
            steps {
                sh '''\
                #!/bin/env bash
                sqlplus -S /nolog <<-EOF
                  conn PDBADMIN/$CDB_PASS@//$CDB_HOST:$CDB_PORT/$PDB_NAME$DB_DOMAIN
                  WHENEVER SQLERROR EXIT FAILURE
                  SELECT sys_context('USERENV', 'DB_NAME') FROM dual;
                EOF
                exit $?
                '''.stripIndent()
            }
        }
    }
    // Destroy-PDB
    post {
        always {
            sh '''\
            #!/bin/env bash
            echo "Dropping $PDB_NAME"
            sqlplus SYS/$CDB_PASS@$CDB_CONN AS SYSDBA <<-EOF
                alter pluggable database $PDB_NAME CLOSE IMMEDIATE;
                drop pluggable database $PDB_NAME including datafiles;
                show pdbs;
            EOF
            echo "$PDB_NAME Dropped"
            '''.stripIndent()
        }
    }
}