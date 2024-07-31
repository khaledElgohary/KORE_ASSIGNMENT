# Project Overview  
This project is an ETL pipeline made using SSIS to migrate data from a CSV file to a production SQL Server table as a part of KORE Software inteview process.

# Files Included
In this repo there is all the files related to the SSIS code for ETL pipeline, a database backup, and a csv file called Send_for_Review which contains data that does not meet the data quality standards, and needs to be sent back for review.

# How to Run
- First make sure to have your SQL server connection established to which will hold the migration results.
- After establishing the connection, if you are running Visual Basic then go to File -> Open -> Project/Solution -> KORE_ASSIGNMENT.sln
- This will open up the SSIS solution.
### Running the DB intialization script in SSMS
- If your preference is running the initial DB script in SSMS, then simply navigate to Control Flow, then right click on the Initizalize DB component.
- Then select Edit, and double click the 3 dots button in the SQLStatement field.
- Copy and paste the SQL script to a new query in your SSMS server, then run the query to intialize DB.
- Make sure to disable the Initialize DB component in the control flow to prevent running script twice, except in case of testing you can enable it since it executes the tables truncation.

### Running the DB initialization using the Execute SQL Task component.
- By default, I have an SQL Task component that is responsible for the Initialization of the DB in the control flow.
- Right click on the component and click on edit.
- Make sure under SQL Statement that the Connection is corresponding with your SQL server connection, and that your connection type is OLE DB.
- Please run package first with all other components disabled in the control flow to create database and initalize it.

### CSV source and destination
- There is 2 Flat file Source/Destination components in the data flow "Data Extraction and Type Standardization" .
- First is the initial extraction of the csv file.
- Right click on the Flat File Source and click edit, then click on new and browse the file the data will be imported from, then navigate to columns and make sure all the columns are checked out and included.
- Second is the flat file destination for the rows getting sent back for review.
- right click on the Flat File destination, then click new connection and choose the csv file you created that will host the redirected rows.

### OLE DB source and destination
- Whether you optioned to create and initialize database using SSMS or the SQL Execute Task component in SSIS, please make sure to edit your connections.
- For the data flow "Data Extraction and Type Standardization" right click on the OLE DB destination and choose edit.
- Click on New next to OLE DB connection manager, and then new.
- Select Provider to be Microsoft OLE DB Driver for SQL Server, then enter your Server name and select initial catalog which is database name that should be KoreAssignment_{Khaled_Elgohary} then click ok.
- Then you will be able to view the tables, and will need to select [stg].Users table.
- Same process will be applied for the connections in the "Duplicates check and Loading to Production" component, except in the OLE DB Destination your target table will be [prod].Users.
- Connection will need to be checked also for the OLE DB Command component.
- Right click on the OLE DB Command, then click on edit, and then select DB name from the Connection Manager drop down menu.

# Lookup component
- You will need to edit connection in the lookup component in the dataflow "Duplicates check and Loading to Production".
- Right click on lookup, click eidt, make sure connection type is OLE DB, then navigate to connections on the left panel, click New and follow same process for OLE DB source and destination connection configuration.
- Make sure to select the table to be [prod].Users . 

### DB Backup component
- A zip file containing the database backup is already included in my repo.
- However, it's optional to disable the backup component in control flow if it's not needed. By default I have turned off this component since the DB has already been backed up.

