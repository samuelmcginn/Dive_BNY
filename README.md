# Dive_BNY
BNY specific Dive code including rules to be loaded by Analytics Layer HDB.  
Links:  
https://github.com/DataIntellectTech/dive?tab=readme-ov-file  
https://data-intellect.atlassian.net/wiki/spaces/DAE/pages/186679306/DIVE+Data+Intellect+Validation+Engine

## Architecture
![image1](https://github.com/user-attachments/assets/3cecdbae-28bf-4c02-91b1-91e91be49492)

## Message Flow
1. Rule ID sent to worker(s) via `.api.runrule`  
   `.api.runrule["04042b45-2e6e-bea2-7a03-f2c663900f15"]`  
2. Worker prepares query (also uses `.api.runrule`, which calls `.api.runmodel`) and calls to Data Layer GW process
3. `getdata` is invoked on Data GW and calls to Data HDB for data (invokes function also called `getdata` on HDB process)
4. Specified data is queried via `getdata` and returned to Data GW
5. Data is returned to the worker process and `.api.runmodel` finishes running, applying the rule to the data from the Data_Layer HDB.
6. Output from the rule is sent from the worker process to the Analytics Layer HDB where it is saved to the `rundata` table and saved to disk on DIVE Analytics side

## Process Details

## Data Layer
Separate stack which should include a HDB with typical trade/quotes data for example. This is the data that the DIVE engine will analyse and run rules on.

### HDB
HDB process that loads up the data  
Tables:  
`multifeed`  
`trade`
### Gateway
Used for routing queries received by DIVE engine (via IPC) to the HDB

## Analytics Layer
### Rules HDB
A HDB process which loads the rules themselves and all their parameters with a corresponding ID number, as well as the run data against each rule.  
Tables:  
`rundata`  
`rules`

### Model Cache
(WIP) - Required for workers to connect

### Gateway
The analytics gateway which is responsible for routing rules to the workers (Entry point). Rules are run from here via `.api.runrule`

### Workers
Q processes which take in calls from the DIVE analytics gateway and route them to the Data-Layer Gateway  
*note - rules need to be loaded in manually to the worker process, this can be done by running:  
``rules:update source:`primary from .connect.hdb"select from rules"``

## Process Start Lines
Order below should be followed for starting Data Layer AND DIVE Stack

### Data layer HDB
`smcginn@homer~/dive/kdbdq/data_layer$ q hdb.q /home/smcginn/dive/kdbdq/nysetaq_generator/hdb`

### Data layer GW
`~/dive/kdbdq/data_layer$ q gw.q -p 31001`

### Rules HDB
`smcginn@homer~/dive/kdbdq/analytics$ q loader.q -process hdb -files hdb.q -hdb /home/smcginn/dive/kdbdq/analytics/hdb`
 
### Analytics Gateway
`smcginn@homer~/dive/kdbdq/analytics$ q loader.q -process gw -files gw.q`
 
### Modelcache
`smcginn@homer~/dive/kdbdq/analytics$ q loader.q -process modelcache -files modelcache.q`
 
### Analytics Worker
`smcginn@homer~/dive/kdbdq/analytics$ q loader.q -process worker -files worker.q -offset 2`

## Rules

### Rules Example

"Test Null Success" - Checks the total percentage of null values of TradePrice column for trade table for a given date("2024.01.04")

Rules are stored as dictionaries withi the following values:  
```
q)first select from rules where ruleid = "G"$"04042b45-2e6e-bea2-7a03-f2c663900f15"
rulename     | "Test Null Success"
userid       | `sym$`a12345
source       | `sym$`
tablename    | `sym$`trade
columns      | "TradePrice"
model        | `sym$`nullcount
grouping     | ""
threshold    | "0.01"
scheduletime | 05:00:00.000
runfreq      | "daily"
compare      | `sym$`<
comparetarget| ""
targetdate   | "2024.01.04"
filter       | `sym$`
custommodel  | ""
aggfunc      | ""
ruleid       | 04042b45-2e6e-bea2-7a03-f2c663900f15
created      | 2024.02.12D22:09:09.673114000
modified     | 2024.02.12D22:09:09.673122000
```
Once a rule is run, it's output can be viewed in the `rundata` table on DIVE HDB:
```
q)first select from rundata where ruleid="G"$"04042b45-2e6e-bea2-7a03-f2c663900f15"
date        | 2024.07.15
ruleid      | 04042b45-2e6e-bea2-7a03-f2c663900f15
pass        | 1b
output      | "0.007017172"
runtime     | 2024.07.15D09:57:12.801007898
datatime    | 0D00:00:00.306594719
completetime| 0D00:00:00.000071842
```
We can verify the output value of this rule manually by looking at the HDB data for the specified date/table/field as shown below:
```
q)count select TradePrice from trade where date = 2024.01.04,null TradePrice
188424
q)count select TradePrice from trade where date = 2024.01.04
26851844
q)188424%26851844
0.007017172
```
