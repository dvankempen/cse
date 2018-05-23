![SAP HANA Academy](https://yt3.ggpht.com/-BHsLGUIJDb0/AAAAAAAAAAI/AAAAAAAAAVo/6_d1oarRr8g/s100-mo-c-c0xffffffff-rj-k-no/photo.jpg)
# Client-Side Data Encryption #
### Tutorial Video Playlist ### 
[SAP HANA Client-Side Data Encryption](https://www.youtube.com/playlist?list=PLkzo92owKnVygoKWpwy4boITfzsJCqgxw)

## What's New? ##
In the first video, the concepts of client-side data encryption are explained. 
### Tutorial Video ### 
[![What's New](https://img.youtube.com/vi/N8TfjarYrac/0.jpg)](https://www.youtube.com/watch?v=N8TfjarYrac "What's New")

## Installation and Configuration ##

The SAP Common Crypto Library (libsapcrypto.so/sapcrypto.dll) and the sapgenpse(.exe) utility required for client-side encryption are included with the SAP HANA client. For the latest version of the library, see [SAP Downloads on the SAP ONE Support Portal](https://support.sap.com/swdc) and search for COMMONCRYPTOLIB.

To initialize the SAP Crypto Library, two environment variables need to be defined: the path to the library and to the PSE tool. For both, this is the SAP HANA client installation directory. 
```
--Windows
set PATH="C:\Program Files\sap\hdbclient;%PATH%"
set SECUDIR="C:\Program Files\sap\hdbclient"

--bash
export LD_LIBRARY_PATH=/usr/sap/hdbclient:$LD_LIBRARY_PATH
export SECUDIR=/usr/sap/hdbclient

--csh
setenv LD_LIBRARY_PATH=/usr/sap/hdbclient:${LD_LIBRARY_PATH}
setenv SECUDIR=/usr/sap/hdbclient
```
For Windows, use Control Panel to set these as system variables. For Linux, add them to the login script (e.g. /etc/profile.d/saphanaclient.sh). For SAP HANA clients that are part of a SAP HANA server installation, use the customer.sh file.
```
export HDBCLIENT=/usr/sap/hdbclient
export LD_LIBRARY_PATH=$HDBCLIENT:$LD_LIBRARY_PATH
export PATH=$HDBCLIENT:$PATH
export SECUDIR=$HDBCLIENT
```
Without these variables, the following error is returned:
```
*  -10429 sapcrypto library was not initialized. Check that the sapcrypto library is in the library path and SECUDIR is set appropriately.  
```
### Tutorial Video ### 
[![DML](https://img.youtube.com/vi/wrcbiueS3j4/0.jpg)](https://www.youtube.com/watch?v=wrcbiueS3j4 "Installation")

### Documentation ### 
* [Configuring the Client for Client-Side Encryption and LDAP](https://help.sap.com/viewer/e7e79e15f5284474b965872bf0fa3d63/2.0.03/en-US/34712c46d7104a2d91ed2f10c66bbc9e.html)

## Getting Started with Client-Side Data Encryption ##
In the next two videos, we are going to set client-side encryption up. For this, we are going to create three users: a key administrator, a data administrator, and a business user (HR Manager). 
```
CREATE USER hrapp_key_admin PASSWORD **** NO FORCE_FIRST_PASSWORD_CHANGE; 
CREATE USER hrapp_data_admin PASSWORD **** NO FORCE_FIRST_PASSWORD_CHANGE; 
CREATE USER hrapp_hr_manager PASSWORD **** NO FORCE_FIRST_PASSWORD_CHANGE; 
````
The data administrator will have the (system) privilege to create the schema and the tables. 
The key administrator will have the (system) privilege to create client-side encryption key pairs. 
```
GRANT CREATE SCHEMA TO hrapp_data_admin;
GRANT CREATE CLIENTSIDE ENCRYPTION KEYPAIR TO hrapp_key_admin WITH ADMIN OPTION; 
GRANT DROP CLIENTSIDE ENCRYPTION KEYPAIR TO hrapp_key_admin; 
```
SQL statements with the clause "CLIENTSIDE ENCRYPTION" included will have to executed on a client that supports and is configured for client-side encryption. Note that only ODBC and JDBC are supported, so you cannot use the SQL Prompt of the SAP HANA Database Explorer (cockpit), for example. 

One example, is to use the SAP HANA Interactive Terminal (hdbsql). You might want to use hdbuserstore to store a key with the connection string (including the password). 
The example below assumes the user user_admin exists and that this user has the privilege WITH ADMIN OPTION itself. 
```
./hdbuserstore -i set user_admin <host>:<port>@<database_name> user_admin  
./hdbsql -U user_admin "GRANT CREATE CLIENTSIDE ENCRYPTION KEYPAIR TO hrapp_key_admin WITH ADMIN OPTION";
```

As the data administrator, we will first grant the (schema) privilege to create column encryption keys to the key administrator. Because the GRANT statement includes the "CLIENTSIDE ENCRYPTION" clause, this statement will need to be executed on the client. 
```
CONNECT hrapp_data_admin PASSWORD ****;
CREATE SCHEMA hrapp; 
GRANT CLIENTSIDE ENCRYPTION COLUMN KEY ADMIN ON SCHEMA hrapp TO hrapp_key_admin WITH ADMIN OPTION; 
```

Next, the key administrator will a local client key pair (on the client) and a encryption column key encrypted with that CKP. The key administrator will also grant the business user the (system) privilege to create local key pairs. 
```
CONNECT hrapp_key_admin PASSWORD ****;
CREATE CLIENTSIDE ENCRYPTION KEYPAIR key_admin_ckp;
CREATE CLIENTSIDE ENCRYPTION COLUMN KEY hrapp.hrapp_cek1 ENCRYPTED WITH KEYPAIR key_admin_ckp;
GRANT CREATE CLIENTSIDE ENCRYPTION KEYPAIR TO hrapp_hr_manager;
```

You could run this as a script, for example. When accessing the local key store, you need to specify the password. For ODBC, the parameter is '-Z CLIENTSIDE_ENCRYPTION_KEYSTORE_PASSWORD=\<password\>'. 
```
hdbuserstore -i set hrapp_key_admin <host>:<port>@<database_name> hrapp_key_admin
hdbsql -U hrapp_key_admin -Z CLIENTSIDE_ENCRYPTION_KEYSTORE_PASSWORD=KeyStore1 -I key_admin.sql
```
  
To view the contents of the local key store, use the hdbkeystore command
```
hdbkeystore -p KeyStore1 LIST
```
  
Initially, the empty key store will not have a password set. The password passed as parameter when creating the first local key pair, will be the password for that key store. 

Next, the data administrator can create (or alter) a table with a column for which client-side encryption will be set to ON using a particular CEK. The business user will be granted the privilege to select, insert and update for that table. 

```
SET SCHEMA hrapp; 
CREATE TABLE employees (
  emp_id INT PRIMARY KEY, 
  name NVARCHAR(32), 
  ssn NVARCHAR(9) CLIENTSIDE ENCRYPTION ON WITH hrapp_cek1 DETERMINISTIC,
  salary NVARCHAR(15));
GRANT INSERT, SELECT, UPDATE ON employees TO hrapp_hr_manager;
```

Client-side encryption information is available in the TABLE_COLUMNS view
``` 
SELECT CLIENTSIDE_ENCRYPTION_STATUS, CLIENTSIDE_ENCRYPTION_COLUMN_KEY_ID, CLIENTSIDE_ENCRYPTION_MODE 
  FROM TABLE_COLUMNS 
 WHERE SCHEMA_NAME = 'HRAPP' 
   AND TABLE_NAME = 'EMPLOYEES' 
   AND COLUMN_NAME = 'SSN';
```

On another client (in our example on a Windows system), the business user also creates a local keypair. For this, we are using a JDBC client (SAP HANA studio) and pass the JDBC connection property 'cseKeyStorePassword=\<password\>'. 
```
CONNECT hrapp_hr_manager PASSWORD ****;
CREATE CLIENTSIDE ENCRYPTION KEYPAIR hr_manager_ckp;
```
  
Information about client-side encryption keypairs is avaiable in a view:
```
SELECT CREATOR, CREATE_TIME, KEYPAIR_NAME, KEYPAIR_ID FROM CLIENTSIDE_ENCRYPTION_KEYPAIRS;
```

Finally, the key administrator will create a copy of the column encryption key (CEK) and encrypt this with the client key pair (CKP) of the business user. 
```
CONNECT hrapp_key_admin PASSWORD ****;
SET SCHEMA hrapp; 
ALTER CLIENTSIDE ENCRYPTION COLUMN KEY hrapp_cek1 ADD KEYCOPY ENCRYPTED WITH KEYPAIR hr_manager_ckp;
```
### Tutorial Video ### 
[![Setup Client-Side Data Encryption](https://img.youtube.com/vi/AuXXG6pF-7c/0.jpg)](https://www.youtube.com/watch?v=AuXXG6pF-7c "Setup Client-Side Data Encryption")

[![Setup Client-Side Data Encryption II](https://img.youtube.com/vi/Ma-0tVV4ROo/0.jpg)](https://www.youtube.com/watch?v=Ma-0tVV4ROo "Setup Client-Side Data Encryption II")

### Documentation ### 
* [Getting Started With Client-Side Encryption](https://help.sap.com/viewer/b3ee5778bc2e4a089d3299b82ec762a7/2.0.03/en-US/445255ea54e5481faf10092118f2a514.html)
* [Client-Side Data Encryption (SAP HANA Security Guide)](https://help.sap.com/viewer/b3ee5778bc2e4a089d3299b82ec762a7/2.0.03/en-US/d7dc0b57c68d442ebc2af3815d9ea11e.html)
* [Client-Side Data Encryption (SAP HANA Administration Guide)](https://help.sap.com/viewer/b3ee5778bc2e4a089d3299b82ec762a7/2.0.03/en-US/445255ea54e5481faf10092118f2a514.html)

## Using DML (Insert, Update, Delete) with Client-Side Data Encryption ##

To insert or update data in the employees table, the business user must use prepared statements. 

```
CONNECT hrapp_hr_manager PASSWORD ****;
SET SCHEMA hrapp; 
/* (2153, John Adams, 123456789, 156000) */
/* (2154, Jane Doe, 987654321, 170000) */
INSERT INTO employees VALUES(?,?,?,?);
SELECT * FROM employees;
UPDATE employees SET salary = ? WHERE ssn = ?;
SELECT * FROM employees WHERE ssn = ?;
```
### Tutorial Video ### 
[![DML](https://img.youtube.com/vi/ei-NsCi4yXk/0.jpg)](https://www.youtube.com/watch?v=ei-NsCi4yXk "DML")

### Documentation ### 
* [Supported DML With Client-Side Encryption](https://help.sap.com/viewer/6b94445c94ae495c83a19646e7c3fd56/2.0.03/en-US/87d64b47692145d6b6e9e53c90e01cab.html)

## Using DDL (Create, Alter, Drop) with Client-Side Data Encryption ##
The CLIENTSIDE ENCRYPTION [ON WITH|WITH|OFF] clause controls client-side column encryption. 

The optional [DETERMINISTIC|RANDOM] specifies which type of encryption should be used. With DETERMINISTIC, the same input data and the same CEK will return the same encrypted values. 

While this enables you to compare encrypted values for equality, when using for booleans, flags (Y/N), or low-cardinality values (Male/Female), the original data might be guessable and hence not a good candidate for this type of encryption. 
Default is RANDOM returning new encrypted values each time. 
```
/* Create table with encrypted column */
CREATE TABLE employees (
  emp_id INT PRIMARY KEY, 
  name NVARCHAR(32), 
  ssn NVARCHAR(9) CLIENTSIDE ENCRYPTION ON WITH hrapp_cek1 DETERMINISTIC,
  salary NVARCHAR(15));
  
/* Enable encryption on a column after creation */
ALTER TABLE employees ALTER (ssn ALTER CLIENTSIDE ENCRYPTION ON WITH hrapp_cek1);
-- ALTER TABLE <table-name> ALTER (<column-name> ALTER CLIENTSIDE ENCRYPTION ON WITH <column-encryption-key> 
--  [ RANDOM | DETERMINISTIC ]);
  
/* Add an encrypted column with same CEK */
ALTER TABLE employees ALTER (salary ALTER CLIENTSIDE ENCRYPTION ON WITH hrapp_cek1);
  
/* Rotate CEK */ 
ALTER TABLE employees ALTER (ssn ALTER CLIENTSIDE ENCRYPTION WITH hrapp_cek2 DETERMINISTIC);

/* Disable encryption */
ALTER TABLE employees ALTER (ssn ALTER CLIENTSIDE ENCRYPTION OFF);
```
The system views CLIENTSIDE_ENCRYPTION_KEYPAIRS, CLIENTSIDE_ENCRYPTION_COLUMN_KEYS, and TABLE_COLUMNS provide information about which columns are encrypted with the corresponding CEK/CKP combination:
```
SELECT CASE cek.CREATED_FOR_COLUMN_KEY_ADMIN WHEN 'FALSE' THEN 'Key Copy' ELSE 'Master Key' END "Type", 
  cek.COLUMN_KEY_NAME CEK, ckp.KEYPAIR_NAME CKP, ckp.CREATOR "Key Pair Creator", cek.CREATE_TIME "Column Key Created", ckp.CREATE_TIME "Key Pair Created", 
  cek.SCHEMA_NAME, tc.TABLE_NAME, tc.COLUMN_NAME, tc.CLIENTSIDE_ENCRYPTION_STATUS "Status"
  FROM CLIENTSIDE_ENCRYPTION_KEYPAIRS ckp
       INNER JOIN 
       CLIENTSIDE_ENCRYPTION_COLUMN_KEYS cek ON ckp.KEYPAIR_ID = cek.ENCRYPTED_WITH_KEYPAIR_ID
       INNER JOIN
       TABLE_COLUMNS tc ON cek.COLUMN_KEY_ID = tc.CLIENTSIDE_ENCRYPTION_COLUMN_KEY_ID
 ORDER BY cek.CREATE_TIME;
```
### Tutorial Video ### 
[![DML](https://img.youtube.com/vi/4WyhrDGho6s/0.jpg)](https://www.youtube.com/watch?v=4WyhrDGho6s "DDL")

### Documentation ### 
* [Change Column Encryption Status](https://help.sap.com/viewer/6b94445c94ae495c83a19646e7c3fd56/2.0.03/en-US/b423b6ae6db248e0a45bb66114b3e389.html)

## Rotate the Column Encryption Key ##

Part of the client-side encryption procedure is to rotate CEKs regularly and re-encrypt your data using the most current CEK. Key copies for the new CEK must be created for users who need access to data.

```
-- Create new CEK as Key Administrator
./hdbsql -U hrapp_key_admin \ 
-Z CLIENTSIDE_ENCRYPTION_KEYSTORE_PASSWORD=$CSEKSPWD \
"CREATE CLIENTSIDE ENCRYPTION COLUMN KEY hrapp.hrapp_cek2 ENCRYPTED WITH KEYPAIR key_admin_ckp";

-- Add a copy for the CEK encrypted with the keypair of users who need to access the data
./hdbsql -U hrapp_key_admin \ 
-Z CLIENTSIDE_ENCRYPTION_KEYSTORE_PASSWORD=$CSEKSPWD \
"ALTER CLIENTSIDE ENCRYPTION COLUMN KEY hrapp.hrapp_cek2 ADD KEYCOPY ENCRYPTED WITH KEYPAIR hr_manager_ckp";

-- Update the table
./hdbsql -U hrapp_data_admin \ 
-Z CLIENTSIDE_ENCRYPTION_KEYSTORE_PASSWORD=$CSEKSPWD \
"ALTER TABLE hrapp.employees ALTER (ssn ALTER CLIENTSIDE ENCRYPTION WITH hrapp.hrapp_cek2)";

-- Optionally, drop the old key
./hdbsql -U hrapp_key_admin \ 
-Z CLIENTSIDE_ENCRYPTION_KEYSTORE_PASSWORD=$CSEKSPWD \
"DROP CLIENTSIDE ENCRYPTION COLUMN KEY hrapp.hrapp_cek1";
```
The system views CLIENTSIDE_ENCRYPTION_KEYPAIRS, CLIENTSIDE_ENCRYPTION_COLUMN_KEYS, and TABLE_COLUMNS provide information about which columns are encrypted with the corresponding CEK/CKP combination:
```
 /* Available CEK with CKP */ 
 SELECT CASE cek.CREATED_FOR_COLUMN_KEY_ADMIN WHEN 'FALSE' THEN 'Key Copy' ELSE 'Master Key' END "Type", 
  cek.COLUMN_KEY_NAME CEK, ckp.KEYPAIR_NAME CKP, ckp.CREATOR "Key Pair Creator", cek.CREATE_TIME "Column Key Created", ckp.CREATE_TIME "Key Pair Created", 
  cek.SCHEMA_NAME
  FROM CLIENTSIDE_ENCRYPTION_KEYPAIRS ckp
       INNER JOIN 
       CLIENTSIDE_ENCRYPTION_COLUMN_KEYS cek ON ckp.KEYPAIR_ID = cek.ENCRYPTED_WITH_KEYPAIR_ID
 ORDER BY cek.CREATE_TIME;
```
### Tutorial Video ### 
[![DML](https://img.youtube.com/vi/W2xyWo2bQLw/0.jpg)](https://www.youtube.com/watch?v=W2xyWo2bQLw "Rotate CEK")

### Documentation ### 
* [Rotate Column Encryption Key](https://help.sap.com/viewer/6b94445c94ae495c83a19646e7c3fd56/2.0.03/en-US/760d41227c9d4d32b8542f47ff02b386.html)

## Exporting Client Key Pairs and Column Encryption Keys ##
You need to export (and backup, that is, store in a safe place) both the client key pairs and column encryption keys. Although a column encryption key (copy) will be encrypted with a particular key pair, you are not required to backup or store them together. You can always create a copy of the CEK for encryption with a new CPK. 

The EXPORT system privilege is required to export CEKs.  
```
CONNECT user_admin PASSWORD ***;
GRANT EXPORT TO hrapp_data_admin;
```
To export, use the EXPORT command, the same one used as for any other SAP HANA object. 

WITH REPLACE is optional. If the object exists (here on the file system for the path and file name provided) an error is returned otherwise. 

In the code snippet below, we also export the table and drop the table to simulate moving the table object with encrypted column key to another database. 
```
CONNECT hrapp_data_admin PASSWORD ***;
EXPORT CLIENTSIDE ENCRYPTION COLUMN KEY hrapp.hrapp_cek1 AS CSV INTO '/export/hrapp_cek1.cek' WITH REPLACE;
EXPORT hrapp.employees AS BINARY INTO '/export/hrapp_employees.table' WITH REPLACE;
DROP TABLE hrapp.employees;
``` 
To export the local client key pair, connect to the client and run the hdbkeystore command. Below an example of the syntax on a Linux client. You can remove the key if you longer want to make use of the particular client. 
```
./hdbkeystore -p *** EXPORT KEY_ADMIN_CKP '/tmp/key_admin.ckp'
./hdbkeystore -p *** REMOVE KEY_ADMIN_CKP
./hdbkeystore -p *** LIST
```
Alternatively, you could use the DROP CLIENTSIDE ENCRYPTION KEYPAIR statement. 
```
/hdbsql -U hrapp_key_admin \ 
-Z CLIENTSIDE_ENCRYPTION_KEYSTORE_PASSWORD=*** \
"DROP CLIENTSIDE ENCRYPTION KEYPAIR key_admin_ckp";
```
Similarly, you can use the DROP CLIENTSIDE ENCRYPTION COLUMN KEY statement to drop the column key in case of database migration. 
```
./hdbsql -U hrapp_key_admin \ 
-Z CLIENTSIDE_ENCRYPTION_KEYSTORE_PASSWORD=*** \
"DROP CLIENTSIDE ENCRYPTION COLUMN KEY hrapp.hrapp_cek1";
```
To query the presence of CEKs and CKPs on the system, you can use the following system views:
```
SELECT * FROM CLIENTSIDE_ENCRYPTION_COLUMN_KEYS;
SELECT * FROM CLIENTSIDE_ENCRYPTION_KEYPAIRS;
```

### Tutorial Video ### 
[![Export](https://img.youtube.com/vi/AIkyHS7UBYs/0.jpg)](https://www.youtube.com/watch?v=AIkyHS7UBYs "Export")

### Documentation ### 
* [Import and Export Column Encryption Keys](https://help.sap.com/viewer/6b94445c94ae495c83a19646e7c3fd56/2.0.03/en-US/604bf3f99bae4c6092ca24205298f99f.html)
* [EXPORT Statement (Data Import Export)](https://help.sap.com/viewer/4fe29514fd584807ac9f2a04f6754767/2.0.03/en-US/20da0bec751910148e69c9668ea3ccb8.html)
* [Secure Key Store (hdbkeystore)](https://help.sap.com/viewer/b3ee5778bc2e4a089d3299b82ec762a7/2.0.03/en-US/65ec40dbe0cd4972bf3a240a1b963dc7.html)
* [DROP CLIENTSIDE ENCRYPTION COLUMN KEY Statement (Encryption)](https://help.sap.com/viewer/4fe29514fd584807ac9f2a04f6754767/2.0.03/en-US/6311317db9864dc09d697715cddaa150.html)
* [DROP CLIENTSIDE ENCRYPTION KEYPAIR Statement (Encryption)](https://help.sap.com/viewer/4fe29514fd584807ac9f2a04f6754767/2.0.03/en-US/331a8d297d5243a2bdff3cbb3b0d25f0.html)
* [CLIENTSIDE_ENCRYPTION_COLUMN_KEYS System View](https://help.sap.com/viewer/4fe29514fd584807ac9f2a04f6754767/2.0.03/en-US/1f8c1178943c416da739e54beb74e35d.html)
* [CLIENTSIDE_ENCRYPTION_KEYPAIRS System View](https://help.sap.com/viewer/4fe29514fd584807ac9f2a04f6754767/2.0.03/en-US/34d4fdb696844734ac6bd06e4de5001d.html)

## Importing Client Key Pairs and Column Encryption Keys ##
Not surprisingly, importing client key pairs and column encryption keys is very similar to exporting. 

To import client key pairs, the password to the local key store is required (and a key export file). This activity is performed on the SAP HANA client. 
```
./hdbkeystore -p **** IMPORT '/tmp/key_admin.ckp'
```

To import column encryption keys, the IMPORT system privilege is required (and export file). The import activity can be performed on any SQL Console client (SAP HANA Database Explorer, SAP HANA Studio). 

WITH REPLACE is optional. If the object exists (in the database) an error is returned otherwise. 

When restoring the data structures (import table), do not forget to re-assign the required privileges. 
```
CONNECT user_admin PASSWORD ***;
GRANT IMPORT TO hrapp_data_admin;

CONNECT hrapp_data_admin PASSWORD ***;
IMPORT CLIENTSIDE ENCRYPTION COLUMN KEY hrapp.hrapp_cek1 FROM '/export/hrapp_cek1.cek' WITH REPLACE;

IMPORT hrapp.employees FROM '/export/hrapp_employees.table' WITH REPLACE;
GRANT INSERT, SELECT, UPDATE ON hrapp.employees TO hrapp_hr_manager;

-- Metadata check
SELECT * FROM CLIENTSIDE_ENCRYPTION_COLUMN_KEYS;

-- Business User check
CONNECT hrapp_hr_manager PASSWORD ***;
SELECT * FROM employees;
```
### Tutorial Video ### 
[![Export](https://img.youtube.com/vi/9aeMDtoNUUE/0.jpg)](https://www.youtube.com/watch?v=9aeMDtoNUUE "Import")

### Documentation ### 
* [Import and Export Column Encryption Keys](https://help.sap.com/viewer/6b94445c94ae495c83a19646e7c3fd56/2.0.03/en-US/604bf3f99bae4c6092ca24205298f99f.html)
* [IMPORT Statement (Data Import Export)](https://help.sap.com/viewer/4fe29514fd584807ac9f2a04f6754767/2.0.03/en-US/20f75ade751910148492a90e5e375b8f.html)
* [Secure Key Store (hdbkeystore)](https://help.sap.com/viewer/b3ee5778bc2e4a089d3299b82ec762a7/2.0.03/en-US/65ec40dbe0cd4972bf3a240a1b963dc7.html)

## HDB Key Store ##

The location on the file system for the Key Store is in the ProgramData\.hdb\<computername>\<UID> folder on Windows and the $HOME/.hdb/<computername> directory on Linux, where the UID and the $HOME correspond to the user that create the first key pair. On Linux, file permissions are restrictive: rw for user only (600). On Windows, the file permissions allow rw for Everyone. 
  
To change (reset) the HDB Key Store password, you need to export the contents of the store to a (n encrypted) file, delete the key store on the file system, and import the export file specifying the new key store password. 
```
hdbkeystore -p <old password> EXPORT * '/tmp/key_admin.ckp' 
rm $HOME/.hdb/$HOSTNAME/hdbkeystore.dat
hdbkeystore -p <new password> '/tmp/key_admin.ckp' 
```

### Tutorial Video ### 
[![DML](https://img.youtube.com/vi/xD1NVukEUYc/0.jpg)](https://www.youtube.com/watch?v=xD1NVukEUYc "HDB Key Store")

### Documentation ### 
* [Secure Key Store (hdbkeystore)](https://help.sap.com/viewer/b3ee5778bc2e4a089d3299b82ec762a7/2.0.03/en-US/65ec40dbe0cd4972bf3a240a1b963dc7.html)
