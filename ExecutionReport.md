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
