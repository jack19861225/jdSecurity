<!-- CSS work goes here for the time being -->
<!-- set a:link text-decoration to none -->
<!-- set a:hover text-decoration to underline -->
<!-- http://forums.markdownpad.com/discussion/143/include-pdf-pagebreak-instructions-in-markdown/p1 -->

# <center> <a name="cdh_security_section"/>CDH Security

* <a href="#security_review">Quick basics overview</a>
* <a href="#security_authentication">Strong Authentication</a>
* <a href="#security_authorization">Better Authorization</a>
* <a href="#security_encryption">Encryption</a>
* <a href="#security_visibility">Auditing &amp; Visibility</a>
* <a href="#security_cm_configuration">CM-based Configuration</a>

---
<div style="page-break-after: always;"></div>

## <center> <a name="security_review">Quick basics overview</a>

* **Perimeter**
    * Strong authentication
    * Network isolation, edge nodes
    * Firewalls, iptables
* **Access**
    * Authorization controls
    * Granular access to HDFS files, Hive/Impala objects
* **Data**
    * Encryption-at-rest
    * Encryption-in-transit
      * Transport Layer Security (TLS)
* **Visibility**
    * Auditing data practices without exposing content
    * Separation of concerns: storage management vs. data stewardship

---
<div style="page-break-after: always;"></div>

## <center> <a name="security_authentication"/>Strong Authentication/Kerberos</a>

