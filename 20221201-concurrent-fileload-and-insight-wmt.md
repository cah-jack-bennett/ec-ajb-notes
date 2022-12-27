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

---
## 2022-12-02

* Need to pick up the pace for Walmart Stuff
* Add a volume to ING02/Stage-PHI
* Experiments with further runs of Fileload / Insight on Stage PHI
* After we add the volume to Stage-PHI hosts, then what? Visible by ING02

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
    `MemberRespository` and `OutcomesIdentification`

---
## 20221205 - Most Important Questions And Activities Right Now

* Do we specify the shard/partition on the command line?
  * Could we call this "Shartition" or is that totally juvenile?
* What is the most valuable use of my time right now?

### What do we really want to do here?
* Perform 2+ concurrent runs of `PUP_fileload` at the same time on the same host
* Perform 2+ concurrent runs of `GR_Insight_Process` at the same time on the same host
* Be able to specify which "set" of clients, policies, patients, etc is selected (These "sets" map to "partition" on an EBS volume.)
  * Command line flag or option? (e.g. as above)
  * Configuration file - starter script kicks off 2+ separate runs at the same time

### What do I want to work on today?
* Figure out how to separate out these runs so that they don't collide with each other
* Figure out how to TELL each run what it is talking to
* Question about `GR_Insight_Config` - what varies between these ones?
* WHat is separate and what is common?


| **"Separate" (break into A, B, ...)**     | **Common (shared between multiple partitions)**                         |
|-------------------------------------------|-------------------------------------------------------------------------|
| Database "target" (db host)               | Code bases (`fileRepository-fileload`, `fileRepository-insightprocess`) |
| Database name (`MemberRepository_A`, etc) | Job runner/launcher host (`ING01`)                                      | 
| Config files?                             |                                                                         |
| Command line options (`--partition A`)    |                                                                         |
| EBS volume (should be transparent)        | asdfsadfs                                                               |


* Where do we encode the distinctions between the runs?
* Partition terminology - partition A, B, etc. Different runs that operate concurrently address 

---
### Looking at singleton objects and data strucutres and mutual dependencies for parallel runs
* What lockfiles are used between the different runs (fileload, insight)?
* What tables get written (updated, truncated, deleted, etc) in the different processes?
* What tables get read in the different processes?


Terminology
* **"Source tables"** - process reads only.
* **"Destination tables"** - process writes and may also read.

### `GR_Insight_Process`

* Locks
  * Insight does not create or use lockfiles.
  * Insight writes a lock flag to table `xxz` to make sure ClientOps/Puppers does not push policies when Insight is running.

