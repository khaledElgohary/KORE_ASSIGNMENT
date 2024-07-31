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
- Make sure to disable the Initialize DB component in the control flow to prevent running script twice 

### Running the DB initialization using the Execute SQL Task component.
- By default, I have an SQL Task component that is responsible for the Initialization of the DB in the control flow.
- Right click on the component and click on edit.
- Make sure under SQL Statement that the Connection is corresponding with your SQL server connection, and that your connection type is OLE DB

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
- First, a csv was manually created to copy and paste data from the provided pdf document, and data was split in rows and columns in the csv file based on the delimiter. I'm aware this could have been done in SSIS, but it in excel it was very simple to do.
- After data was extracted, before attempting to do any data type conversions, there was a few columns that needed to have some their format changed.
- Columns with format changing were have derived columns with their fixed formatting. 
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
- However, later I realized that while most UserIDs have 3 digits, this won't always be the case this ETL pipeline was processed for other data from same source, hence the Length check was only for length 6 since we can have the UserID starting from only 1 digit ( ex. 1,2,3, ---> 11,12,15 ---> 100,101,103).
### Email Format
- Verifying emails format was one of the challenges I had in this project.
- Emails do need to have a standard format which is local-part@domain.
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
- To start off, since each user has a unique userID, hence userID is the most suitable column that can be used for dublicate checks.
- The staging table itself contained 3 rows that were dublicates( John Doe user ID 101), and one of them was redirected for review since it failed on of the checks.
- The other 2 rows were identified using Lookup component, for comparison with the production table values on column UserID
- The rows without a match were redirected for insertion into the Production table, and the rows with a match were redirected to and OLE DB Command component for update.
- The 2 duplicated rows had different values for each registrationDate and PurchaseTotal.
- However, I decided to the record in production based upon the newest registrationDate in the staging table.
- Since there was no domain knowledge provided, no aggregation was done on PurchaseTotal, but it could have been an option if requested by client to combine the new registeration and old registration purchaseTotal and SUM them.
- Finally, there could have been duplicated record in the staging table, for which I could have sorted on userID and removed duplicates. But since sorting is an expensive operation, and also dependent on whether or not there's hashing involved on this column, it was best to be avoided in this scenario.


# Summary of Results
## Rows Extracted versus Loaded
- 33 rows were extracted from staging table.
- 15 rows were excluded due to data quality issues and were sent back for review.
- 18 rows extracted from staging table before duplication resolving and loading into production.
- 2 rows reviewed for duplication.
- 16 rows inserted into production table
- 1 row updated in production table.
- Total of 26 rows in production table after ETL pipeline exeuction.

## Rows Excluded
- UserID 111 excluded for missing Age, LastLoginDate, and PurchaseTotal.
- UserID 112 excluded for missing PurchaseTotal.
- Null User excluded for missing UserID and Email.
- UserID 115 excluded for missing Age and PurchaseTotal.
- UserID 121 excluded for missing Age and PurchaseTotal.
- userID 130 excluded for invalid date.
- userID 134 excluded for having a negative age.
- userID 133 excluded for having LastLoginDate>currentDate.
- userID 135 excluded for having invalid (very old) RegistrationDate.
- userID 101 John Doe ( 1 of 3 duplicates) excluded for having RegistrationDate>LastLoginDate.
- userID 113 excluded for missing Age.
- userID 137 excluded for having invalid email address.
- userID 136 excluded for OutOfBound PurchaseTotal.
- userID 131 excluded for invalid email address ( consecutive special characters in email local-part).

## Rows that should be excluded
- As discussed above, it would have been easier to exclude the old date format, but since the string manipulation was easier due to the fact the csv data types were generalized to string, I made the decision to include it and not exclude it.
- UserID 138 should have had a matching ID due to it's name. However, there were no matches found for this userID and hence it was not excluded.

## Challenges Faced
- Rigorous check on email address was needed for safe validation of the email format.
- PurchaseTotal large value exclusion took a very long decision time for me, the value being an outlier wasn't sufficient for the exclusion, but in comparison with other values of PurchaseTotal it would suffice to include it as invalid. However, with enough data for statistical tests and more domain knowledge it would have been easier to make this exlusion decision.
- PurchaseTotal data type conversion was challenging to decide, knowing that the database could scale to a larger one made the decision easier by optioning for the smaller size data type ( 4 bytes vs 5 bytes). However, float is less precise than decimal which also works well for the purpose of the PurchaseTotal data.
- FullName check wasn't challenge but involved a lot of thinking. The safer option would be to assume that the user can have the option to input their middle name, which led me to use the criteria of TOKENCOUNT==3 or 2.
- For the purpose of this inteview process, i was not sure if splitting the columns in Excel versus SSIS was acceptable or not. But since both of them are easily achievable i decided to split the data in excel before extracting them in SSIS.