### DB initialization Script
Below is the script that executes everytime the ETL pipeline is ran. 2 lines were added which just truncates the staging and production table for every run. 
```
-- TODO: Replace {FirstName} and {LastName} with your actual first and last names before running this script 2 

IF NOT EXISTS (SELECT * FROM sys.databases WHERE name = N'KoreAssignment_{Khaled_Elgohary}')
BEGIN 
	CREATE DATABASE [KoreAssignment_{Khaled_Elgohary}]; 
END 
GO 
 
USE [KoreAssignment_{Khaled_Elgohary}] 
GO 
 
-- Check and create stg schema if it does not exist 
IF NOT EXISTS (SELECT * FROM sys.schemas WHERE name = N'stg') 
BEGIN 
	EXEC('CREATE SCHEMA stg'); 
END 
GO 
 
 -- Check and create prod schema if it does not exist 
IF NOT EXISTS (SELECT * FROM sys.schemas WHERE name = N'prod') 
BEGIN 
	EXEC('CREATE SCHEMA prod'); 
END 
GO 
 
-- Check and create stg.Users table if it does not exist 
IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'stg.Users') AND type in (N'U')) 
BEGIN 
	CREATE TABLE stg.Users ( 
		StgID INT IDENTITY(1,1) PRIMARY KEY, 
		UserID INT, 
		FullName NVARCHAR(255), 
		Age INT, 
		Email NVARCHAR(255), 
		RegistrationDate DATE, 
		LastLoginDate DATE, 
		PurchaseTotal FLOAT 
); 
END 
GO 

-- Check and create prod.Users table if it does not exist 
IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'prod.Users') AND type in (N'U'))
BEGIN 
	CREATE TABLE prod.Users ( 
		ID INT IDENTITY(1,1) PRIMARY KEY, 
		UserID INT, 
		FullName NVARCHAR(255), 
		Age INT, 
		Email NVARCHAR(255), 
		RegistrationDate DATE, 
		LastLoginDate DATE, 
		PurchaseTotal FLOAT, 
		RecordLastUpdated DATETIME DEFAULT GETDATE() 
		); 
END


truncate table [prod].Users
 
GO 

BEGIN
	INSERT INTO prod.Users (UserID, FullName, Age, Email, RegistrationDate, LastLoginDate, PurchaseTotal)
	VALUES
		(101, 'John Doe', 30, 'johndoe@example.com', '2021-01-10', '2023-03-01', 150.00),
		(102, 'Jane Smith', 25, 'janesmith@example.com', '2020-05-15', '2023-02-25', 200.00),
		(103, 'Emily Johnson', 22, 'emilyjohnson@example.com', '2019-03-23', '2023-01-30', 120.50),
		(104, 'Michael Brown', 35, 'michaelbrown@example.com', '2018-07-18', '2023-02-20', 300.75),
		(105, 'Jessica Garcia', 28, 'jessicagarcia@example.com', '2022-08-05', '2023-02-18', 180.25),
		(106, 'David Miller', 40, 'davidmiller@example.com', '2017-12-12', '2023-02-15', 220.40),
		(107, 'Sarah Martinez', 33, 'sarahmartinez@example.com', '2018-11-30', '2023-02-10', 140.60),
		(108, 'James Taylor', 29, 'jamestaylor@example.com', '2019-06-22', '2023-02-05', 210.00),
		(109, 'Linda Anderson', 27, 'lindaanderson@example.com', '2021-04-16', '2023-01-25', 165.95),
		(110, 'Robert Wilson', 31, 'robertwilson@example.com', '2020-02-20', '2023-01-20', 175.00);
END


truncate table [stg].Users
```

# Approach and methodologies used 
## Extraction and data conversion
- First, a csv was manually created to copy and paste data from the provided pdf document, and data was split in rows and columns in the csv file based on the delimiter. I'm aware this could have been done in SSIS, but in excel it was very simple to do.
- After data was extracted, before attempting to do any data type conversions, there was a few columns that needed to have some their format changed.
- Columns with format changing have derived columns with their fixed formatting. 
### Date Format
- It was clear that there's some fields in the RegistrationDate and LastLoginDate column that had a different format than the others.
- Most dates had the format YYYY-MM-DD, while a few had the format DD/MM/YYYY.
- While it was easy to send the rows that contained these fields back for review, a simple string manipulation solved this issue and generalized all the date format.
- This generalization converted all the columns for the dates to be in this format YYYYMMDD.
- Finally, the substrings YYYY, MM, DD were concatenated with '-' between them to have them in the correct format for DT_DBDATE conversion.
### UserID Format
- UserID as well only needed a simple string manipulation and data verification.
- I first checked if the LEN(UserID)>6, or else the UserID is only ''''''.
- If this check yields a true state, then Subtring will be processed to only extract the digits in the fields.
- However, later I realized that while most UserIDs have 3 digits, this won't always be the case if this ETL pipeline was processed for other data from same source, hence the Length check was only for length 6 since we can have the UserID starting from only 1 digit ( ex. 1,2,3, ---> 11,12,15 ---> 100,101,103).
### Email Format
- Verifying emails format was one of the challenges I had in this project.
- Emails do need to have a standard format which is local-part@domain.domainSuffix
- My Email verification was divided into the 2 steps.
- For sake of simplicity, since this is a DB from a client source, and most of their users email address will follow the same domain and domain suffix. Then the most suitable verification is to check that there's only 1 occurence of example.com.
- Further testing will be discussed after the data conversion step.

### Age verification
- Age verification only included 1 step.
- Step is casting it to Unsigned Integer, which will display negative age values as Null.

### FullName Format
- FullName validation involved only 1 step.
- A FullName can have one of 2 formats, either FirstName LastName, or FirstName MiddleName LastName.
- A space is expected between each name, hence I decided to do a TOKENCOUNT on FullName with character " ".
- The expected TokenCount for FullName is <=3 and >=2, in other words, only 3 tokens or 2 tokens are acceptable.

