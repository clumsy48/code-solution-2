# code-solution-2
## HLD for Google Analytics System

**Assunptions:**
1) No. of Active clients + clients added in next 10 years = 100,000.
2) Users of every Client + Users added in next years = 100,00,000.
3) User from same location visted multiple times within 1 min is counted as single visit.
4) 10% users of every client active wihtin a min.

**Capacity Estimation and Constraints:**
1) Traffic estimates - 1000,00,00,00,00,000 / 1440 ~ 100,00,00,000 per minutes .
2) Storage estimates : 360tb (Defined in Database design section)

**Features:**
1) Count of users visted a page by minute,day,week,month,year.
2) Count of users by region , Browser , operating sytem, Gender , Age group (0-10,11-17,18-30,31-40,above 40).
3) Count of new users vs existing users.

**Database Design :**
`Since we are dealing with large volumes/scale of data and it needs to be highly available , therefore NoSql DyanmoDb will be ideal
choice for it (there are other nosql database also available like Apache Cassandra).`

Write per day ~ 1 billion

- Size of Databse in one month ~ 30 billion records.
- Size of databse in one year ~ 360 billion records.
- Size of database in 10 years ~ 3600 billion records.

Tables :
Master_Table  (To store minutes basis records coming from various merchants) 
Columns: 
- Localdatetime (HashKey) :String  format (yyyy-MM-ddHH:mm)
- ClientPageID (Sort Key) :String (combination of ClientId + PageUrl(in encoded form))
- UserInfo :String

> UserInfo is Json String having data
> UserId:String , Region (Country:String ,City:String) ,Operating_System:String , Browser:String ,Age : Integer.

Size of each row :
> 15 bytes(LocalDatetime) + 20 bytes (Considering CLientId is 10 chars at most + PageId 10 chars at most ) 
> 30 bytes (UserId) + 10 bytes (Country) + 10 bytes (City) + 10 bytes (Operating System) + 10 bytes (Browser) + 2 bytes (Age) = 107 bytes

> Size of table in in one day ~ 1000,00,000 * 107 bytes ~= 100gb / day.
> in next 10 year ~= 360tb

**Component Design:**

     rt-message-publisher-service
     
     - Incoming-Data : Merchant Id , Page Url , UserInfo (post request by Merchant)
     - Purpose       : parse incoming post data to suitable message and publish to Notification service (using aws SNS here)
     - Flow diagram  : https://github.com/clumsy48/code-solution-2/blob/master/rt-message-publisher-service.PNG
     
     rt-data-collector-service
     
     - Model         : Publisher-Subcsriber Model (Useful in case of reprocess)
     - Incoming-Data : Merchant Id , Page Url , UserInfo (message published by rt-message-publisher-service)
     - Purpose       : collect data comming from various merchants ,parses it and stores in DB
     - Scaling       : horizontal scaling is a bad idea , since there are so many data coming update database from various instances will
                       lead to data inconsistency.
                       So ,vertical scaling will the preferred choice .
     - Processing    : Running Single thread will need to process  1000,00,000 / 24*60*60 request per second ~= 1158 requests / second.                 
     - Flow diagram  : https://github.com/clumsy48/code-solution-2/blob/master/rt-data-collector-service.PNG
     
     Data-Retriever-Service:
     - Scaling       : horizontal scaling will suite for this , since we not updating anything in the db.
                       Scaling parameters : 
                        1) memory.
                        2) requests.
     - Processing    : In a sec, we can max of 100,000 requests . 
                       Using single thread , we will need to process 100,000 / sec.
                       By running 10 threads , we can process 10,000 /sec.
                       By scaling horizontly , suppose we use 10 instances we can reduce processing request per second to 1000 /sec.
     - Purpose       : Fetch data from dynamo db based on api fired , process and return to Merchant.
     - Flow diagram  : https://github.com/clumsy48/code-solution-2/blob/master/rt-data-collector-service.PNG
     - Apis:
      1) getDatabyHour  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount of current hour vs last hour
      2) getDatabyday   : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current day vs last day
      3) getDataByweek  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current week vs last week
      4) getDataByMonth : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current month vs last month
      5) getDataByYear  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current year vs last year
      6) getDatabyHourAndCity  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount of current hour vs last hour based on City
      7) getDatabydayAndCity   : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current day vs last day
      8) getDataByweekAndCity  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current week vs last week based on City
      9) getDataByMonthAndCity : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current month vs last month based on City
      10) getDataByYearAndCity  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current year vs last year based on City
      11) getDatabyHourAndCountry  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount of current hour vs last hour based on Country
      12) getDatabydayAndCountry   : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current day vs last day
      13) getDataByweekAndCountry  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current week vs last week based on Country
      14) getDataByMonthAndCountry : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current month vs last month based on Country
      15) getDataByYearAndCountry  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current year vs last year based on Country
      
    operational-data-collector-service (to reprocess history in case of failure)
    
     - Model         : Publisher-Subcsriber Model (Useful in case of reprocess)
     - Incoming-Data : Merchant Id , Page Url , UserInfo (message published by rt-message-publisher-service)
     - Purpose       : Migrate the old data to new database such every insertion in new the databse will generate an event to download                          that entry from table and reprocess with fixed code plus simultaneously running to process the new data.
                       when we receive an event to reprocess the old data which is already processed , we will switch the rt-Data-                              collector-Service to point to the new datadase.
                       Invalidate Cache.
   
**Future Scope:**
1) Provide Info on count of users spedning time on a particular page.
2) Implement Cache to improve latency of response of Apis. 

