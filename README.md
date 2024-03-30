## Running SQL syntax in several DBs with the help of R

### Project overview
While working in several similar DBs utilized by different countries, it can get time-consuming to connect one database after the other to execute the same SQL query. While this may seem norm, advanced tools and their capabilities can bridge this gap and hence I utilized  R to iterate through several Databases to execute the SQL query. 

### Tools
- SQL
- R
- Microsoft Excel

### Prerequisite
1. Access to the concerned DBs
2. Listed DBs and their servers in an Excel file
3. Above mentioned tools

### Scripts
  #### SQL

```SQL
Select Distinct b.FileID,
  ha.ActivityTypeCodeOfActivity,
  b.ActiveInactive,
  b.StatusCode,
  b.ModifiedOn /*,
  b.ModifiedBy */
From b
  Left Join ha On b.beneficiary_id =
    ha.beneficiary_id
Where ha.TypeCode Is Null And
  b.StatusCode = 'Operational Data'
```
  #### R
  ``` R

rm(list = ls())
setwd("C:/Users/GN/Documents/R/R docs")

# load relevant packages
library(RODBC)
library(dplyr)
library(tidyverse)
library(readxl)
library(janitor)
library (ggplot2)


#load the TS file (need to download it each time we run)
DB_list <- read_excel("DB and servers.xlsx", sheet="DBDate")
# initialise empty tibble, to then rbind on all the DB extractions
DB_repr <- tibble()
#initialise index for iterations
i=1
#loop over DB list
for(database in DB_list$DB){
  
  # handles for DB connection (note the new server for upgraded Prot6)
  print(DB_list$ref_server[i]) #for debugging/info on progress
  if(DB_list$ref_server[i] != "NA"){
    
    #prepare the string for calling the server (use additional server column and check reporting DB
    db_connection_string <- paste0("driver={SQL Server};server=",DB_list$ref_server[i],".GIN.ikuro.priv;database=",database,"_Reporting;ApplicationIntent=READONLY;Integrated Security=SSPI")
    dbhandle <- odbcDriverConnect(db_connection_string, readOnlyOptimize = T)
    
    # read query from external .sql file
    query_repr <- readr::read_file("nameofsqlfile.sql") 
    
    # build the tibble from our connection and query
    DB_repr_i <- tibble (sqlQuery(dbhandle, query_repr, stringsAsFactors=FALSE))
    #close DB connection
    odbcClose(dbhandle)
    
    #add a column with the DB name
    DB_repr_i$DB <- database
    print(database) #for debugging/info
    print(DB_repr_i) #for debugging/info
    #populate consolidated file
    DB_repr <- rbind(DB_repr, DB_repr_i)
  }
  else{
    
    print(paste0("not checked since NA: ",database)) #debugging/info for skipped databases
  }
  #increment index for next iteration
  i <- i+1
}


NoAT_T <- DB_repr %>% group_by(DB) %>% summarise(UniqueSystemNrOfActivity=n())
NoAT_T<-NoAT_T [order(NoAT_T$UniqueSystemNrOfActivity),] 
View(NoAT_T)


ggplot(NoAT_T, aes(x = reorder(DB,UniqueSystemNrOfActivity) , y=UniqueSystemNrOfActivity, fill=DB)) +
  geom_bar( stat = "identity")+geom_text(aes(label=UniqueSystemNrOfActivity))+
  labs(title="Put the relevant title", x="DB", y="No. of Beneficiaries")+ coord_flip()
```
## Output
<img width="312" alt="Capture" src="https://github.com/ikuro/R--SQL-data-cleaning-controls/assets/14724683/a5b373d5-d6f8-42da-9170-abacfa7dca09">

