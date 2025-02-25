---
description: "Tutorial: Signing Stored Procedures with a Certificate"
title: "Tutorial: Signing Stored Procedures with a Certificate"
ms.custom:
  - seo-dt-2019
  - intro-quickstart
ms.date: "08/23/2018"
ms.prod: sql
ms.prod_service: "database-engine"
ms.reviewer: ""
ms.technology: 
ms.topic: quickstart
helpviewer_keywords:
  - "signing stored procedures tutorial [SQL Server]"
ms.assetid: a4b0f23b-bdc8-425f-b0b9-e0621894f47e
author: "MashaMSFT"
ms.author: "mathoma"
monikerRange: "=azuresqldb-current||>=sql-server-2016||>=sql-server-linux-2017||=azuresqldb-mi-current"
---
# Tutorial: Signing Stored Procedures with a Certificate
[!INCLUDE [SQL Server Azure SQL Database SQL Managed Instance](../includes/applies-to-version/sql-asdb-asdbmi.md)]
This tutorial illustrates signing stored procedures using a certificate generated by [!INCLUDE[ssNoVersion](../includes/ssnoversion-md.md)].  
  
> [!NOTE]  
> To run the code in this tutorial you must have both Mixed Mode security configured and the AdventureWorks2017 database installed.   
  
Signing stored procedures using a certificate is useful when you want to require permissions on the stored procedure but you do not want to explicitly grant a user those rights. Although you can accomplish this task in other ways, such as using the EXECUTE AS statement, using a certificate allows you to use a trace to find the original caller of the stored procedure. This provides a high level of auditing, especially during security or Data Definition Language (DDL) operations.  
  
You can create a certificate in the master database to allow server-level permissions, or you can create a certificate in any user databases to allow database-level permissions. In this scenario, a user with no rights to base tables must access a stored procedure in the AdventureWorks2017 database, and you want to audit the object access trail. Rather than using other ownership chain methods, you will create a server and database user account with no rights to the base objects, and a database user account with rights to a table and a stored procedure. Both the stored procedure and the second database user account will be secured with a certificate. The second database account will have access to all objects, and grant access to the stored procedure to the first database user account.  
  
In this scenario you will first create a database certificate, a stored procedure, and a user, and then you will test the process following these steps:  
  
