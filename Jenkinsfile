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
                sqlplus -S /nolog <<-EOP
                    conn PDBADMIN/$CDB_PASS@//$CDB_HOST:$CDB_PORT/$PDB_NAME$DB_DOMAIN
                    SELECT sys_context('USERENV', 'DB_NAME') FROM dual;
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
            sqlplus -S /nolog <<-EOF
                conn SYS/$CDB_PASS@//$CDB_HOST:$CDB_PORT/$CDB_NAME$DB_DOMAIN AS SYSDBA
                alter pluggable database $PDB_NAME CLOSE IMMEDIATE;
                drop pluggable database $PDB_NAME including datafiles;
                show pdbs;
            EOF
            exit $?
            '''.stripIndent()
        }
    }
}