* Destination tables
  * Documented the destination tables in [Excel document](https://cardinalhealth-my.sharepoint.com/:x:/p/jack_bennett/Ef2bB2hDhs9DlTunnpwBeg0B5-t3rxmes2uc8PpDM38bwQ?email=vedu.hariths%40cardinalhealth.com&e=jQsuDF)

* Source tables
  * Documented the source tables in [Excel document](https://cardinalhealth-my.sharepoint.com/:x:/p/jack_bennett/Ef2bB2hDhs9DlTunnpwBeg0B5-t3rxmes2uc8PpDM38bwQ?email=vedu.hariths%40cardinalhealth.com&e=jQsuDF)

### `PUP_fileload`

* Locks
  * Fileload uses lockfiles to prevent multiple instances from running.
  * We will need to qualify these lockfiles with a partition identifier in order to permit instances to run simultaneously on different partitions.

* Destination tables
  * Documented the destination tables in [Excel document](https://cardinalhealth-my.sharepoint.com/:x:/p/jack_bennett/Ef2bB2hDhs9DlTunnpwBeg0B5-t3rxmes2uc8PpDM38bwQ?email=vedu.hariths%40cardinalhealth.com&e=jQsuDF)

* Source tables
  * Documented the source tables in [Excel document](https://cardinalhealth-my.sharepoint.com/:x:/p/jack_bennett/Ef2bB2hDhs9DlTunnpwBeg0B5-t3rxmes2uc8PpDM38bwQ?email=vedu.hariths%40cardinalhealth.com&e=jQsuDF)

### `PUP_fileload_publish`

* Locks
  * xyz

* Destination tables
  * xyz

* Source tables
  * xyz

---
## 20221206 - Actions on partitioned instance

### What next?
* Work with Dale and Tracy on the duplication / migration of the databases
* Do we want the new ones to be empty?
* What changes should happen on the file system?
* How do we address a partition set in a run?
  * How does this happen under the hood? SSA commands, Python library calls, etc?
  * **Every** reference to `MemberRepository` or `OutcomesIdentification` ... would be a lot of changes!
    * `PUP_fileload` references to `MemberRepository` are few and 'should' only be in config! What about `OutcomesIdentification`? Also looks OK.
    * `GR_Insight_Process` references to databases as string literals
  * What functions and capabilities really _need_ a separate DB instance?
    * IOPS
    * Truncate tables
    * Write to tables (?)
* The policy (patient, client, etc) sets must always be disjoint. Given this, what constraints can we relax?
  * All destination tables will be on different partitions
  * What about source tables? In principle they could read from one common partition, but ... ?

* What is the situation with `MemberRepository.masterdata.approvalRepositoryPatient`? It is the only `MemberRepository` table written by Insight?
  * Nothing in fileload writes to it
  * Some code in fileload reads this table.
  * Is it a subset of `repositorypatient` table that is to be pushed on a given day?
  * Why is it in `masterdata` schema? Seems to be Insight "writing back" into `MemberRepository` (atypical pattern)
  * `fileRepository-insightprocess` truncates this table and then inserts data into it (this runs in `run_insightPushPolicies.py` - step 3 of `GR_Insight_Config`)

* Cards to create (WOOF and GROWL, depending on repo and scope)
  * Refactor `fileRepository-fileload` to have all instances of database name literals gone, replaced by env variable name or command line argument
  * Refactor `fileRepository-insightprocess` to have all instances of database name literals gone, replaced by env variable name or command line argument
  * Scripts for testing on Stage-PHI
    * Rebuild database set "B"
    * Construct/copy all tables for experimental purposes - Where do these come from? Where 'should' they?
  * Temporary: use Sam's Club data - copy tables and data into database set B to play with it
  * Manual testing of Insight and Fileload on Stage-PHI (test branches from command line)
  * Upgrade EC2 hardware / instance type (is this even a GROWL/WOOF card?)

---
## 20221207 - T4/WMT activities

### What are the next actions?
* Create new cards
* Code review the code for needed changes (fileload, insight, etc)

### Tasks to convert into new cards (GROWL or WOOF)
* Test new databases MemberRepository_B and OutcomesIdentification_B (get more specific - test WHAT?)
  * Reconcile tables (schemas, contents, etc) between A set (original) and B set (Walmart-specific)
  * Reconcile stored procedures and other database objects between A set (original) and B set (Walmart-specific)
* Develop scripts to rebuild tables and other database objects on MemberRepository_B and OutcomesIdentification_B
* Test load sample file data (Walmart) into new "B" databases/tables (PUP_fileload)
* Test run Insight process on data loaded in new "B" databases/tables (GR_Insight_Process)
* Enable command line argument on fileload script(s) to select database set (A, B)
* Enable command line argument on insight script(s) to select database set (A, B)
* Evaluate the migrations for database literals ... (???)

### Questions and comments
* What about tables that were created/loaded long ago and their origin is lost or unknown? i.e. look up tables, etc. Created with one-off manual commands and not repeatable migrations?
* Flip side - what about no longer used tables? (left over, etc) ... can we purge them?
* How do we duplicate a database? Copy it table by table, object by object. How do we know we have everything there?
* The "B" tables will not get overwritten by prod yet... (none existing on prod)
* Will new and/or larger hosts be able to handle the load? (1.2M Rx records per day)
* Unknowns:
  * How do we build/copy all the tables programmatically? Then truncate them. I guess we see what is there in docker to assemble the test instances?
  * Would we use `restore.sql` and restore from the `.bak` database files?
  * Possible methodology - follow what we do in Docker. Restore from `.bak` files and then truncate (?) all the tables. Apply the migrations?
  * What if db migrations contain literal references to `MemberRepository` or `OutcomesIdentification`? (can these be templated the same way as we are going to do with the regular code?)
  * Migrations are different between `fileRepository-fileload` and `fileRepository-insightprocess` ... does that mean we have to work on `ec-data-resource` as well for the migrations?

---
## 2022-12-08 - T4/WMT activities

### Questions and notes
* How do we deal with migrations in the various repos? Migrations are a big issue and a major risk.
* Not just `fileRepository-fileload` and `fileRepository-insightprocess` ... there is also `ec-data-resource` and many others. A conversion of all instances of `MemberRepository` and `OutcomesIdentification` to variables `MEMBER_REPOSITORY_DB` and `OUTCOMES_IDENTIFICATION_DB` is needed in multiple repositories.
* Risk of transition on the same `ING02` host versus creating the new host `ING03` - clean slate and unchanged codebase is good.

---
## 2022-12-09 - T4/WMT activities

### Questions and notes
* I am not sure what the priority is right now.
  * I don't seem to have urgent tasks or coding work since we don't know exactly what we're doing. Proposing `ING03` and going to install Windows, SQL Server, etc. Totally separate server.
  * Given this, what is the most valuable use of my time right now?
  * I can dive into some devops activities and deployment.
  * Udemy courses for Ansible, Jenkins, Terraform, Kubernetes, etc.
  * "If I had six months to become a DevOps guru, what steps would I take?"

* Possible activities for `ING03` and WMT transition

* Adapt several cards in anticipation of the `ING03` dedicated host transition in place of the database split
* Evaluate what might be needed in terms of deployment (Jenkins, etc)
* Pivot code analysis to what would be needed to operate on a different host with the same database name(s) rather than the same host with different database names  

---
## 2022-12-12 - T4/WMT activities

### Questions and notes
* Fundamental question: what is the most valuable use of my time right now? (aka "please tell me what to do!")

* TODO
  * Complete cards today for `ING03` pivot of cards (former cards)
  * Cards for me and Dale
  * Leverage pre-existing knowledge of `ING02` cards ... recap and review
  * Summary: generate cards for `ING03`
  * No need to worry about CMJ for `ING03` because it doesn't run there, it runs on `SQL01`

### Actions and cards needed
* GR Log in to `ING03` and run commands on `git-bash`; check out repos on `ING03` and get dev environments up and running - as named user(s) (Jack 2022-12-12)
* GR Test out basic operation, access, etc of the migrated `MemberRepository` and `OutcomesIdentification` databases on `ING03`; manual test (command line) of `insight` and `fileload` runs on `ING03` (Jack 2022-12-12)
* GR Create and test Jenkins job for deploying `insight` to `ING03` (Dale 2022-12-09)
* GR Investigate code changes needed in `insight` for operation on `ING03` (Jack 2022-12-12)
* GR Clean out new databases on ING03 (migrated) from `ING02` - existing tables need to be truncated, etc (Dale 2022-12-09)
* GR Add required config table data for test Walmart onboarding - fileload config, client config, other stuff? (baseline config to ingest data for a client) (Jack 2022-12-12)
* GR Add, configure, update daily SQL Server Agent jobs to `ING01` to run on `ING03` (PUP_fileload, GR_Insight_Process, etc) (Jack 2022-12-12)
* GR Add observability and external logging for `ING03` - transmitting logs to ELK, Slack alerts, etc (Jack 2022-12-12)
* PUP Create and test Jenkins job for deploying `fileload` to `ING03` (NOT NEEDED)
* PUP Investigate code changes needed in `fileload` for operation on `ING03` (Jack 2022-12-12)

---
## 2022-12-13 - T4/WMT activities

### Questions and notes
* What comes next today on the `ING03/T4/WMT` project 2022-12-13?
* What new cards are needed?
* Gave Angie a sitrep of the current status of the `ING03`/WMT project
  * Meeting planned with Ricardo and Tracy
  * Questions about CMJ from AS
  * Reported out the cards created, transition from "Plan-B" project to "Plan-3" project
  * WITMVUOMTRN? Focus on WMT/`ING03` naturally ...
* Transition to "Plan-3" from "Plan-B" means:
  * We don't have to change a million database definitions in code (although we should do that anyway). Also: MIGRATIONS 😬
  * We don't have other test/prod jobs competing on these hosts - performance testing can be more reliable.
  * Lower technical risk at the cost of higher up-front financial investment
  * We DO need to add in the ability to refer to database host `AOCWSAPING03` (stage), `AOCWPAPING03` (prod) (cards created) 

### Themes and focus areas ⬇️
  * Infrastructure
    * AWS, EC2, deployment, backups, runbooks, practices, etc
  * Database
    * Reconstruction of tables, copy them in, etc
  * Configuration (Customer)
    * What is in files versus what is in database tables?
    * What do we need to write/create for WMT?
  * Testing
    * Local, Stage-PHI, prod-pre-production.
    * When does usable test data land?
    * Can we work with Sam's Club test data to get out ahead of things? (same input schema for client files)
  * *Minimum Effective Dose* - leverage, automation, change as little code as possible to get the necessary effect.

* Completed part of WOOF-3671 - updated `ec-data-resource` and `ec-config-package` which are dependencies of several Growlers repos. Added config that will select the correct `ING03` when operating on that server.
* Where do the Python scripts run? It looks like they run on `ING01` but hit databases on `ING02` ... but the Python scripts look at the localhost to determine where they are. This is confusing.

---
## 2022-12-14 - T4/WMT activities

### Questions and notes
* Waiting for Ricardo to complete the database transition on `ING03`
* Continuing work on `GROWL-3671` - reviewing the repositories for any unexpected dependencies 
* Continuing work on `GROWL-3672` - evaluating different approaches to run the `ING03` jobs
* Starting to take a look at `WOOF-4253` - the same review on `fileRepository-fileload` and other ClientOperations repos

---
## 2022-12-15 - T4/WMT activities

### Questions and notes
* Work on `WOOF-4253` - find where the db name is hardcoded in `fileRepository-fileload`
* I can't find any references to `ING02` in migrations on `fileRepository-fileload`. Am I insane?
* Goals for today include push on `WOOF-4253` and `GROWL-3672`
* Work on experiment with changing the ordering of `PUP_fileload_audit` (which launches `PUP_fileload`), placing it after completion of `GR_Insight_Identification`

---
## 2022-12-16 - T4/WMT activities

### Questions and notes
* Communicating these bullet reports to `#insight-brainstorming`
* Coordinate with Vedu about these comms
* What's the next action here?
* Submit Sailpoint / self-serve request 1091845 for access - group `gLGS-SQLDDLAdminsOCINGP` hits `ING01` (I think I only had `ING02` ... so I DO have access on `ING03`)
* Timing of jobs - any effects due to the changes in ordering put in place last night? None are visible.
* The fears about the new server are less about financial cost and more about impact/risk on CMJ and what happens if a new Step breaks it (how is CMJ operating on Staging?)
  * New step in CMJ would be due to retrieving data from `ING03` as well as `ING02`
* What can I do now in `ING03`? Translate SQL Server Agent jobs over?
* Migrating SQL Server Agent jobs from one host to another
  * (1) script the jobs to an editor window
  * (2) save to file
  * (3) copy the files to the new host
  * (4) create the jobs from the SQL files (just run the SQL directly to create the jobs)
* This sounds awful and I am wondering if there is an easier way to do this...
* At least we can concatenate all the files and run it all at once!
* Do I mirror the jobs from `prod` or `stage-PHI`?
* What have I confirmed today?
  * I am able to create and destroy SSA jobs on `ING03` (stage)
  * I can try to run SSA jobs on `ING03`
  * No underlying (Python) code is present yet on `ING03` to be able to run ... this must be deployed by Jenkins

---
## 2022-12-19 - T4/WMT activities

### Questions and notes
* Focus this week: run Walmart policy push AND a regular policy push through `ING02` (stage) 
  * Test out how this operates in a real performance scenario with realistic data (real Walmart test data?)
  * See if it is delayed, slowed down, etc.
  * What do the numbers look like?
  * Dependencies:
    * Walmart test data - real files (whom do I need to ask for this and when can we get it?)
    * Walmart config install and setup on `ING02` (stage) - client config, etc
* Procedure:
  * (1) push ALL regular policies (same list as on prod)
  * (2) add in Walmart policies and data - load files, push policies
  * (3) analyze and evaluate what happened after the fact - timing, etc

---
## 2022-12-20 - T4/WMT activities

### Questions and notes
* Hypothesis: "It will be okay to run Walmart data on an upgraded `ING02` because we have the idle time window between 0100-0700 (approx) where `PUP_fileload` is delaying"
* Questions about the hypothesis:
  * What happens when a "big day" (Walmart + CVS, for example) happens on day T and then it all hits Insight on T+1?
  * Approval process for the big day policies?
* What do I need to do in order to push Walmart policies TODAY?
  * Walmart data set - file(s) to load manually.
  * Config information for Walmart - fileconfigload, clientID, policy IDs, policy mapping(s), etc. What am I forgetting?
  * Configure Walmart as a client on StagePHI to be able to load files and push policies
  * Push new Walmart policies alongside the SAME set of policies pushed in prod to load StagePHI as hard as possible. (No reduced set of 11 policies or whatever.)

* What next?
  * Notes from Pravin about EC2 instances and upgrades
    * Upgrade from `r3` to `r6` instances (`ING02`).
    * Upgrade disk type from `gp3` to `io1` - doubles peak IOPS from 16k/s to 32k/s (`ING02`).
    * Upgrade CPU count from 32 -> 48 (`ING02`).
    * What is the marginal cost of upgrading the SQL Server license for this?

* What do I do until we get data from Vandelay and then Airbud?
  * Still need access for group `gLGS-SQLDDLAdminsOCINGR` (`ING01` admin capability to edit SSA jobs)
  * Write conops, etc.

---
## 2022-12-21 - T4/WMT activities

### Questions and notes
* Reorganize timing data and add Prod and Stage-PHI data into the pre-existing spreadsheet
* Continue performance testing during upgrade of `ING02` EBS volumes from `gp3` to `io1` disk types

---
## 2022-12-22 - T4/WMT activities

### Questions and notes
* Continue evaluating timing data - log in the appropriate spreadsheet.
* Continue running tests on Stage-PHI `ING01/02` to stay busy and give the impression of urgent action.
* Continue to wait for WMT data from Vandelay.
* Mirror policy push actions from 2022-12-21 for test on Stage-PHI
* Check job timings before Stage-PHI rollover

---
## 2022-12-23 - T4/WMT activities

### Question and commentary I sent to Dale and Jonathan. 
OK, here’s a dumb question (along with a lot of editorial commentary:
If the anticipated 1.2M patient records per day gets filtered down to 30K patient records per day after qualification of the patients and removing those who qualify for zero programs …
Do we need all this infrastructure investment and improvement for “only” 30K patients?
(I know this is one single data point and may not be typical … but if that’s the case, then I wonder what would be typical. Averages, distributions, spikes, etc. There are two variable numbers in here - N, the initial patient number, and q, the fraction remaining after qualification.)

Looking at what changes are proposed/happening:
The updating of the EC2 instance types (r3 :arrow_right: r6) is likely long overdue. r3 are 2014 state-of-the-art instances.
Same thing with the EBS volume types - we are bumping up against the IOPS limits so it makes perfect sense to migrate from the gp3:arrow_right: io1.
But … if it is at all typical for the qualification to shrink down a patient population of 1.2M by 40x (i.e. N=1.2M, q=0.025), then I strongly suspect we do not need to worry about bigger and more complex infrastructure upgrades right now (e.g. adding ING03).

What do you guys think?

### Questions and notes
* If the shrinkage factor is 0.025 on a "typical" population of 1.2M, do we need this new infrastructure right now to handle the WMT pharma program?
* How do I load the appropriate files on Stage-PHI? I have a naive approach.
  1. Find the names of the files that were loaded on 2022-12-20
  1. Stage all of those files by name on Stage-PHI
  1. Load all of those files by their newly assigned `fileID` on Stage-PHI
  1. Reconcile all of those files on Stage-PHI.
* As of 0824: running into trouble with logging in on Stage-PHI. Not able to connect via RDS. Right before this, I ran into issues when logged in. where SSMS could not connect to `ING01` or `ING02`.
* 0928: Unfucked the connectivity issues (the usual FuseAD unlock thing)
* Walmart: `clientID=279` and `policyID=1883`

### Action Plan for Walmart load (from Vedu)

**2022-12-22**
* Configure Stage-PHI `ING01` morning jobs in the same way as Prod `ING01`
* As soon as `IO1` EBS volume conversion is finished
  * First, start `GR_Insight_Config` with the first test run using policies (11-set)
  * Next, start the second run with same policies immediately afterward.

**2022-12-23**
* Starting at 0700
  * If WM files are available then add CVS (12/20 raw files) + all other files that were dropped on 12/20+Walmart
  * Otherwise use CVS (12/20 raw files) + all other files to first go through FL, then policy push that was done on 12/21, and run Insight Config….

---
## 2022-12-27 - T4/WMT activities

### Questions and notes
* Loading the appropriate files on Stage-PHI - basic approach
  1. Find the names of the files that were loaded in Prod on day T.
  1. In the morning on day T+1, place all these files in the `.../incoming/` directory on Stage-PHI
  1. Run `PUP_fileload` on Stage-PHI to stage and load the new files "received" on Stage-PHI
  1. Push the same set of policies that we pushed on day T on Prod.
  1. Start Insight with `GR_Insight_Config`
  1. Need to verify with Jonathan about Walmart files. I think he can stage/load them at any time before morning of day T+1. (Confirmed: Jonathan has staged Walmart files 2022-12-27 afternoon)
* The problem I saw before with staging the files was due to unconventional directory paths in my Stage-PHI environment. I fixed this and staging works normally now.
* Generate the `file`,`fileConfigID` csv file from the database (Prod)
* Copy all the relevant files to Stage-PHI.
* Stage the files using `bowwow.py stage-file-batch $FILENAME.csv` with the list of files paired with the `fileConfigID` values
* Get the files from Prod and copy to Stage-PHI. Strip off time stamps if I can (script it, just `cut` or whatever).
* Report out this afternoon with a plan
* Start early tomorrow morning to get files loaded and get `PUP_fileload` started before running Insight

---
# Appendix: Progress notes to team (#insight-brainstorming)

* **2022-12-02**
  * Met with Dale and Tracy about provisioning the new EBS volumes and new databases on top of those volumes.
  * Built a draft/sketch architectural diagram for the new sharded/partitioned approach to isolate Walmart Insight/Fileload to its own databases.
  * Made notes and discussed nomenclature and namespacing for the tools (scripts, code, config, etc) to push policies, run Insight, run Fileload on the separate databases.
* **2022-12-05**
  * Investigated the tables that `PUP_fileload` and `GR_Insight_Process` modify (truncate, delete, insert, update) in their operation
  * Listed these tables in Excel document 20221205-fileload-insight-modify-data.xlsx
  * Brief discussion with Demetrius about Insight in the context of Walmart changes
  * Need to check a few more things but I think I have most of the tables that are updated. I also need to add in tables that are read/accessed but not changed.
  * Related to this: do you think there’s any advantage from using a reduced set of tables (compared to the existing tables in `MemberRepository` or `OutcomesIdentification`) or does that add complexity with no benefit. It looks like `PUP_fileload` writes to both `MemberRepository` as well as `OutcomesIdentification` while `GR_Insight_Process` writes to mostly `OutcomesIdentification` tables but also a couple of `Outcomes` schema tables and one `MemberRepository` table (I want to check that last one - it seemed strange to me, but it is in there in the Insight code).
* **2022-12-06**
  * Work with Dale and Tracy to set up new "database set" (pair of DBs `MemberRepository_B` and `OutcomesIdentification_B`) on new EBS volume on Stage-PHI `ING02` host. Start experimenting with new database (may be blocked by Admin access; requested this access from CoreDM through SailPoint)
  * Write up several new cards (both GROWL and WOOF - GROWL-3645, GROWL-3646, GROWL-3647, WOOF-4222 ) for tasks to be done to make the new fileload/insight runs possible. Brainstorm more cards (noted but not created yet).
  * Additional investigation of why and how Insight writes data back into `MemberRepository.masterdata.approvalRepositoryPatient`
  * Code review in `fileRepository-fileload` and `fileRepository-insightprocess` of where we would need to make changes and factor out any occurrences of literal database name strings
* **2022-12-07**
  * Continue reviews of fileload and insight code for multi-database capability
  * Start trying out interactive commands in "B" set databases (MemberRepository and InsightProcess)
  * Notes for several new WOOF/GROWL card titles and descriptions (created the cards now; still need to populate the descriptions with cleaned up notes)
* **2022-12-08**
  * Finalize text of GROWL/WOOF cards for separate Walmart OutcomesIdentification and MemberRepository databases; present and review cards in Client Ops and EC Walmart Readiness meeting. 
  * Investigate CMJ, job location, code location, and other relevant parameters.
  * Continue code reviews + experiments with `fileRepository-fileload` and `fileRepository-insightprocess` to add externally addressable databases.
  * Rising concern about risks of migrations breaking for multiple repositories (fileload, insight, as well as `ec-data-resource` and several others).
*  **2022-12-09**
  * Adapt several cards in anticipation of the `ING03` dedicated host transition in place of the database split
  * Evaluate what might be needed in terms of deployment (Jenkins, etc)
  * Pivot code analysis to what would be needed to operate on a different host with the same database name(s) rather than the same host with different database names
* **2022-12-12**
  * Add several new cards and descriptions for `ING03` actions (`GROWL-3667`, `GROWL-3668`, `GROWL-3669`, `GROWL-3670`, `GROWL-3671`, `WOOF-4253`)
  * Revise and add to the descriptions of cards Dale and I came up with on Friday (`GROWL-3661`, `GROWL-3662`, `GROWL-3663`, `GROWL-3664`)
* **2022-12-13**
  * Completed some work and drafted PRs on of WOOF-3671 - updated `ec-data-resource` and `ec-config-package` which are dependencies of several Growlers repos. Added config that will select the correct `ING03` when operating on that server.
  * Created card `GROWL-3672` to address the issue of where the Python scripts run from. (Existing scripts launch from SQL Server Agent on `ING01` and hit databases on `ING02` ... this raises some questions about the Python environments.)
  * Met w/ Ricardo and Tracy to clarify our intentions for the database objects in the new `ING03.MemberRepository` and `ING03.MemberRepository`
* **2022-12-14**
  * Waiting for Ricardo to complete the database transition on `ING03`
  * Continuing work on `GROWL-3671` - reviewing the repositories for any unexpected dependencies 
  * Continuing work on `GROWL-3672` - evaluating different approaches to run the `ING03` jobs
  * Starting to take a look at `WOOF-4253` - the same review on `fileRepository-fileload` and other ClientOperations repos
* **2022-12-15**
  * `WOOF-4253` - find where the db name is hardcoded in `fileRepository-fileload` and adapt this to multiple server names
  * `GROWL-3672` - decision to use new server `ING03` for SQL Server Agent operation, Python environments, and Python processes in addition to SQL operations
  * Experiment with changing the ordering of `PUP_fileload_audit` task (which launches `PUP_fileload` itself), placing it after completion of `GR_Insight_Identification`. We expect to have some data on this tomorrow since it will run overnight.
* **2022-12-16 updates**
  * Analysis of first set of results from recent reordering of SQL Server Agent jobs
  * Migrate several SQL Server Agent jobs to `ING03` host and testing (`GROWL-3669`)
  * Work on Jenkins jobs for deployment of `fileRepository-fileload` and `fileRepository-insightprocess` to `ING03` (`GROWL-3663`)
* **2022-12-19 updates**
  * Develop workflow and method to run Walmart policy push AND a regular policy push through `ING02` (stage)
  * Continue to run and analyze reordered production process with `PUP` and `GR` jobs
  * Meet and discuss process for interleaving Walmart file ingest with idle time on `PUP_fileload` (notes in `GROWL-3676`)
* **2022-12-20 updates**
  * `GROWL-3680` for updating test environment (SQL Server Agent jobs) on `ING01` Stage-PHI
  * Ongoing baseline testing and timing data collection of `PUP_*` and `GR_*` jobs in Prod and Stage-PHI (`GROWL-3676` and others)
  * Ongoing efforts to get Walmart test data into the hands of Client Operations and Eligibility Central for realistic performance testing
*  **2022-12-22 updates**
  * Continue running test runs of `PUP_*` and `GR_*` jobs on Stage-PHI while monitorong progress of disk (EBS volume) upgrades
  * Puppers received data from Vandelay but have some concerns about date representations in one of the fields that is not parseable as a date
  * Harmonize workflow order on Stage-PHI to mirror the same order as we have on Prod
  * Continue tracking job timings on Prod for comparison between "normal push" and "normal push + Walmart data"