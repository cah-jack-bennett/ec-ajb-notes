# 2022-12-20 Walmart data discussion

* How do we get WMT data from Vandelay into Airbud and pass that downstream to `mrfileload` and beyond? 

Testing Goal: on stage-PHI, run a test of ingesting and pushing data (regular data PLUS Walmart) to possibly break `ING02` and observe if/how this breaks (e.g. Insight takes too long, Fileload takes too long) 

* VH and PR discussed on Friday
  * This is a data gathering exercise, not a definitive proposal for operational activity
  * Not (necessarily) proposing that we would use `ING02` for actual Walmart data processing
  * AWS EC2 and EBS instances to get upgrades _regardless of other changes_ proposed
  * Can we run things earlier in the day? (e.g. start Insight at 1300? Potential approval issue...)
  * Proposed timing of jobs:
    * `PUP_fileload` - regular one but started early (e.g. 1300 CT)
    * `PUP_fileload_2` - new WMT-focused one (interleaved). Time budget from 2100-0700 (approx)
    * `PUP_fileload_3` - existing 0700 one
  * Continue to run Insight _one time per day_ in this arrangement
    * Allow more time budget for more data processing (e.g. start at 1300CT rather than later) 

* Key questions
  * How long does this process run? Is this feasible with existing hardware?
  * Can we deal with expected increases in runtime based on the anticipated Walmart data?
  * If fileload runs at 0730, does Clinical/QRC have any time to review data prior to 1330CT? (specific human steps needed)

* AWS resource EC2/EBS updates (from Pravin)
  * Go from `gp3` type to `io1` to double IOPS peak from 16k/s to 32k/s
  * Go from `r3` instance type to `r6` instance type
  * Go from 32 CPUs to 48 CPUs on the instance

* Airbud questions and issues about Walmart data
  * Where to receive it? S3 or SFTP - let's ask Vandelay to put it in SFTP for simplicity
  * Data format: promised to be in the same format as Sam's Club.
  * Use the same Sam's Club DAG to process the received WMT data (for the current test, definitely)
  * Need the actual data to actually do any actual testing (Sam's Club DAGs, fileload, etc).
  * Parser API: can it handle the data volume? (recursion bug - probably not an issue -JG, DS) 
  * "Treat it as Sam's 2.0 and move forward"
  * Pydantic model for future replacement of Parser API (for full launch, better performance and error reporting 2023-02-01)