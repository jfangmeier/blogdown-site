---
title: Introducing the claimsdb package for teaching and learning
author: Josh Fangmeier
date: '2022-02-12'
slug: []
categories: []
tags:
  - health care
  - claims
  - Medicare
  - databases
toc: no
images: ~
---



![diagram](README-diagram.PNG)

In the fields of health informatics and health services research, health insurance claims data are a valuable source to answer questions about health care access and financing. However, claims data in the real world often contains both sensitive (protected health information) and proprietary (trade secrets) elements. For most students and educators seeking opportunities to learn how to use claims data, there are few available sources for practice.

To help with this problem, [__claimsdb__](https://github.com/jfangmeier/claimsdb) 📦 provides easy access to a sample of health insurance enrollment and claims data from the [Centers for Medicare and Medicaid Services (CMS) Data Entrepreneurs’ Synthetic Public Use File (DE-SynPUF)](https://www.cms.gov/Research-Statistics-Data-and-Systems/Downloadable-Public-Use-Files/SynPUFs/DE_Syn_PUF), as a set of relational tables or as an in-memory database using [DuckDB](https://duckdb.org/). All the data is contained within a single package, so users do not need to download any external data. This package is inspired by and based on the [starwarsdb](https://github.com/gadenbuie/starwarsdb) package.

The data are structured as actual Medicare claims data but are fully “synthetic,” after a process of alterations meant to reduce the risk of re-identification of real Medicare beneficiaries. The synthetic process that CMS adopted changes the co-variation across variables, so analysts should be cautious about drawing inferences about the actual Medicare population.

The data included in claimsdb comes from 500 randomly selected 2008 Medicare beneficiaries from Sample 2 of the DE-SynPUF, and it includes all the associated claims for these members for 2008-2009. CMS provides [resources](https://www.cms.gov/Research-Statistics-Data-and-Systems/Downloadable-Public-Use-Files/SynPUFs/DESample02), including a codebook, FAQs, and other documents with more information about this data. 

This post covers the following topics: 
- Installation of the claimsdb package
- How to setup a remote database with the included data
- Examples of how to use claimsdb for analysis
- An overview of the limitations of the data included in package

## Installation and components of the *claimsdb* package

You can install the development version of claimsdb from GitHub with:


```r
# install.packages("devtools")
devtools::install_github("jfangmeier/claimsdb")
```

You can then load claimsdb alongside your other packages. We will be using the *tidyverse* packages in our later examples.


```r
library(tidyverse)
library(claimsdb)
```

The tables are available after loading the claimsdb package. This includes a `schema` that describes each of the tables and the included variables from the [CMS DE-SynPUF](https://www.cms.gov/Research-Statistics-Data-and-Systems/Downloadable-Public-Use-Files/SynPUFs/DE_Syn_PUF). You can see that claimsdb includes five tables. One of the tables, *bene* contains beneficiary records, while the others include specific types of claims data.


```r
schema
```

```
## # A tibble: 5 x 5
##   TABLE      TABLE_TITLE              TABLE_DESCRIPTI~ UNIT_OF_RECORD PROPERTIES
##   <chr>      <chr>                    <chr>            <chr>          <list>    
## 1 bene       CMS Beneficiary Summary~ Synthetic Medic~ Beneficiary    <tibble>  
## 2 carrier    CMS Carrier Claims DE-S~ Synthetic physi~ claim          <tibble>  
## 3 inpatient  CMS Inpatient Claims DE~ Synthetic inpat~ claim          <tibble>  
## 4 outpatient CMS Outpatient Claims D~ Synthetic outpa~ claim          <tibble>  
## 5 pde        CMS Prescription Drug E~ Synthetic Part ~ claim          <tibble>
```

You can access details on the variables in each of the tables like in this example with the *inpatient* table. You can see that this table contains 35 fields, including a beneficiary code to identify members across tables as well as detailed information on each inpatient claim.


```r
schema %>% 
  filter(TABLE == "inpatient") %>% 
  pull(PROPERTIES)
```

```
## [[1]]
## # A tibble: 35 x 3
##    VARIABLE                 TYPE    DESCRIPTION                                 
##    <chr>                    <chr>   <chr>                                       
##  1 DESYNPUF_ID              string  Beneficiary Code                            
##  2 CLM_ID                   string  Claim ID                                    
##  3 SEGMENT                  numeric Claim Line Segment                          
##  4 CLM_FROM_DT              date    Claims start date                           
##  5 CLM_THRU_DT              date    Claims end date                             
##  6 PRVDR_NUM                string  Provider Institution                        
##  7 CLM_PMT_AMT              numeric Claim Payment Amount                        
##  8 NCH_PRMRY_PYR_CLM_PD_AMT numeric NCH Primary Payer Claim Paid Amount         
##  9 AT_PHYSN_NPI             string  Attending Physician National Provider Ident~
## 10 OP_PHYSN_NPI             string  Operating Physician National Provider Ident~
## # ... with 25 more rows
```

## Access claims data as a remote database

Many organizations store claims data in a remote database, so claimsdb also includes all of the tables as an in-memory DuckDB database. This can be a great way to practice working with this type of data, including building queries with dplyr code using dbplyr.

To setup the in-memory database, you need to create a database connection using `claims_connect()` and create connections to each of the tables you want to use.


```r
library(dbplyr)

# Setup connection to duckDB database
con <- claims_connect()

# Setup connections to each of the enrollment and claims tables in the database
bene_rmt <- tbl(con, "bene")
inpatient_rmt <- tbl(con, "inpatient")
outpatient_rmt <- tbl(con, "outpatient")
carrier_rmt <- tbl(con, "carrier")
pde_rmt <- tbl(con, "pde")
```

You can then preview your connection to each of the remote tables.


```r
# Preview the prescription drug event table in the database
pde_rmt
```

```
## # Source:   table<pde> [?? x 8]
## # Database: duckdb_connection
##    DESYNPUF_ID      PDE_ID SRVC_DT    PROD_SRVC_ID QTY_DSPNSD_NUM DAYS_SUPLY_NUM
##    <chr>            <chr>  <date>     <chr>                 <dbl>          <dbl>
##  1 00E040C6ECE8F878 83014~ 2008-12-20 49288010404              30             30
##  2 00E040C6ECE8F878 83594~ 2009-04-25 52959037200              20             30
##  3 00E040C6ECE8F878 83144~ 2009-09-22 00083400141              80             30
##  4 00E040C6ECE8F878 83614~ 2009-10-03 63481043301              60             10
##  5 00E040C6ECE8F878 83014~ 2009-11-16 51129404301              60             30
##  6 00E040C6ECE8F878 83234~ 2009-12-11 58016096489             270             30
##  7 014F2C07689C173B 83294~ 2009-09-14 00904582062              90             30
##  8 014F2C07689C173B 83874~ 2009-10-11 59604053055              40             20
##  9 014F2C07689C173B 83314~ 2009-11-24 51079017760              30             30
## 10 014F2C07689C173B 83874~ 2009-12-29 51129370802              30             30
## # ... with more rows, and 2 more variables: PTNT_PAY_AMT <dbl>,
## #   TOT_RX_CST_AMT <dbl>
```

## Examples of using *claimsdb* for analysis

To analyze and explore the claims and beneficiary data, you can execute SQL code on the database using the `DBI` package.


```r
DBI::dbGetQuery(
  con, 
  paste0(
    'SELECT "DESYNPUF_ID", "BENE_BIRTH_DT"',
    'FROM "bene"',
    'WHERE "BENE_SEX_IDENT_CD" = 1.0',
    'LIMIT 10'
  )
)
```

```
##         DESYNPUF_ID BENE_BIRTH_DT
## 1  001115EAB83B19BB    1939-12-01
## 2  03ADA78C0FEF79F4    1923-02-01
## 3  040A12AB5EAA444C    1943-10-01
## 4  0507DE00BC6E6CD6    1932-07-01
## 5  05672CCCED56BCAD    1937-07-01
## 6  060CDE3A044F64BA    1943-02-01
## 7  061B5B3D9A459675    1932-04-01
## 8  08C8E0A0C6EAC884    1954-06-01
## 9  09EEB5C4C4FAEF10    1935-04-01
## 10 0A58C6D6B9BE67CF    1942-07-01
```

However, in the following examples, we will use the `dbplyr` package which translates dplyr code into SQL that can be executed against the database. You can see that the results below with dplyr functions match the results above that used a SQL query.


```r
bene_rmt %>% 
  filter(BENE_SEX_IDENT_CD == 1) %>% 
  select(DESYNPUF_ID, BENE_BIRTH_DT)
```

```
## # Source:   lazy query [?? x 2]
## # Database: duckdb_connection
##    DESYNPUF_ID      BENE_BIRTH_DT
##    <chr>            <date>       
##  1 001115EAB83B19BB 1939-12-01   
##  2 03ADA78C0FEF79F4 1923-02-01   
##  3 040A12AB5EAA444C 1943-10-01   
##  4 0507DE00BC6E6CD6 1932-07-01   
##  5 05672CCCED56BCAD 1937-07-01   
##  6 060CDE3A044F64BA 1943-02-01   
##  7 061B5B3D9A459675 1932-04-01   
##  8 08C8E0A0C6EAC884 1954-06-01   
##  9 09EEB5C4C4FAEF10 1935-04-01   
## 10 0A58C6D6B9BE67CF 1942-07-01   
## # ... with more rows
```

We can use the `show_query()` function to see the SQL code that dbplyr created that it closely matches the SQL code above.


```r
bene_rmt %>% 
  filter(BENE_SEX_IDENT_CD == 1) %>% 
  select(DESYNPUF_ID, BENE_BIRTH_DT) %>% 
  show_query()
```

```
## <SQL>
## SELECT "DESYNPUF_ID", "BENE_BIRTH_DT"
## FROM "bene"
## WHERE ("BENE_SEX_IDENT_CD" = 1.0)
```

### First, a note about working with dates/times in databases

dbplyr is an amazing tool for working with databases, especially if you want to use many functions from the dplyr and tidyr packages. However, it does not currently have SQL translations for all functions in the tidyverse family of packages. For example, the `lubridate` package's date and time functions work on local dataframes but cannot be translated to work on remote tables at this time. In the example below, you can see that the lubridate function for parsing the year from a date works on the local dataframe but generates an error on the remote table with the same data.


```r
bene %>% 
  transmute(
    DESYNPUF_ID,
    BENE_BIRTH_DT,
    BIRTH_YEAR = lubridate::year(BENE_BIRTH_DT)
  )
```

```
## # A tibble: 998 x 3
##    DESYNPUF_ID      BENE_BIRTH_DT BIRTH_YEAR
##    <chr>            <date>             <dbl>
##  1 001115EAB83B19BB 1939-12-01          1939
##  2 0018A1975BC0EE4F 1935-11-01          1935
##  3 00E040C6ECE8F878 1932-11-01          1932
##  4 014F2C07689C173B 1935-07-01          1935
##  5 029A22E4A3AAEE15 1921-05-01          1921
##  6 03410742A11DDC52 1943-07-01          1943
##  7 03ADA78C0FEF79F4 1923-02-01          1923
##  8 040A12AB5EAA444C 1943-10-01          1943
##  9 043AAAE41C9A37B7 1924-03-01          1924
## 10 0507DE00BC6E6CD6 1932-07-01          1932
## # ... with 988 more rows
```

```r
bene_rmt %>% 
  transmute(
    DESYNPUF_ID,
    BENE_BIRTH_DT,
    BIRTH_YEAR = lubridate::year(BENE_BIRTH_DT)
  )
```

```
## Error in lubridate::year(BENE_BIRTH_DT): object 'BENE_BIRTH_DT' not found
```

Fortunately, dbplyr allows you pass along SQL functions in your dplyr code, and it will include these functions in the generated query. For date/time functions, we need to consult the [documentation](https://duckdb.org/docs/sql/functions/date) from DuckDB on date functions. To parse a part of a date (e.g., the year), we need to use the `date_part()` function for DuckDB. 

The function to do this task may vary across database backends, so if you are doing this with a different database (Oracle, SQL Server, etc.), you will need to read the documentation for that database management system.


```r
bene_rmt %>% 
  transmute(
    DESYNPUF_ID,
    BENE_BIRTH_DT,
    BIRTH_YEAR = date_part('year', BENE_BIRTH_DT)
  )
```

```
## # Source:   lazy query [?? x 3]
## # Database: duckdb_connection
##    DESYNPUF_ID      BENE_BIRTH_DT BIRTH_YEAR
##    <chr>            <date>             <dbl>
##  1 001115EAB83B19BB 1939-12-01          1939
##  2 0018A1975BC0EE4F 1935-11-01          1935
##  3 00E040C6ECE8F878 1932-11-01          1932
##  4 014F2C07689C173B 1935-07-01          1935
##  5 029A22E4A3AAEE15 1921-05-01          1921
##  6 03410742A11DDC52 1943-07-01          1943
##  7 03ADA78C0FEF79F4 1923-02-01          1923
##  8 040A12AB5EAA444C 1943-10-01          1943
##  9 043AAAE41C9A37B7 1924-03-01          1924
## 10 0507DE00BC6E6CD6 1932-07-01          1932
## # ... with more rows
```

### Example 1: *which members had the highest prescription drug costs for 2008?*

For this first example, we are going to identify the beneficiaries with the highest total prescription drug costs in 2008. We need to use the "pde" table that has claims on prescription drug events and the "bene" table that has beneficiary records. We create an object that is the aggregated costs for prescription drugs at the beneficiary level in 2008. Note that we had to use `date_part()` to parse the year from the service date. 


```r
# Calculate rx costs for utilizing members in 2008
rx_costs_rmt <- pde_rmt %>% 
  mutate(BENE_YEAR = date_part('year', SRVC_DT)) %>% 
  filter(BENE_YEAR == 2008) %>% 
  group_by(BENE_YEAR, DESYNPUF_ID) %>% 
  summarize(TOTAL_RX_COST = sum(TOT_RX_CST_AMT, na.rm = T), .groups = "drop") %>% 
  ungroup()

rx_costs_rmt
```

```
## # Source:   lazy query [?? x 3]
## # Database: duckdb_connection
##    BENE_YEAR DESYNPUF_ID      TOTAL_RX_COST
##        <dbl> <chr>                    <dbl>
##  1      2008 00E040C6ECE8F878            10
##  2      2008 03ADA78C0FEF79F4          5020
##  3      2008 040A12AB5EAA444C           120
##  4      2008 043AAAE41C9A37B7           810
##  5      2008 05672CCCED56BCAD             0
##  6      2008 061B5B3D9A459675           270
##  7      2008 08BB74BA9DFD5C06            20
##  8      2008 08C8E0A0C6EAC884          5860
##  9      2008 09EEB5C4C4FAEF10          1590
## 10      2008 0A58C6D6B9BE67CF           930
## # ... with more rows
```

Then we join the aggregated cost data to the beneficiary table. This is necessary because the "pde" table does not include beneficiaries who didn't use any prescription drugs. After joining the table we reassign missing cost data to zero for those beneficiaries with no utilization. We can then use `collect()` to bring the results into a local dataframe.


```r
# Join the rx costs data to the beneficiary file and include members with no costs
rx_bene_rmt <- bene_rmt %>% 
  filter(BENE_YEAR == 2008) %>% 
  select(
    BENE_YEAR,
    DESYNPUF_ID
  ) %>% 
  left_join(
    rx_costs_rmt, by = c("BENE_YEAR", "DESYNPUF_ID")
  ) %>% 
  mutate(TOTAL_RX_COST = ifelse(is.na(TOTAL_RX_COST), 0, TOTAL_RX_COST)) %>% 
  arrange(desc(TOTAL_RX_COST))

rx_bene_rmt
```

```
## # Source:     lazy query [?? x 3]
## # Database:   duckdb_connection
## # Ordered by: desc(TOTAL_RX_COST)
##    BENE_YEAR DESYNPUF_ID      TOTAL_RX_COST
##        <dbl> <chr>                    <dbl>
##  1      2008 484FAAB99C7F90F2         10410
##  2      2008 1187122DCAD04B48          9300
##  3      2008 5199E41C3B2B36AD          9110
##  4      2008 801D1702FC0C105F          8210
##  5      2008 92158DA24D7A8799          7910
##  6      2008 A705B8508EB5747D          7380
##  7      2008 A9142A5D9895479C          6780
##  8      2008 D20C21DD9F8177D0          6590
##  9      2008 08C8E0A0C6EAC884          5860
## 10      2008 EBE9CC1D92AA56C0          5690
## # ... with more rows
```

```r
# You can use collect() to bring the results into a local dataframe
rx_bene_df <- rx_bene_rmt %>% 
  collect()

rx_bene_df
```

```
## # A tibble: 500 x 3
##    BENE_YEAR DESYNPUF_ID      TOTAL_RX_COST
##        <dbl> <chr>                    <dbl>
##  1      2008 484FAAB99C7F90F2         10410
##  2      2008 1187122DCAD04B48          9300
##  3      2008 5199E41C3B2B36AD          9110
##  4      2008 801D1702FC0C105F          8210
##  5      2008 92158DA24D7A8799          7910
##  6      2008 A705B8508EB5747D          7380
##  7      2008 A9142A5D9895479C          6780
##  8      2008 D20C21DD9F8177D0          6590
##  9      2008 08C8E0A0C6EAC884          5860
## 10      2008 EBE9CC1D92AA56C0          5690
## # ... with 490 more rows
```

### Example 2: *what percent of beneficiaries received an office visit within 30 days of discharge from a hospital?*

In the next example, we are identifying which beneficiaries had an office visit within 30 days of being discharged. We will start with the "inpatient" table that contains records for all inpatient stays, including when a beneficiary was discharged. We create an object that includes the beneficiary ID, the discharge date, and the date 30 days after discharge. Note that for DuckDB we need to coerce "30" to an integer to calculate the new date.


```r
# Identify all member discharge dates and the dates 30 days after discharge from the inpatient table
ip_discharges <- inpatient_rmt %>% 
  transmute(
    DESYNPUF_ID, 
    DSCHRG_DT = NCH_BENE_DSCHRG_DT,
    DSCHRG_DT_30 = NCH_BENE_DSCHRG_DT + as.integer(30)) %>% 
  distinct()

ip_discharges
```

```
## # Source:   lazy query [?? x 3]
## # Database: duckdb_connection
##    DESYNPUF_ID      DSCHRG_DT  DSCHRG_DT_30
##    <chr>            <date>     <date>      
##  1 014F2C07689C173B 2009-09-20 2009-10-20  
##  2 029A22E4A3AAEE15 2008-01-25 2008-02-24  
##  3 060CDE3A044F64BA 2008-10-18 2008-11-17  
##  4 08BB74BA9DFD5C06 2009-03-07 2009-04-06  
##  5 08BB74BA9DFD5C06 2009-03-08 2009-04-07  
##  6 08C8E0A0C6EAC884 2008-02-22 2008-03-23  
##  7 08C8E0A0C6EAC884 2009-05-28 2009-06-27  
##  8 08C8E0A0C6EAC884 2009-06-14 2009-07-14  
##  9 08C8E0A0C6EAC884 2009-07-07 2009-08-06  
## 10 08C8E0A0C6EAC884 2009-08-23 2009-09-22  
## # ... with more rows
```

Next, we need to identify office visits from the "carrier" table. I created a vector of five office visit codes for this example. Since these code must match the values in the "HCPCS" columns, we reshape the table with `pivot_longer()` then filter for the office visit codes.


```r
# Create vector of office visit codes
off_vis_cds <- as.character(99211:99215)

# Identify members who had an office visit from the carrier table
off_visit <- carrier_rmt %>% 
  select(DESYNPUF_ID, CLM_ID, CLM_FROM_DT, matches("HCPCS")) %>% 
  pivot_longer(cols = matches("HCPCS"), names_to = "LINE", values_to = "HCPCS") %>% 
  filter(HCPCS %in% off_vis_cds) %>% 
  distinct(DESYNPUF_ID, CLM_FROM_DT) %>% 
  mutate(OFFICE_VISIT = 1)

off_visit
```

```
## # Source:   lazy query [?? x 3]
## # Database: duckdb_connection
##    DESYNPUF_ID      CLM_FROM_DT OFFICE_VISIT
##    <chr>            <date>             <dbl>
##  1 00E040C6ECE8F878 2009-01-17             1
##  2 00E040C6ECE8F878 2009-07-20             1
##  3 00E040C6ECE8F878 2009-07-26             1
##  4 029A22E4A3AAEE15 2008-01-25             1
##  5 029A22E4A3AAEE15 2008-01-30             1
##  6 029A22E4A3AAEE15 2008-08-28             1
##  7 029A22E4A3AAEE15 2008-11-19             1
##  8 029A22E4A3AAEE15 2009-04-01             1
##  9 029A22E4A3AAEE15 2009-05-13             1
## 10 029A22E4A3AAEE15 2009-10-02             1
## # ... with more rows
```

Finally, we join the office visits object to the discharges object. We can use the `sql_on` option in `left_join()` to inject some custom SQL to join when the office visit date is within 30 days of the discharge date.


```r
# Join discharges data and office visit data
discharge_offvis <- 
  ip_discharges %>% 
  left_join(
    off_visit,
    sql_on = paste0(
      "(LHS.DESYNPUF_ID = RHS.DESYNPUF_ID) AND",
      "(RHS.CLM_FROM_DT >= LHS.DSCHRG_DT) AND",
      "(RHS.CLM_FROM_DT <= LHS.DSCHRG_DT_30)"
      )
  )

discharge_offvis
```

```
## # Source:   lazy query [?? x 6]
## # Database: duckdb_connection
##    DESYNPUF_ID.x  DSCHRG_DT  DSCHRG_DT_30 DESYNPUF_ID.y CLM_FROM_DT OFFICE_VISIT
##    <chr>          <date>     <date>       <chr>         <date>             <dbl>
##  1 029A22E4A3AAE~ 2008-01-25 2008-02-24   029A22E4A3AA~ 2008-01-25             1
##  2 029A22E4A3AAE~ 2008-01-25 2008-02-24   029A22E4A3AA~ 2008-01-30             1
##  3 060CDE3A044F6~ 2008-10-18 2008-11-17   060CDE3A044F~ 2008-11-01             1
##  4 1187122DCAD04~ 2009-04-06 2009-05-06   1187122DCAD0~ 2009-04-07             1
##  5 1187122DCAD04~ 2009-04-06 2009-05-06   1187122DCAD0~ 2009-04-27             1
##  6 1187122DCAD04~ 2009-04-06 2009-05-06   1187122DCAD0~ 2009-05-04             1
##  7 12D6FF0C18764~ 2009-05-08 2009-06-07   12D6FF0C1876~ 2009-05-19             1
##  8 12D6FF0C18764~ 2009-05-08 2009-06-07   12D6FF0C1876~ 2009-05-24             1
##  9 144C187653FBB~ 2008-06-08 2008-07-08   144C187653FB~ 2008-07-04             1
## 10 1E14EA81B43B5~ 2009-12-06 2010-01-05   1E14EA81B43B~ 2009-12-20             1
## # ... with more rows
```

After the join is complete, we can calculate the share of discharges with a timely office visit.


```r
# Find the percent of members with a office visit within 30 days of discharge
discharge_offvis %>% 
  distinct(
    DESYNPUF_ID.x,
    DSCHRG_DT,
    OFFICE_VISIT
  ) %>% 
  mutate(OFFICE_VISIT = ifelse(is.na(OFFICE_VISIT), 0, OFFICE_VISIT)) %>% 
  summarize(
    OFV_RATE = mean(OFFICE_VISIT, na.rm = T)
  )
```

```
## # Source:   lazy query [?? x 1]
## # Database: duckdb_connection
##   OFV_RATE
##      <dbl>
## 1    0.540
```

### Example 3: *how well are beneficiaries filling their hypertension drug prescriptions?*

For this final example, we need to identify hypertension medications within the "pde" table and calculate medication adherence rates for each beneficiary on a medication. To isolate the hypertension medications, we can borrow from the HEDIS medication list for ACE inhibitors and ARB medications (which are commonly used to treat hypertension). This medication list is for 2018/2019, so it likely includes new drugs that did not exist in 2008/2009 and may not include older drugs that are no longer used (ideally it's best to use external lists that match the time frame of the claims data).


```r
hedis_wb <- tempfile()

download.file(
  url = "https://www.ncqa.org/hedis-2019-ndc-mld-directory-complete-workbook-final-11-1-2018-3/",
  destfile = hedis_wb,
  mode = "wb"
)

hedis_df <- readxl::read_excel(
  path = hedis_wb,
  sheet = "Medications List to NDC Codes"
)

hyp_ndc_df <- 
  hedis_df %>% 
  filter(`Medication List` == "ACE Inhibitor/ARB Medications") %>% 
  select(
    LIST = `Medication List`,
    PRODUCTID = `NDC Code`
  )

hyp_ndc_df
```

```
## # A tibble: 3,230 x 2
##    LIST                          PRODUCTID  
##    <chr>                         <chr>      
##  1 ACE Inhibitor/ARB Medications 00093512501
##  2 ACE Inhibitor/ARB Medications 00093512505
##  3 ACE Inhibitor/ARB Medications 00185005301
##  4 ACE Inhibitor/ARB Medications 00185005305
##  5 ACE Inhibitor/ARB Medications 00247213730
##  6 ACE Inhibitor/ARB Medications 00378044301
##  7 ACE Inhibitor/ARB Medications 00781189201
##  8 ACE Inhibitor/ARB Medications 13811062810
##  9 ACE Inhibitor/ARB Medications 13811062850
## 10 ACE Inhibitor/ARB Medications 21695032630
## # ... with 3,220 more rows
```

One of the cool features of dbplyr is that we can copy this local dataframe to the database as a temporary table using the `copy_to()` function. 


```r
hyp_ndc_rmt <- copy_to(con, hyp_ndc_df, overwrite = T)

hyp_ndc_rmt
```

```
## # Source:   table<hyp_ndc_df> [?? x 2]
## # Database: duckdb_connection
##    LIST                          PRODUCTID  
##    <chr>                         <chr>      
##  1 ACE Inhibitor/ARB Medications 00093512501
##  2 ACE Inhibitor/ARB Medications 00093512505
##  3 ACE Inhibitor/ARB Medications 00185005301
##  4 ACE Inhibitor/ARB Medications 00185005305
##  5 ACE Inhibitor/ARB Medications 00247213730
##  6 ACE Inhibitor/ARB Medications 00378044301
##  7 ACE Inhibitor/ARB Medications 00781189201
##  8 ACE Inhibitor/ARB Medications 13811062810
##  9 ACE Inhibitor/ARB Medications 13811062850
## 10 ACE Inhibitor/ARB Medications 21695032630
## # ... with more rows
```

With the hypertension codes in the database you can use `inner_join()` to find the matching drugs from the "pde" table.


```r
pde_hyp_rmt <- 
  pde_rmt %>% 
  inner_join(
    hyp_ndc_rmt, by = c("PROD_SRVC_ID" = "PRODUCTID")
  )

pde_hyp_rmt
```

```
## # Source:   lazy query [?? x 9]
## # Database: duckdb_connection
##    DESYNPUF_ID      PDE_ID SRVC_DT    PROD_SRVC_ID QTY_DSPNSD_NUM DAYS_SUPLY_NUM
##    <chr>            <chr>  <date>     <chr>                 <dbl>          <dbl>
##  1 03ADA78C0FEF79F4 83554~ 2008-02-01 51079098420              30             30
##  2 040A12AB5EAA444C 83974~ 2009-11-27 63874037930              90             30
##  3 043AAAE41C9A37B7 83024~ 2008-08-11 00247228590             200             90
##  4 0D540BEBC45ADCA7 83454~ 2009-02-18 60429007210              60             90
##  5 0D540BEBC45ADCA7 83874~ 2009-06-15 66105010309              30             30
##  6 0E0A0107787B32AA 83714~ 2008-06-30 66105050406             100             90
##  7 109D67D114778917 83064~ 2008-11-21 13411014302              30             60
##  8 109D67D114778917 83034~ 2009-07-03 66685070203              30             30
##  9 10D75CDD5B4AD3B0 83804~ 2008-07-01 00247202160              90             90
## 10 1183CA4884F8A0A8 83934~ 2009-01-25 00185002501              90             20
## # ... with more rows, and 3 more variables: PTNT_PAY_AMT <dbl>,
## #   TOT_RX_CST_AMT <dbl>, LIST <chr>
```

We can now use `collect()` to retrieve the data as a local dataframe. As a dataframe, we can now use the `AdhereR` package to calculate the __medication possession ratio (MPR)__ for members who filled a hypertension medication. We can see the MPR for each member who filled one of the medications, and we can calculate the median and mean MPRs across these beneficiaries.


```r
# install.packages("AdhereR")
library(AdhereR)

pde_hyp_df <- pde_hyp_rmt %>% 
  collect()

hyp_adhere <- CMA4(
  data = pde_hyp_df,
  ID.colname = 'DESYNPUF_ID',
  event.date.colname = 'SRVC_DT',
  event.duration.colname = 'DAYS_SUPLY_NUM',
  date.format = "%Y-%m-%d"
)

hyp_adhere_cma <- hyp_adhere$CMA %>% tibble()

hyp_adhere_cma
```

```
## # A tibble: 134 x 2
##    DESYNPUF_ID         CMA
##    <chr>             <dbl>
##  1 03ADA78C0FEF79F4 0.0411
##  2 040A12AB5EAA444C 0.0411
##  3 043AAAE41C9A37B7 0.123 
##  4 0D540BEBC45ADCA7 0.164 
##  5 0E0A0107787B32AA 0.123 
##  6 109D67D114778917 0.123 
##  7 10D75CDD5B4AD3B0 0.123 
##  8 1183CA4884F8A0A8 0.0274
##  9 11C45CF0DDD03ACE 0.0822
## 10 1226AD9944384B64 0.0411
## # ... with 124 more rows
```

```r
hyp_adhere_cma %>% 
  summarize(
    median_mpr = median(CMA),
    mean_mpr = mean(CMA)
  )
```

```
## # A tibble: 1 x 2
##   median_mpr mean_mpr
##        <dbl>    <dbl>
## 1     0.0411   0.0848
```

## Data Limitations

While data included in claimsdb is useful for many types of analyses, it does include a few notable limitations.
- As noted earlier, the data is a small sample (500 beneficiaries) and is not intended to be representative of the Medicare population. In addition, the data is synthetic and not intended to be used for drawing inferences on the Medicare population.
- Since the data is more than 10 years old, it doesn't capture newer medications or procedures. It also includes procedure codes that have been retired or replaced.
- The diagnosis fields in the data use the International Classification of Diseases, Ninth Revision (ICD-9), but the United States converted to ICD-10 in 2015. If you are interesting in a mapping between ICD-9 and ICD-10, CMS has [resources](https://www.cms.gov/Medicare/Coding/ICD10/Archive-ICD-10-CM-ICD-10-PCS-GEMs) to consider.
- The Medicare population is mostly Americans aged 65 and over, so the data will not have claims on certain specialties such as pediatrics or maternity care.
