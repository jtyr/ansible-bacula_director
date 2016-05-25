bacula_director
===============

Ansible role which helps to install and configure Bacula Director.

The configuration of the role is done in such way that it should not be
necessary to change the role for any kind of configuration. All can be
done either by changing role parameters or by declaring completely new
configuration as a variable. That makes this role absolutely
universal. See the examples below for more details.

Please report any issues or send PR.


Examples
--------

```
---

# Example of how to install Bacula on a single host
- name: Bacula all-in-one
  hosts: bacula-all
  vars:
    # Run client on the same host like the Bacula Director
    my_client: localhost.localdomain
    my_bacula_director_clients:
      - host: "{{ my_client }}"
        password: bacula
        file_retention: 30 days
        jobdefs: Default
    # Client
    bacula_client_config_filedaemon_options_name: "{{ my_client }}-fd"
    # Director
    bacula_director_config_schedule:
      - Schedule:
        - Name = Daily
        - Run = Full sun at 23:10
        - Run = Incremental mon-sat at 23:10
    bacula_director_config_fileset:
      - FileSet:
        - Name = Home
        - Include:
          - File = /home
        - Exclude:
          - File = /home/nobody
    bacula_director_config_jobdefs:
      - JobDefs:
        - Name = Default
        - Type = Backup
        - Level = Incremental
        - FileSet = Home
        - Messages = Standard
        - Pool = File
        - Storage = File
        - Schedule = Daily
    bacula_director_config_job:
      - template:
          - Job:
            - Name = Job-{[{ item['jobdefs'] }]}-{[{ item['host'] }]}
            - Client = {[{ item['host'] }]}-fd
            - JobDefs = {[{ item['jobdefs'] }]}
        items: "{{ my_bacula_director_clients }}"
    bacula_director_config_client:
      - template:
          - Client:
            - Name = {[{ item['host'] }]}-fd
            - Address = {[{ item['host'] }]}
            - FD Port = 9102
            - Catalog = Default
            - Password = {[{ item['password'] }]}
            - File Retention = {[{ item['file_retention'] }]}
            - Job Retention = 3 months
            - AutoPrune = yes
        items: "{{ my_bacula_director_clients }}"
  roles:
    - role: postgresql
      tags: postgresql
    - role: bacula_console
      tags: bacula_console
    - role: bacula_storage
      tags: bacula_storage
    - role: bacula_client
      tags: bacula_client
    - role: bacula_director
      tags: bacula_director

# The following plays are used for multi-node Bacula installation
- name: Bacula DB
  hosts: bacula-db
  vars:
    # Listen on all network interfaces
    postgresql_config_listen_addresses: 0.0.0.0
    # Configure access
    postgresql_hba_config__custom:
      # Allow access for the postgres user from the Bacula director host
      - type: host
        database: all
        user: postgres
        address: 192.168.50.201/32
        method: md5
      # Allow access for the bacula user to the bacula DB from the Bacula director host
      - type: host
        database: bacula
        user: bacula
        address: 192.168.50.201/32
        method: md5
  roles:
    - postgresql

- name: Bacula Console
  hosts: bacula-console
  vars:
    bacula_console_config_director_options_address: 192.168.50.201
  roles:
    - bacula_console

- name: Bacula Storage
  hosts: bacula-storage
  roles:
    - bacula_storage

- name: Bacula Client
  hosts: bacula-client
  roles:
    - bacula_client

- name: Bacula Director
  hosts: bacula-director
  vars:
    # Client runs on a different host now
    my_client: 192.168.50.204
    my_bacula_director_clients:
      - host: "{{ my_client }}"
        password: bacula
        file_retention: 30 days
        jobdefs: Default
    # Director
    bacula_director_config_storage_options_address: 192.168.50.202
    bacula_director_config_catalog_options_db_address: 192.168.50.205
    bacula_director_config_schedule:
      - Schedule:
        - Name = Daily
        - Run = Full sun at 23:10
        - Run = Incremental mon-sat at 23:10
    bacula_director_config_fileset:
      - FileSet:
        - Name = Home
        - Include:
          - File = /home
        - Exclude:
          - File = /home/nobody
    bacula_director_config_jobdefs:
      - JobDefs:
        - Name = Default
        - Type = Backup
        - Level = Incremental
        - FileSet = Home
        - Messages = Standard
        - Pool = File
        - Storage = File
        - Schedule = Daily
    bacula_director_config_job:
      - template:
          - Job:
            - Name = Job-{[{ item['jobdefs'] }]}-{[{ item['host'] }]}
            - Client = {[{ item['host'] }]}-fd
            - JobDefs = {[{ item['jobdefs'] }]}
        items: "{{ my_bacula_director_clients }}"
    bacula_director_config_client:
      - template:
          - Client:
            - Name = {[{ item['host'] }]}-fd
            - Address = {[{ item['host'] }]}
            - FD Port = 9102
            - Catalog = Default
            - Password = {[{ item['password'] }]}
            - File Retention = {[{ item['file_retention'] }]}
            - Job Retention = 3 months
            - AutoPrune = yes
        items: "{{ my_bacula_director_clients }}"
  roles:
    - bacula_director

```


