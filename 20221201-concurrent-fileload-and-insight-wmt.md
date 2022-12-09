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
# Appendix: Progress notes to VH

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