## Data Conversion
- Finally, after doing perliminary verification on the columns, I proceeded with the data conversion step to convert all the columns string to appropriate data types .
- Below is a list of the types each column data was casted to.
```
RegistrationDate --> DT_DBDATE
LastLoginDate --> DT_DBDATE
UserID --> DT_UI2 ( 2 byte unsigned integer)
Email --> DT_WSTR
Age --> DT_UI1 ( 1 byte unsigned integer)
FullName --> DT_WSTR
PurchaseTotal --> DT_R4 ( float).
```
- Only challenge I had with one of the data conversions is for PurchaseTotal.
- From a first glance at the data it would make sense to cast it into Decimal.
- However, since we are looking to have this ETL pipeline used when data is scaled, and cost is DB size and query running time, I decided to cast it into float which consumes less space.
- UserID was casted to 2 byte unsigned integer since it can't be a negative value, and can hold up to value 65,535. If scaling is done, we would need to go for a higher byte value.
- Age was casted to 1 byte unsigned integer since it can't be negative, and 1 byte holds up to value 255.

## Final verifications and checks

### Dates check
- Since we casted the dates to DT_DBDATE, any invalid dates can be detected by a NULL value.
- Last check was made to make sure that the LastLoginDate <= CurrentDate AND RegistrationDate >=2015.
- Since we don't have domain knowledge of the DB, a random value of 2015 was selected to verify RegistrationDate.
- 2015 can be replaced with any other year that would make sense specific to the scenario ( year sports team user DB was created, business launched, and etc...)

### PurchaseTotal Check
- This was the most challenging part of the ETL process.
- Since I don't have domain knowledge at all, I couldn't decide which range of PurchaseTotals would be acceptable.
- It was clear the value of 1000000 is suspicious, but it can also be an outlier.
- For simplicity I checked if PuchaseTotal<=1000, which was decided considering only the other values of PurchaseTotal.
- However, if this DB was to scale up, a more rigourous check would be needed to be done on PurchaseTotal, or domain knowledge would be needed in order to have a sufficient and acceptable evaluation.
- A statistical test ( Z-Score) could have been done on the PurchaseTotal column.
- Unfortunately, not enough data was provided, and I don't know if the scope of the position would allow me to perform statistical tests or data science techniques.

### Email reverification check
- As discussed above, another check for the email was done.
- derived column was processed which only extracts the local-part of the email addresses ( part before @).
- A C# Script component was written to validate that no localpart could contain a special character as the first character of the local part, nor the last character.
- Further, no special characters can occur consecutively.
```
char[] specialCharacters = "!#$%&'*+-/=?^_`{|}~.".ToCharArray();
public override void Input0_ProcessInputRow(Input0Buffer Row)
{
    string localPart = Row.Emaillocapart;
    bool isValid = true;

    if (specialCharacters.Contains(localPart[0]) || specialCharacters.Contains(localPart[localPart.Length - 1]))
    {
        isValid = false;
    }

    for (int i = 0; i < localPart.Length; i++)
    {
        if (specialCharacters.Contains(localPart[i]) && specialCharacters.Contains(localPart[i - 1])){
            isValid = false;
            break;
        }
    }

    Row.isValid = isValid;
}
```
### Null Checks
- Finally, a general null check was done on all the columns.
- If any field in any of the columns were to be NULL, the row would be redirected to be sent back for review.

## Rows Redirection
- At every step of every check, if a check fails, the rows for which the field the check was failed at would be redirected to a Union All component.
- The union all component is used to group all the rows that have failed one of the data checks, and these rows are Loaded into a csv file called Send_for_review.


## Duplicate checks and Incremental load
- To start off, since each user has a unique userID, hence userID is the most suitable column that can be used for duplicate checks.
- The staging table itself contained 3 rows that were duplicates( John Doe user ID 101), and one of them was redirected for review since it failed on of the checks.
- The other 2 rows were identified using Lookup component, for comparison with the production table values on column UserID.
- The rows without a match were redirected for insertion into the Production table, and the rows with a match were redirected to an OLE DB Command component for update.
- The 2 duplicated rows had different values for each registrationDate and PurchaseTotal.
- However, I decided to update the record in production based upon the newest registrationDate in the staging table.
- Below is the T-SQL script for record updating
```
UPDATE [prod].Users
SET
      FullName=src.FullName,
      Age= src.Age,
      Email=src.Email,
      RegistrationDate=src.RegistrationDate,
      LastLoginDate=src.LastLoginDate,
      PurchaseTotal=src.PurchaseTotal
FROM [prod].Users AS prod
JOIN(
    SELECT TOP 1*
    FROM [stg].Users
    ORDER BY RegistrationDate DESC
) AS src
ON prod.userID = src.UserID
```
- Since there was no domain knowledge provided, no aggregation was done on PurchaseTotal, but it could have been an option if requested by client to combine the new registeration and old registration purchaseTotal and SUM them.
- Finally, there could have been duplicated record in the staging table, for which I could have sorted on userID and removed duplicates. But since sorting is an expensive operation, and also dependent on whether or not there's hashing involved on this column, it was best to be avoided in this scenario.