Role variables
--------------

Next to the normal role variables (e.g. package name), there are
mainly variables describing resources of the
Director service configuration file. Those are of two main types:

- Resource that can occure multiple times but usually occures only once
  (e.g. `Catalogue {}`)
- Resource that can occure multiple times and is usually used
  multiple times (e.g. `Job {}`)

The full list of variables used by the role is as follows:

```
# Service name
bacula_director_service: bacula-dir

# Package to be installed (exact version can be specified here)
bacula_director_pkg: "{{
  'bacula-director-postgresql'
    if bacula_director_db_engine == 'pgsql'
    else
  'bacula-director-mysql' }}"

# Path the to the Director config file
bacula_director_config_file_path: /etc/bacula/bacula-dir.conf


# Default values of Director resource options
bacula_director_config_director_options_name: bacula-dir
bacula_director_config_director_options_port: 9101
bacula_director_config_director_options_query_file: /usr/libexec/bacula/query.sql
bacula_director_config_director_options_working_directory: /var/spool/bacula
bacula_director_config_director_options_pid_directory: /var/run
bacula_director_config_director_options_max_concurrent_jobs: 1
bacula_director_config_director_options_password: bacula
bacula_director_config_director_options_messages: Standard

# Default Director resource options
bacula_director_config_director_options__default:
  - Name = {{ bacula_director_config_director_options_name }}
  - DIR Port = {{ bacula_director_config_director_options_port }}
  - Query File = {{ bacula_director_config_director_options_query_file }}
  - Working Directory = {{ bacula_director_config_director_options_working_directory }}
  - Pid Directory = {{ bacula_director_config_director_options_pid_directory }}
  - Maximum Concurrent Jobs = {{ bacula_director_config_director_options_max_concurrent_jobs }}
  - Password = {{ bacula_director_config_director_options_password }}
  - Messages = {{ bacula_director_config_director_options_messages }}

# Custom Director resource options
bacula_director_config_director_options__custom: []

# First Director resource
bacula_director_config_director__default:
  - Director: "{{
      bacula_director_config_director_options__default +
      bacula_director_config_director_options__custom }}"

# Variable for additional Director resources (e.g. for monitor)
bacula_director_config_director__custom: []

# Final Director resource
bacula_director_config_director: "{{
  bacula_director_config_director__default +
  bacula_director_config_director__custom }}"


# Default DB engine (can be 'pgsql' or 'mysql')
bacula_director_db_engine: pgsql

# Packages required for the PostgreSQL DB creation
bacula_director_db_pgsql_pkgs:
    - python-psycopg2
    - postgresql

# Packages required for the MySQL DB creation
bacula_director_db_mysql_pkgs:
    - MySQL-python
    - mysql

# Default PostgreSQL DB port
bacula_director_db_port_pgsql_default: 5432

# Default MySQL DB port
bacula_director_db_port_mysql_default: 3306

# Default path to the MySQL config file
bacula_director_db_mysql_config: /etc/my.cfg

# Host from which the user can login to the MySQL server (all hosts by default)
bacula_director_db_mysql_user_host: "%"

# User used to create DB and user
bacula_director_db_login_user: "{{
  'postgres'
    if bacula_director_db_engine == 'pgsql'
    else
  'root' }}"

# Password used to create DB and user
bacula_director_db_login_password: "{{
  'postgres'
     if bacula_director_db_engine == 'pgsql'
     else
  None }}"

# DB user priviledges
bacula_director_db_user_priv: "{{
  'ALL/unsavedfiles:ALL/basefiles:ALL/jobmedia:ALL/file:ALL/job:ALL/media:ALL/client:ALL/pool:ALL/fileset:ALL/path:ALL/filename:ALL/counters:ALL/version:ALL/cdimages:ALL/mediatype:ALL/storage:ALL/device:ALL/status:ALL/location:ALL/locationlog:ALL/log:ALL/jobhisto:ALL/pathhierarchy:ALL/pathvisibility:SELECT,UPDATE/filename_filenameid_seq:SELECT,UPDATE/path_pathid_seq:SELECT,UPDATE/fileset_filesetid_seq:SELECT,UPDATE/pool_poolid_seq:SELECT,UPDATE/client_clientid_seq:SELECT,UPDATE/media_mediaid_seq:SELECT,UPDATE/job_jobid_seq:SELECT,UPDATE/file_fileid_seq:SELECT,UPDATE/jobmedia_jobmediaid_seq:SELECT,UPDATE/basefiles_baseid_seq:SELECT,UPDATE/storage_storageid_seq:SELECT,UPDATE/mediatype_mediatypeid_seq:SELECT,UPDATE/device_deviceid_seq:SELECT,UPDATE/location_locationid_seq:SELECT,UPDATE/locationlog_loclogid_seq:SELECT,UPDATE/log_logid_seq:ALL'
    if bacula_director_db_engine == 'pgsql'
    else
  (bacula_director_config_catalog_options_db_name + '.*:ALL') }}"


# Values of the default Catalog options
bacula_director_config_catalog_options_name: Default
bacula_director_config_catalog_options_db_name: bacula
bacula_director_config_catalog_options_db_user: bacula
bacula_director_config_catalog_options_db_password: bacula
bacula_director_config_catalog_options_db_address: localhost
bacula_director_config_catalog_options_db_port: "{{
  bacula_director_db_port_pgsql_default
    if bacula_director_db_engine == 'pgsql'
    else
  bacula_director_db_port_mysql_default }}"

# Default Catalog resource options for the first Catalog definition
bacula_director_config_catalog_options__default:
  - Name = {{ bacula_director_config_catalog_options_name }}
  - DB Name = {{ bacula_director_config_catalog_options_db_name }}
  - User = {{ bacula_director_config_catalog_options_db_user }}
  - Password = {{ bacula_director_config_catalog_options_db_password }}
  - DB Address = {{ bacula_director_config_catalog_options_db_address }}
  - DB Port = {{ bacula_director_config_catalog_options_db_port }}

# Custom Catalog resource options for the first Catalog definition
bacula_director_config_catalog_options__custom: []

# Default Catalog resource options
bacula_director_config_catalog__default:
  - Catalog: "{{
      bacula_director_config_catalog_options__default +
      bacula_director_config_catalog_options__custom }}"

# Custom Catalog resource options
bacula_director_config_catalog__custom: []

# Final Catalog resource
bacula_director_config_catalog: "{{
  bacula_director_config_catalog__default +
  bacula_director_config_catalog__custom }}"


# Values of the default Storage options
bacula_director_config_storage_options_name: File
bacula_director_config_storage_options_address: "{{ ansible_fqdn }}"
bacula_director_config_storage_options_sdport: 9103
bacula_director_config_storage_options_password: bacula
bacula_director_config_storage_options_device: FileStorage
bacula_director_config_storage_options_media_type: File

# Default Storage resource options for the first Storage definition
bacula_director_config_storage_options__default:
  - Name = {{ bacula_director_config_storage_options_name }}
  - Address = {{ bacula_director_config_storage_options_address }}
  - SD Port = {{ bacula_director_config_storage_options_sdport }}
  - Password = {{ bacula_director_config_storage_options_password }}
  - Device = {{ bacula_director_config_storage_options_device }}
  - Media Type = {{ bacula_director_config_storage_options_media_type }}

# Custom Storage resource options for the first Storage definition
bacula_director_config_storage_options__custom: []

# Default Storage resource options
bacula_director_config_storage__default:
  - Storage: "{{
      bacula_director_config_storage_options__default +
      bacula_director_config_storage_options__custom }}"

# Custom Storage resource options
bacula_director_config_storage__custom: []

# Final Storage resource
bacula_director_config_storage: "{{
  bacula_director_config_storage__default +
  bacula_director_config_storage__custom }}"


# Values of the default Pool options
bacula_director_config_pool_options_name: File
bacula_director_config_pool_options_pool_type: Backup
bacula_director_config_pool_options_recycle: "yes"
bacula_director_config_pool_options_autoprune: "yes"
bacula_director_config_pool_options_volume_retention: 365 days

# Default Pool resource options for the first Pool definition
bacula_director_config_pool_options__default:
  - Name = {{ bacula_director_config_pool_options_name }}
  - Pool Type = {{ bacula_director_config_pool_options_pool_type }}
  - Recycle = {{ bacula_director_config_pool_options_recycle }}
  - AutoPrune = {{ bacula_director_config_pool_options_autoprune }}
  - Volume Retention = {{ bacula_director_config_pool_options_volume_retention }}

# Custom Pool resource options for the first Pool definition
bacula_director_config_pool_options__custom: []

# Default Pool resource options
bacula_director_config_pool__default:
  - Pool: "{{
      bacula_director_config_pool_options__default +
      bacula_director_config_pool_options__custom }}"

# Custom Pool resource options
bacula_director_config_pool__custom: []

# Final Pool resource
bacula_director_config_pool: "{{
  bacula_director_config_pool__default +
  bacula_director_config_pool__custom }}"


# Schedule resource
bacula_director_config_schedule: []
# Example of a Schedule resource:
#bacula_director_config_schedule:
#  - Schedule:
#    - Name = Daily
#    - Run = Full sun at 23:10
#    - Run = Incremental mon-sat at 23:10


# FileSet resource
bacula_director_config_fileset: []
# Example of a FileSet resource:
#bacula_director_config_fileset:
#  - FileSet:
#    - Name = Home
#    - Include:
#      - File = /home
#    - Exclude:
#      - File = /home/nobody


# Default values of Messages resource options
bacula_director_config_messages_options_name: Standard
bacula_director_config_messages_options_console: all, !skipped, !saved
bacula_director_config_messages_options_append: /var/log/bacula/bacula.log = all, !skipped
bacula_director_config_messages_options_catalog: all

# Default Messages resource options
bacula_director_config_messages_options__default:
  - Name = {{ bacula_director_config_messages_options_name }}
  - Console = {{ bacula_director_config_messages_options_console }}
  - Append = {{ bacula_director_config_messages_options_append }}
  - Catalog = {{ bacula_director_config_messages_options_catalog }}

# Custom Messages resource options
bacula_director_config_messages_options__custom: []

# First Messages resource
bacula_director_config_messages__default:
  - Messages: "{{
      bacula_director_config_messages_options__default +
      bacula_director_config_messages_options__custom }}"

# Variable for additional Messages resources
bacula_director_config_messages__custom: []

# Final Messages resource
bacula_director_config_messages: "{{
  bacula_director_config_messages__default +
  bacula_director_config_messages__custom }}"


# JobDefs resource
bacula_director_config_jobdefs: []
# Example of a JobDefs resource:
#bacula_director_config_jobdefs:
#  - JobDefs:
#    - Name = Default
#    - Type = Backup
#    - Level = Incremental
#    - FileSet = Home
#    - Messages = Standard
#    - Pool = File
#    - Storage = File
#    - Schedule = Daily

# Job resource
bacula_director_config_job: []
# Example of a Job resource:
#bacula_director_config_job:
#  - template:
#      - Job:
#        - Name = TestJob
#        - JobDefs = Default
#        - Client = {[{ item }]}-fd
#    items:
#      - localhost.localdomain


# Client resource
bacula_director_config_client: []
# Example of a Client resource:
#bacula_director_config_client:
#  - template:
#      - Client:
#        - Name = {[{ item['host'] }]}-fd
#        - Address = {[{ item['host'] }]}
#        - FD Port = 9102
#        - Catalog = Default
#        - Password = {[{ item['password'] }]}
#        - File Retention = {[{ item['file_retention'] }]}
#        - Job Retention = 3 months
#        - AutoPrune = yes
#    items:
#      - host: localhost.localdomain
#        password: bacula
#        file_retention: 30 days


# Custom resources (e.g. Counter, include subfiles, ...)
bacula_director_config_custom: []


# Path to the sysconfig file
bacula_director_sysconfig_file: /etc/sysconfig/bacula-dir

# Default values of the default sysconfig options
bacula_director_sysconfig_user: ""
bacula_director_sysconfig_group: ""

# Default sysconfig options
bacula_director_sysconfig__default:
  DIR_USER: "{{ bacula_director_sysconfig_user }}"
  DIR_GROUP: "{{ bacula_director_sysconfig_group }}"

# Custom sysconfig options
bacula_director_sysconfig__custom: {}

# Final sysconfig
bacula_director_sysconfig: "{{
  bacula_director_sysconfig__default.update(bacula_director_sysconfig__custom) }}{{
  bacula_director_sysconfig__default }}"
```


Dependencies
------------

- [`bacula_client`](https://github.com/jtyr/ansible-bacula_client) (optional)
- [`bacula_console`](https://github.com/jtyr/ansible-bacula_console) (optional)
- [`bacula_storage`](https://github.com/jtyr/ansible-bacula_storage) (optional)
- [`config_encoder_filters`](https://github.com/jtyr/ansible-config_encoder_filters)
- [`postgresql`](https://github.com/jtyr/ansible-postgresql) (optional)


License
-------

MIT


Author
------

Jiri Tyr
