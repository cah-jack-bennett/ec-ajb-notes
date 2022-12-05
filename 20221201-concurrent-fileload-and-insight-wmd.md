# Notes on concurrent Fileload and Insight running for Walmart

**Key goal: Need to get insight and program launch parts done by
2023-01-01 (?!)**

## 2022-12-01 - Initial notes
* "Do whatever it takes on Stage PHI" - max priority
* Start planning that we are going to run two parallel/concurrent runs
  of fileload and insight
* Tracy: add resources to Stage-PHI
  * Add new EBS volumes ... 
  * Change instance type? More memory ... more IOPS (or not)
  * IOPS 16K ... 14.5K ... ?? read/write?
  * up to 48 CPUs?
  * Add a volume to Stage-PHI (EBS volume) ... separate databases goes
    on this new volume. (MemberRepository and OutcomesIdentification
    for Walmart-only)
* RUN CONCURRENTLY by EOD Wednesday December 7 -- >>> Provide list of
  required changes BY EOD WEDNESDAY DECEMBER 7 <<<
* Let's see how we can run these things at the same time without
  conflicting.
* Goals: 2+ concurrent runs of `PUP_fileload` and 2+ concurrent runs
  of `GR_Insight_Process`
* Key question: How can we run at the same time? How do they reference
  different data?
* TESTING - work on Sam's Club as an example ("same" as Walmart):
  Break out Sam's Club as a separate policy. Instance A operates on
  policy 1, 2, 3 ... Instance B operates on policy 4, 5, 6
* Draw an architecture diagram to understand what is going on here
  ... at what level of abstraction are we going to
  split/shard/partition this?

## 2022-12-02

* Need to pick up the pace for Walmart Stuff
* Add a volume to ING02/Stage-PHI
* Experiments with further runs of Fileload / Insight on Stage PHI
* After we add the volume to Stage-PHI hosts, then what? Visible by ING02

----
### My biggest questions about running Fileload (FL) and Insight Process (IP)  in parallel:

* How do we "point" one process at one set of
  customers/policies/records, and the other process at another set of
  customers/policies/records?
* Brainstorm ideas
  * Config file contains reference to volume, database instance, etc.
  * Question - ?? - could we select config set with command line
    argument - e.g. `python run_insight.py --shard A` for non-Walmart
    and `python run_insight.py --shard B` for Walmart (example)
  * How do we identify or select a source db host. e.g. ING02-A and
    ING02-B - config, etc?
  * Intention - make as FEW code changes and conditions as
    possible. e.g. it would be ideal if both instances/runs pointed to
    MemberRespository and
