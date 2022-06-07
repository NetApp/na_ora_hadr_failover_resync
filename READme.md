na_ora_hadr_failover_resync
=========

This toolkit is developed to automate the tasks of setting up and managing a HR/DR environment for Oracle database deployed in AWS public cloud with FSx storage and EC2 compute instance. The toolkit delivers the following key functions:

- Setup HA/Dr target host: kernal configuration, Oracle configuration to match up with source server host

- Setup ONTAP: cluster peering, vserver peering, Oracle volumes snapmirror setup from Primary to DR

- Backup Oracle database data via snapshot: execute off crontab

- Backup Oracle database archive log via snapshot: execute off crontab

- Run failover and recovery on HA/DR host: test and validate HA/DR 

- Run resync after failover test: re-establish database volumes snapmirror in HA/DR mode

License
-------

By accessing, downloading, installing or using the content in this repository, you agree the terms of the License laid out in [License](LICENSE.TXT) file.

Note that there are certain restrictions around producing and/or sharing any derivative works with the content in this repository. Please make sure you read the terms of the [License](LICENSE.TXT) before using the content. If you do not agree to all of the terms, do not access, download or use the content in this repository.

Requirements
------------

    Ansible v.2.10 and higher
    ONTAP collection 21.19.1
    Python 3
    Python libraries:
      netapp-lib
      xmltodict
      jmespath
  
    AWS FSx storage as is available
  
    AWS EC2 Instnace
      RHEL 7/8, Oracle Linux 7/8
      Network interfaces for NFS, public (internet) and optional management
      Existing Oracle environment on source, and the equivalent Linux operating system at the destination (DR or HA)
  
Global variables configuration
------------ 

The Ansible playbooks are variable driven. An example global variable file fsx_vars_example.yml is included to demonstrate typical configuration. Following are key considerations: 

    ONTAP - retrieve FSx storage parameters using AWS FSx console when both source and target FSx cluster are deployed.
      cluster name: source/destination
      cluster management IP: source/destination
      inter-cluster IP: source/destination
      vserver name: source/destination
      vserver management IP: source/destination
      NFS lifs: source/destination
      cluster credentials: fsxadmin and vsadmin pwd to be updated in roles/ontap_setup/defaults/main.yml file
  
    Oracle database volumes - they should have been created from AWS FSx console, volume naming should follow strictly with following standard: 
      Oracle binary: {{ host_name }}_bin, generally one lun/volume
      Oracle data: {{ host_name }}_data, can be multiple luns/volume, add additonal line for each additional lun/volume in variable such as {{ host_name }}_data_01, {{ host_name }}_data_02 ..., the code can handle multiple data lun/volume without change
      Oracle log: {{ host_name }}_log, can be multiple luns/volume, add additonal line for each additional lun/volume in variable such as {{ host_name }}_log_01, {{ host_name }}_log_02 ..., the code can handle multiple data lun/volume without change
  	  host_name: as defined in hosts file in root directory, the code is written to be specifically matched up with host name defined in host file.
      
    Linux and DB specific global variables - keep as it is.
      Enter redhat subscription if you have one, otherwise leave it black.
       
Host variables configuration
---------

Host variables are defined in host_vars directory named as {{ host_name }}.yml

    Oracle - define host specific variables when deploying Oracle in multiple hosts concurrently
      ansible_host: IP address of database server host
      log_archive_mode: enable archive log archiving (true) or not (false)
      oracle_sid: Oracle instance identifier
      pdb: default deployment of Oracle in containerized configuration, name pdb_name string and number of pdbs (Oracle allow 3 pdbs free of multitenant license fee)
      listener_port: Oracle listener port, default 1521
      memory_limit: set Oracle SGA size, normally up to 75% RAM
      host_datastores_nfs: combining of all Oracle volumes (binary, data, and log) as defined in global vars file. If multi luns/volumes, keep exactly the same number of luns/volumes in host_var file
    
    Linux:
      hugepages_nr: set hugepage for large DB with large SGA for performance
      swap_blocks: AWS EC2 instance specific, add swap space to EC2 instance. If swap exist, it will be ignored.
      
DB server host configuration
---------

    Server name - AWS EC2 instance use IP address as host naming by default. If you use different name in hosts file for Ansible, setup host naming resolution in /etc/hosts file for both source and target server
      127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
      ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
      172.30.15.96 db1
      172.30.15.107 db2
      
Playbook Execution - executed in sequence
---------

    Install Ansible controller prerequsites:
    
      ansible-playbook -i hosts requirements.yml -u ec2-user --private-key db1.pem -e @vars/fsx_vars.yml
      ansible-galaxy collection install -r collections/requirements.yml --force
      
    Setup HA/Dr target host:
     
      ansible-playbook -i hosts ora_dr_setup.yml -u ec2-user --private-key db2.pem -e @vars/fsx_vars.yml
      
    Setup ONTAP:
    
      ansible-playbook -i hosts ontap_setup.yml -u ec2-user --private-key db2.pem -e @vars/fsx_vars.yml
      
    Backup Oracle database data via snapshot:
      
      10 * * * * cd /home/admin/na_ora_hadr_failover_resync && /usr/bin/ansible-playbook -i hosts ora_replication_cg.yml -u ec2-user --private-key db1.pem -e @vars/fsx_vars.yml >> logs/snap_data_`date +"%Y-%m%d-%H%M%S"`.log 2>&1
    
    Backup Oracle database archive log via snapshot:
    
      0,20,30,40,50 * * * * cd /home/admin/na_ora_hadr_failover_resync && /usr/bin/ansible-playbook -i hosts ora_replication_logs.yml -u ec2-user --private-key db1.pem -e @vars/fsx_vars.yml >> logs/snap_log_`date +"%Y-%m%d-%H%M%S"`.log 2>&1
    
    Run failover and recovery on HA/DR host: test and validate HA/DR:
    
      ansible-playbook -i hosts ora_recovery.yml -u ec2-user --private-key db2.pem -e @vars/fsx_vars.yml
    
    Run resync after failover test: re-establish database volumes snapmirror in HA/DR mode
    
      ansible-playbook -i hosts ontap_ora_resync.yml -u ec2-user --private-key db2.pem -e @vars/fsx_vars.yml
          
Help
---------

If you need help with this solution and use cases, please join the [NetApp Solution Automation community support Slack channel](https://netapppub.slack.com/archives/C021R4WC0LC) and look for the solution-automation channel to post your questions or inquires.
      
