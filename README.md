# ACG-Deploy v12 for Planning Analytics SaaS

ACG-Deploy moves TM1 objects between Planning Analytics databases: dimensions, subsets, cubes, views, rules, processes and chores. It can also run TI processes on a target. Everything runs in TurboIntegrator through the TM1 v12 REST API (`ExecuteHttpRequest`). 

You list the objects to move in the **Deployment** cube, then run one process. That process copies each item to up to five targets and writes the result back to the cube. Items can run one at a time or in parallel, with large cubes split into slices that load at the same time.

## Requirements

- Planning Analytics SaaS (TM1 v12), or another v12 database that can reach the targets over HTTPS
- An API key for each tenant involved (the same key works for every database on one tenant)
- Admin rights on the database where the tool is installed
- PAW Git integration (Git push and pull) to install from this repository

## What's in this repository

| Path | Contents |
| --- | --- |
| `processes/` | All the tool's TI processes (`.ti` code + `.json` definition), including the `PLog` logging framework |
| `files/Metdata_PAW_v12.gz` | The Master Data Management PAW book |
| `tm1project.json` | Which objects PAW Git pushes and pulls, and the attribute export that runs before a push |

API keys and connection values are **never** in this repository. They live only in the `}API Control` cube of each database.

## Installing on a new database

A pull only brings in the objects. Nothing runs automatically afterwards, so setup is never attempted before this database's connection values are entered.

1. In PAW, connect the database to this repository and **pull** the `main` branch.
2. Run **`}Sys.Deploy.Setup`** with this database's connection values:

   | Parameter | Value |
   | --- | --- |
   | `pThisDB` | This database's exact name, e.g. `GitDemo-Prod` |
   | `pBaseURL` | The tenant URL, e.g. `https://us-east-1.planninganalytics.saas.ibm.com` |
   | `pTenantID` | The tenant ID |
   | `pApiKey` | An API key for this tenant. It is only ever logged masked |
   | `pSourceDB`, `pTarget1DB` | Optional: database names for the Source and Target1 environments |

   It creates the cubes and dimensions, and its log should end with **"Settings ok"**. A blank parameter keeps the value already stored, so check every field is filled in on the first run.
3. Run **`}Sys.Attr.Load.All`** to restore the attribute values carried in the repository.
4. Run **`}Sys.Deploy.Setup.Rules`**. It first checks that the values from step 2 actually reach this database, and stops with an ERROR naming the database if they don't. It then sets the Deployment and picklist rules and creates the **Deploy_Admin** and **Deploy_User** groups. The changes appear a few seconds after it finishes.
5. Add users to Deploy_Admin (manage environments and keys) and Deploy_User (run deployments).
6. Register each target database (see below).

After a later pull that updates the tool, run step 3 again if attribute values changed, and steps 2 and 4 if the setup processes changed. The parallel deployment update changes both: step 2 adds the `}Deployment Jobs` cube, its two dimensions, the `max_workers` setting and the **Wave** measure, and step 4 lets Parameter 1–10 be entered on Cube items.

Both setup processes are safe to re-run. They add what's missing and keep what's there.

## Registering targets

Run **`}Sys.Deploy.Environment.Register`** for each target:

| Parameter | Meaning |
| --- | --- |
| `pEnvironment` | `Target1` … `Target10` |
| `pDatabase` | The target database name |
| `pBaseURL`, `pTenantID`, `pApiKey` | Only for a target on a **different** tenant. Leave blank for the same tenant |
| `pTest` | 1 (default) tests the connection first and saves only if the target answers |

Only Deploy_Admin members can run it. Keys for other tenants are stored in `}API Control` and referenced by name from the **Deployment Environments** cube.

## Running a deployment

