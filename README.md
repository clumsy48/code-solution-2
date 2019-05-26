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
     
     - Apis: getDataforbyHour  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount of current hour vs last hour
     - Apis: getDataforbyday   : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current day vs last day
     - Apis: getDataforByweek  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current week vs last week
     - Apis: getDataforByMonth : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current month vs last month
     - Apis: getDataforByYear  : Merchant Id , Page Url  Response : TotalUserCount , newUserCount , oldUserCount current year vs last year
    
    operational-data-collector-service (to reprocess history in case of failure)
    
    - Model         : Publisher-Subcsriber Model (Useful in case of reprocess)
    - Incoming-Data : Merchant Id , Page Url , UserInfo (message published by rt-message-publisher-service)
    - Purpose       : Migrate the old data to new database such every insertion in new the databse will generate an event to download that entry   from table 
    and reprocess with fixed code plus simultaneously running to process the new data.
    when we receive an event to reprocess the old data which is already processed , we will switch the realtime Data-Collector-Service to 
    point to the new datadase.
   
System-Architecture:




