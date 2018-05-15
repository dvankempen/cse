# cse
client-side data encryption
=======
![SAP HANA Academy](https://yt3.ggpht.com/-BHsLGUIJDb0/AAAAAAAAAAI/AAAAAAAAAVo/6_d1oarRr8g/s100-mo-c-c0xffffffff-rj-k-no/photo.jpg)
=======
Playlist URL:
[SAP HANA Client-Side Data Encryption ](https://www.youtube.com/playlist?list=PLkzo92owKnVygoKWpwy4boITfzsJCqgxw)

What's New? 
-------
In the first video, the concepts of client-side data encryption are explained. 

[![What's New](https://img.youtube.com/vi/N8TfjarYrac/0.jpg)](https://www.youtube.com/watch?v=N8TfjarYrac "What's New")


Getting Started with Client-Side Data Encryption
-------
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

[![Setup Client-Side Data Encryption](https://img.youtube.com/vi/AuXXG6pF-7c/0.jpg)](https://www.youtube.com/watch?v=AuXXG6pF-7c "Setup Client-Side Data Encryption")

Using DML (Insert, Update, Delete) with Client-Side Data Encryption
-------

[![Setup Client-Side Data Encryption II](https://img.youtube.com/vi/Ma-0tVV4ROo/0.jpg)](https://www.youtube.com/watch?v=Ma-0tVV4ROo "Setup Client-Side Data Encryption II")

To insert or update data in the employees table, the business user will use prepared statements
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

[![DML](https://img.youtube.com/vi/ei-NsCi4yXk/0.jpg)](https://www.youtube.com/watch?v=ei-NsCi4yXk "DML")

Exporting Client Key Pairs and Column Encryption Keys
-------

[![Export](https://img.youtube.com/vi/AIkyHS7UBYs/0.jpg)](https://www.youtube.com/watch?v=AIkyHS7UBYs "Export")