* ["Hadoop in Secure Mode"](http://hadoop.apache.org/docs/r2.6.0/hadoop-project-dist/hadoop-common/SecureMode.html) lists four areas of authentication concern. All of them depend on Kerberos, directly or indirectly
  * Users
  * Hadoop services
  * Web consoles
  * Data confidentiality

* Linux supports [MIT Kerberos](http://web.mit.edu/kerberos/)
    * See your [Hadoop for Administrators](http://university.cloudera.com/course/administrator) notes for an overview
* ["Hadoop in Secure Mode"](http://hadoop.apache.org/docs/r2.6.0/hadoop-project-dist/hadoop-common/SecureMode.html) relies on Kerberos
    * Data encryption services available out of the box
      * RPC (SASL QOP "quality-of-protection")
    * Browser authentication supported by [HTTP SPNEGO](http://en.wikipedia.org/wiki/SPNEGO)
* LDAP/Active Directory integration
    * Applying existing user databases to Hadoop cluster is a common ask
* [ELI5: Kerberos](http://www.roguelynn.com/words/explain-like-im-5-kerberos/): Great introduction / refresher to Kerberos concepts.        

---
<div style="page-break-after: always;"></div>

## <center> Active Directory Integration </center>

* Cloudera recommends Direct-to-AD integration as preferred practice.
* The alternative is a [one-way cross-realm trust to AD](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_sg_hadoop_security_active_directory_integrate.html)
    * Requires MIT Kerberos realm in Hadoop cluster
    * Avoids adding service principals to AD
* Common sticking points
    * Admin reluctance(!)
    * Version / feature incompatibility
    * Misremembered details
    * Other settings that "shouldn't be a problem"

---
<div style="page-break-after: always;"></div>

## <center> Common Direct-to-AD Issues </center>

* <code>/etc/krb5.conf</code> doesn't authenticate to KDC
    * Test with <code>kinit *AD_user*</code>
* Required encryption type isn't allowed by JDK
    * Install Unlimited Policy files
* Supported encryption types are disjoint
    * Check AD "functional level"

* To trace Kerberos & Hadoop
    * <code>export KRB5_TRACE=/dev/stderr</code>
    * Include <code>-Dsun.security.krb5.debug=true</code> in <code>HADOOP_OPTS</code> (& export it )
    * <code>export HADOOP_ROOT_LOGGER="DEBUG,console"</code>

---
<div style="page-break-after: always;"></div>

## <center> <a name="security_authorization">Fine-grained Authorization</a>

* <a href="#hdfs_perms_acls">HDFS permissions & ACLs</a>
    * Need principal definitions beyond user-group-world
    * Relief from edge cases and implications of hierarchical data
    * Can provide permissions for a restricted list of users and groups
* [Apache Sentry (incubating)](https://sentry.incubator.apache.org/)
    * Database servers need files for storage, managed by admins
    * Authorizations needed for database objects may be disjoint

---
<div style="page-break-after: always;"></div>

## <center> <a name="hdfs_perms_acls"/>[HDFS Permissions](http://hadoop.apache.org/docs/r2.6.0/hadoop-project-dist/hadoop-hdfs/HdfsPermissionsGuide.html) </center>

* Plain HDFS permissions are largely POSIX-ish
    * Execution bit doesn't work except as a sticky bit
    * Applied to simple or Kerberos credentials
        * The NameNode process owner <i>is</i> the HDFS superuser
* [POSIX-style ACLs also supported](http://hadoop.apache.org/docs/r2.6.0/hadoop-project-dist/hadoop-hdfs/HdfsPermissionsGuide.html#ACLs_Access_Control_Lists)
    * Disabled by default (<code>dfs.namenode.acls.enabled</code>)
    * Additional permissions for named users, groups, other, and the mask
        * chmod operates on mask filter -> effective permissions
    * Best used to refine, not replace, file permissions
        * Some overhead to store/process them

---
<div style="page-break-after: always;"></div>

## <center> [Apache Sentry Basics](http://blog.cloudera.com/blog/2013/07/with-sentry-cloudera-fills-hadoops-enterprise-security-gap/) </center>

* Originally a Cloudera project, now [Apache incubating](http://sentry.apache.org/)
    * Some useful docs are not yet migrated to ASF
* Supports authorization for database objects
    * Objects: server, database, table, view, URI
    * Authorizations: <code>SELECT</code>, <code>INSERT</code>, <code>ALL</code>
* A Sentry policy is defined by mapping a role to a privilege
    * A group (LDAP or Linux) is then assigned to an Sentry role
    * Users can be added or removed from the group as necessary
* Supports Hive (through [HiveServer2](http://blog.cloudera.com/blog/2013/07/how-hiveserver2-brings-security-and-concurrency-to-apache-hive/)), Impala and Search (Solr) out of the box
* Sentry policy is defined by mappings
    * Local/LDAP groups -> Sentry roles
    * Sentry roles -> database object, privileges

---
<div style="page-break-after: always;"></div>

## <center> Sentry Design </center>

### <center> Graphic overview</center>

<center><img src="http://blog.cloudera.com/wp-content/uploads/2013/07/Untitled.png"></center>

---
<div style="page-break-after: always;"></div>

## <center> Sentry Design Notes

* Each service has to bind to a policy engine
    * Currently <code>impalad</code> and HiveServer2 have hooks
    * Cloudera Search integration is a workaround
* Service Provider interfaces for persisting policies to a store
    * Support for file storage to HDFS or local filesystem
* The policy engine grants/revokes access
    * Rules applied to user, the objects requested and the necessary permission
* Sentry / HDFS Synchronization
    * Automatically adds ACLs to match permission grants in Sentry
* A fully-formed [config example is here](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_sg_sentry.html#concept_iw1_5dp_wk_unique_1)
* You can watch a short [video overview here](http://vimeo.com/79936560)

---
<div style="page-break-after: always;"></div>

## <center> Sentry and [HiveServer2](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_ig_hiveserver2_configure.html)

<center><img src="https://blogs.apache.org/sentry/mediaresource/1554e24d-1365-4feb-9d0d-5832ecb90628"></center>

---
<div style="page-break-after: always;"></div>

## <center> [Sentry as a Service](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_sg_sentry_service.html)

* Relational model and storage
* Introduced in C5.1
* Uses a database to store policies
* CDH supports migrating file-based authorization
    * <code>sentry --command config-tool --policyIni *policy_file* --import</code>
* Impala & Hive must use the same provider (db or file)
* Cloudera Search can only use the file provider

---
<div style="page-break-after: always;"></div>

## <center> <a name="security_encryption">Encryption in transit </a></center>

[Network ("in-flight") encryption](http://blog.cloudera.com/blog/2013/03/how-to-set-up-a-hadoop-cluster-with-network-encryption/)
* For communication between web services (HTTPS)
  * Digital certificates, private key stores
* HDFS Block data transfer
  * `dfs.encrypt.data.transfer` (very slow - not recommended for now)
* RPC support already in place
* Support includes MR shuffling, Web UI, HDFS data and fsimage transfers

At-rest encryption
* Encryption/decryption that is transparent to Hadoop applications
* Need: Key-based protection
* Need: Minimal performance cost
  * AES-NI on recent Intel CPUs.
* Navigator Encrypt
  * Block device encryption at OS level
* HDFS Transparent Data Encryption
  * Encryption Zones
  * Key Management Server (KMS)
* Key Trustee
  * Cloudera's enterprise-grade keystore

Other requirements
* Tokenization
* Data masking
  * Leverage partners for this (Protegrity, Dataguise etc)

---
<div style="page-break-after: always;"></div>

## <center> <a name="security_visibility">Auditing &amp; Visibility</a></center>

* Provided by Cloudera Navigator
* See who has accessed resources (filesystem, databases, log of queries run)
* Custom reports
  * e.g. show all failed access attempts
* Redaction of sensitive information
  * Separation of duties

---
<div style="page-break-after: always;"></div>

## <center> <a name="security_cm_configuration">Preparing Kerberos Configuration</a>

* Know the [network ports that CDH and third-party software use](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_ig_ports_cdh5.html)
* Set up a dedicated Kerberos Domain Controller
* KRB5 MIT [instructions are here](http://web.mit.edu/Kerberos/krb5-1.8/krb5-1.8.6/doc/krb5-install.html#Realm-Configuration-Decisions)
* Cloudera [slightly higher-level instructions are here](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_sg_intro_kerb.html)
* Or you can use [RedHat's documentation](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Managing_Smart_Cards/installing-kerberos.html)
* Make sure your KDC allows *renewable tickets*
=======
* Make sure your KDC allows renewable tickets
  * Kerberos tickets are not renewable by default in many Linux distributions (see [RHEL docs](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/Configuring_Domains-Setting_up_Kerberos_Authentication.html)). 
  * Configure renewable tickets **before** Kerberos database is initialized.
  * If you modify these parameters after initialization, you can:
    1. Change the maxlife for all principals (`krbtgt/REALM` too) with `modprinc`, or 
    2. Destroy the KDB and remake it. 
* Create a KDC account for the Cloudera Manager user

---
<div style="page-break-after: always;"></div>

## <center> Security Lab
## <center> Integrating Kerberos with Cloudera Manager

* Plan one: follow the [documentation here](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_sg_s4_kerb_wizard.html)
* Plan two: Launch the Kerberos wizard and complete the checklist.
* Set up an MIT KDC
* Create a Linux account with your GitHub name
* Once your integration succeeds, add these files to your <code>security/</code> folder:
    * <code>/etc/krb5.conf</code>
    * <code>/var/kerberos/krb5kdc/kdc.conf</code>
    * <code>/var/kerberos/krb5kdc/kadm5.acl</code>
* Create a file <code>kinit.md</code> that includes:
    * The <code>kinit</code> command you use to authenticate your user
    * The output from <code>klist</code> showing your credentials
* Create a file <code>cm_creds.png</code> that shows the principals CM generated

---
<div style="page-break-after: always;"></div>

## <center> Optional challenge - Test-driven setup
## <center>[JDBC Connections in a Kerberized Cluster](http://blog.cloudera.com/blog/2014/05/how-to-configure-jdbc-connections-in-secure-apache-hadoop-environments/)</center>

* There's a lot work in this lab. If you choose to do it, be sure to:
* Ignore the steps to set up CDH 5 (already done)
* Test client connectivity with JDBC
* Set up and integrate an Active Directory instance
* Test with a secured client connection
* Enable Kerberos
* Add a Sentry configuration to the mix
* Test client connection again

If you're comfortable with AD, this may take an hour. If not, maybe 2-3 hours. Let your instructors know if you want to attempt this lab.

---
<div style="page-break-after: always;"></div>

## <center> Security Lab (Choose A or B)

Complete *one* of the following labs:<p>

* [Sentry Policy File Configuration](http://www.cloudera.com/documentation/enterprise/latest/topics/cdh_sg_sentry.html)
* [Sentry as a Service Configuration (new in CDH 5.1)](http://www.cloudera.com/documentation/enterprise/latest/topics/sg_sentry_service_config.html)
