# Dive_BNY
BNY specific Dive code including rules

## Architecture
![image1](https://github.com/user-attachments/assets/0ac4558b-5a49-4304-902c-9b6d64ebbbdc)

## Processes

## Data Layer
Separate stack which should include a HDB with typical trade/quotes data for example. This is the data that the DIVE engine will analyse and run rules on.

### HDB
HDB process that loads up the data
### Gateway
Used for routing queries received by DIVE engine (via IPC) to the HDB

## Analytics Layer
### Rules HDB
A HDB process which loads the rules themselves and all their parameters with a corresponding ID number, as well as the run data against each rule.

Tables: rundata, rules

### Model Cache
(WIP) - Required for workers to connect

### Gateway
The analytics gateway which is responsible for routing rules to the workers (Entry point). Rules are run from here via `.api.runrule`

### Workers
Q processes which take in calls from the DIVE analytics gateway and route them to the Data-Layer Gateway