1. Open the **Deployment** cube (or the Master Data Management book) and pick a deployment (`Deployment 1` … `Deployment 24`).
2. On **All Items**, set Name, Owner (informational), Source and Active = Yes.
3. Add one row per object (`Item 1` … `Item 49`):

   | Type | Name | Notes |
   | --- | --- | --- |
   | Dimension | dimension | Options: Include Attribute Values, Copy Attribute Rules, Include Subsets, Include Security Cube |
   | Subset | `dimension\|subset1;subset2` or `dimension:hierarchy\|subset` | |
   | Cube | cube | Structure, rules and data; the target cube is cleared first. For parallel runs, Parameter 1–10 can hold `pSplitDim\|<dimension>` and `pSlices\|<n>` |
   | View | `cube\|view` | |
   | Rule | cube | Rules only |
   | Process | process | Copies the process |
   | Chore | chore | Its processes must already be on the target |
   | RunTI | process | Runs it on the target with Parameter 1–10 (`name\|value`) and Timeout |

4. Choose Target 1–5 for each item. Optionally set **Wave** (1–9) to change when the item runs in a parallel deployment.
5. Run **`Deployment - Deploy Items`** with `pDeployment`, e.g. `Deployment 1`, and `pParallel`: 0 (default) runs items one at a time in item order, 1 runs them in parallel (see below).

Each item's **Deployment Status** shows `Success` or the reason it failed, per target. All Items shows the overall result, start, end and duration.

## Parallel deployments

With `pParallel = 1`, each item going to each target becomes its own job (`}Sys.Deploy.Job`, started with `RunProcess`). Jobs run in waves, and a wave starts only when the previous one has finished:

| Wave | Item types |
| --- | --- |
| 1 | Dimension |
| 2 | Subset |
| 3 | Cube (structure and rules, target cleared) |
| 4 | Rule |
| 5 | View |
| 6 | Cube data, in slices |
| 7 | Process |
| 8 | Chore |
| 9 | RunTI, one at a time in item order |

- **How many at once:** up to `max_workers` jobs (System - Admin Parameters, default 4).
- **Cube data in slices:** each slice copies every Nth leaf of a split dimension. The defaults are the cube's largest dimension and `max_workers` slices. Override them with `pSplitDim|<dimension>` and `pSlices|<n>` (at most 32; 1 = no slicing) in the item's Parameters.
- **Failures:** if a job fails on a target, that target skips its later waves ("Skipped – earlier failure"). Other targets carry on.
- **Time limit:** `process_timeout` (System - Admin Parameters, default 14400 seconds) covers the whole run. Jobs still running at the limit are marked Timeout and left to finish, because a job can't be stopped.
- **Results:** jobs write their results to the `}Deployment Jobs` cube. The orchestrator collects them after the last wave, so Deployment Status and the log are the same as in a one-at-a-time run. Only the orchestrator writes to the Process Log.

Both modes leave the targets in the same state. In testing, a 100,000-cell cube took 64 seconds in parallel and 1 minute 58 seconds one at a time.

## Logging

Every process logs to the **Process Log** cube (and Process Log by User) through `PLOG`. The status cell (T08) shows **ERROR only for critical errors**, the ones that stop a run: missing rights, an inactive deployment, wrong or missing connection values. A single item that fails to copy doesn't stop the deployment. It's reported in that item's status and in the log, and T08 stays SUCCESS.

## Moving attribute values with Git

PAW Git moves objects, not data. To carry attribute values (such as `System - Admin Parameters`):

- **Before a push**, `}Sys.Attr.Export` writes the values into `}Sys.Attr.Load.<dimension>.NN` processes, which are then pushed like any other process. It runs as a PrePush task.
- **After a pull**, run `}Sys.Attr.Load.All`. It runs every loader and restores the values.

## Notes and limitations

- **Parallel jobs can't be stopped:** a job still running when `process_timeout` is reached finishes on its own.
- **Same-database calls:** REST calls a process makes into its own database are sent asynchronously; a synchronous call would wait on the process's own lock.
- **Large objects:** a TI process parameter holds about 64 KB, so request bodies and view axes are passed between processes in global variables (`gHttpReqBody`, `gAxisIn`). Dimensions, subsets, views, processes and rules of any size copy correctly.
- **PAW pushes:** a PAW Git push keeps `README.md` but removes any other non-TM1 file from the branch. Keep files like `LICENSE` on `main` and merge into it.
- **Pulls don't delete:** a pull never deletes objects from a database. Remove retired objects by hand.

## License

see LICENSE on this Branch