Each code block in this example is explained in line. To copy the complete example, see [Complete Example](#CompleteExample) at the end of this tutorial.  

## Prerequisites
To complete this tutorial, you need SQL Server Management Studio, access to a server that's running SQL Server, and an AdventureWorks database.

- Install [SQL Server Management Studio](../ssms/download-sql-server-management-studio-ssms.md).
- Install [SQL Server 2017 Developer Edition](https://www.microsoft.com/sql-server/sql-server-downloads).
- Download [AdventureWorks2017 sample databases](../samples/adventureworks-install-configure.md).

For instructions on restoring a database in SQL Server Management Studio, see [Restore a database](./backup-restore/restore-a-database-backup-using-ssms.md).   
  
## 1. Configure the Environment  
To set the initial context of the example, in [!INCLUDE[ssManStudioFull](../includes/ssmanstudiofull-md.md)] open a new Query and run the following code to open the Adventureworks2017 database. This code changes the database context to `AdventureWorks2017` and creates a new server login and database user account (`TestCreditRatingUser`), using a password.  
  
```sql  
USE AdventureWorks2017;  
GO  
-- Set up a login for the test user  
CREATE LOGIN TestCreditRatingUser  
   WITH PASSWORD = 'ASDECd2439587y'  
GO  
CREATE USER TestCreditRatingUser  
FOR LOGIN TestCreditRatingUser;  
GO  
```  
  
For more information on the CREATE USER statement, see [CREATE USER &#40;Transact-SQL&#41;](../t-sql/statements/create-user-transact-sql.md). For more information on the CREATE LOGIN statement, see [CREATE LOGIN &#40;Transact-SQL&#41;](../t-sql/statements/create-login-transact-sql.md).  
  
## 2. Create a Certificate  
You can create certificates in the server using the master database as the context, using a user database, or both. There are multiple options for securing the certificate. For more information on certificates, see [CREATE CERTIFICATE &#40;Transact-SQL&#41;](../t-sql/statements/create-certificate-transact-sql.md).  
  
Run this code to create a database certificate and secure it using a password.  
  
```sql  
CREATE CERTIFICATE TestCreditRatingCer  
   ENCRYPTION BY PASSWORD = 'pGFD4bb925DGvbd2439587y'  
      WITH SUBJECT = 'Credit Rating Records Access',   
      EXPIRY_DATE = '12/31/2021';  -- Error 3701 will occur if this date is not in the future
GO  
```  
  
## 3. Create and Sign a Stored Procedure Using the Certificate  
Use the following code to create a stored procedure that selects data from the `Vendor` table in the `Purchasing` database schema, restricting access to only the companies with a credit rating of 1. Note that the first section of the stored procedure displays the context of the user account running the stored procedure, which is to demonstrate the concepts only. It is not required to satisfy the requirements.  
  
```sql  
CREATE PROCEDURE TestCreditRatingSP  
AS  
BEGIN  
   -- Show who is running the stored procedure  
   SELECT SYSTEM_USER 'system Login'  
   , USER AS 'Database Login'  
   , NAME AS 'Context'  
   , TYPE  
   , USAGE   
   FROM sys.user_token     
  
   -- Now get the data  
   SELECT AccountNumber, Name, CreditRating   
   FROM Purchasing.Vendor  
   WHERE CreditRating = 1  
END  
GO  
```  
  
Run this code to sign the stored procedure with the database certificate, using a password.  
  
```sql  
ADD SIGNATURE TO TestCreditRatingSP   
   BY CERTIFICATE TestCreditRatingCer  
    WITH PASSWORD = 'pGFD4bb925DGvbd2439587y';  
GO  
```  
  
For more information on stored procedures, see [Stored Procedures &#40;Database Engine&#41;](../relational-databases/stored-procedures/stored-procedures-database-engine.md).  
  
For more information on signing stored procedures, see [ADD SIGNATURE &#40;Transact-SQL&#41;](../t-sql/statements/add-signature-transact-sql.md).  
  
## 4. Create a Certificate Account Using the Certificate  
Run this code to create a database user (`TestCreditRatingcertificateAccount`) from the certificate. This account has no server login, and will ultimately control access to the underlying tables.  
  
```sql  
USE AdventureWorks2017;  
GO  
CREATE USER TestCreditRatingcertificateAccount  
   FROM CERTIFICATE TestCreditRatingCer;  
GO  
```  
  
## 5. Grant the Certificate Account Database Rights  
Run this code to grant `TestCreditRatingcertificateAccount` rights to the base table and the stored procedure.  
  
```sql  
GRANT SELECT   
   ON Purchasing.Vendor   
   TO TestCreditRatingcertificateAccount;  
GO  
  
GRANT EXECUTE   
   ON TestCreditRatingSP   
   TO TestCreditRatingcertificateAccount;  
GO  
```  
  
For more information on granting permissions to objects, see [GRANT &#40;Transact-SQL&#41;](../t-sql/statements/grant-transact-sql.md).  
  
## 6. Display the Access Context  
To display the rights associated with the stored procedure access, run the following code to grant the rights to run the stored procedure to the `TestCreditRatingUser` user.  
  
```sql  
GRANT EXECUTE   
   ON TestCreditRatingSP   
   TO TestCreditRatingUser;  
GO  
```  
  
Next, run the following code to run the stored procedure as the dbo login you used on the server. Observe the output of the user context information. It will show the dbo account as the context with its own rights and not through a group membership.  
  
```sql  
EXECUTE TestCreditRatingSP;  
GO  
```  
  
Run the following code to use the `EXECUTE AS` statement to become the `TestCreditRatingUser` account and run the stored procedure. This time you will see the user context is set to the USER MAPPED TO CERTIFICATE context. Note that this option is not supported in a contained database or Azure SQL Database or Azure Synapse Analytics.
  
```sql  
EXECUTE AS LOGIN = 'TestCreditRatingUser';  
GO  
EXECUTE TestCreditRatingSP;  
GO  
```  
  
This shows you the auditing available because you signed the stored procedure.  
  
> [!NOTE]  
> Use EXECUTE AS to switch contexts within a database.  
  
## 7. Reset the Environment  
The following code uses the `REVERT` statement to return the context of the current account to dbo, and resets the environment.  
  
```sql  
REVERT;  
GO  
DROP PROCEDURE TestCreditRatingSP;  
GO  
DROP USER TestCreditRatingcertificateAccount;  
GO  
DROP USER TestCreditRatingUser;  
GO  
DROP LOGIN TestCreditRatingUser;  
GO  
DROP CERTIFICATE TestCreditRatingCer;  
GO  
```  
  
For more information about the REVERT statement, see [REVERT &#40;Transact-SQL&#41;](../t-sql/statements/revert-transact-sql.md).  
  
## <a name="CompleteExample"></a>Complete Example  
This section displays the complete example code.  
  
```sql  
/* Step 1 - Open the AdventureWorks2017 database */  
USE AdventureWorks2017;  
GO  
-- Set up a login for the test user  
CREATE LOGIN TestCreditRatingUser  
   WITH PASSWORD = 'ASDECd2439587y'  
GO  
CREATE USER TestCreditRatingUser  
FOR LOGIN TestCreditRatingUser;  
GO  
  
/* Step 2 - Create a certificate in the AdventureWorks2017 database */  
CREATE CERTIFICATE TestCreditRatingCer  
   ENCRYPTION BY PASSWORD = 'pGFD4bb925DGvbd2439587y'  
      WITH SUBJECT = 'Credit Rating Records Access',   
      EXPIRY_DATE = '12/31/2021';   -- Error 3701 will occur if this date is not in the future
GO  
  
/* Step 3 - Create a stored procedure and  
sign it using the certificate */  
CREATE PROCEDURE TestCreditRatingSP  
AS  
BEGIN  
   -- Shows who is running the stored procedure  
   SELECT SYSTEM_USER 'system Login'  
   , USER AS 'Database Login'  
   , NAME AS 'Context'  
   , TYPE  
   , USAGE   
   FROM sys.user_token;     
  
   -- Now get the data  
   SELECT AccountNumber, Name, CreditRating   
   FROM Purchasing.Vendor  
   WHERE CreditRating = 1;  
END  
GO  
  
ADD SIGNATURE TO TestCreditRatingSP   
   BY CERTIFICATE TestCreditRatingCer  
    WITH PASSWORD = 'pGFD4bb925DGvbd2439587y';  
GO  
  
/* Step 4 - Create a database user for the certificate.   
This user has the ownership chain associated with it. */  
USE AdventureWorks2017;  
GO  
CREATE USER TestCreditRatingcertificateAccount  
   FROM CERTIFICATE TestCreditRatingCer;  
GO  
  
/* Step 5 - Grant the user database rights */  
GRANT SELECT   
   ON Purchasing.Vendor   
   TO TestCreditRatingcertificateAccount;  
GO  
  
GRANT EXECUTE  
   ON TestCreditRatingSP   
   TO TestCreditRatingcertificateAccount;  
GO  
  
/* Step 6 - Test, using the EXECUTE AS statement */  
GRANT EXECUTE   
   ON TestCreditRatingSP   
   TO TestCreditRatingUser;  
GO  
  
-- Run the procedure as the dbo user, notice the output for the type  
EXEC TestCreditRatingSP;  
GO  
  
EXECUTE AS LOGIN = 'TestCreditRatingUser';  
GO  
EXEC TestCreditRatingSP;  
GO  
  
/* Step 7 - Clean up the example */  
REVERT;  
GO  
DROP PROCEDURE TestCreditRatingSP;  
GO  
DROP USER TestCreditRatingcertificateAccount;  
GO  
DROP USER TestCreditRatingUser;  
GO  
DROP LOGIN TestCreditRatingUser;  
GO  
DROP CERTIFICATE TestCreditRatingCer;  
GO  
```  
  
## See Also  
[Security Center for SQL Server Database Engine and Azure SQL Database](../relational-databases/security/security-center-for-sql-server-database-engine-and-azure-sql-database.md)  
  
  
