# ACG-Deploy v12 for Planning Analytics SaaS

ACG-Deploy moves TM1 objects between Planning Analytics databases: dimensions, subsets, cubes, views, rules, processes and chores. It can also run TI processes on a target. Everything runs in TurboIntegrator through the TM1 v12 REST API (`ExecuteHttpRequest`). 

You list the objects to move in the **Deployment** cube, then run one process. That process copies each item to up to five targets and writes the result back to the cube.

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
| `tm1project.json` | Which objects PAW Git pushes and pulls, and the tasks that run after a pull |

API keys and connection values are **never** in this repository. They live only in the `}API Control` cube of each database.

## Installing on a new database

1. In PAW, connect the database to this repository and **pull** the `feature_ACG-Deploy_v12` branch.
   The post-pull tasks then run `}Sys.Deploy.Setup`, `}Sys.Deploy.Setup.Rules` and `}Sys.Attr.Load.All`.
   On a new database, Setup.Rules stops with *"connection values missing"*. This is expected.
2. Run **`}Sys.Deploy.Setup`** with this database's connection values:

   | Parameter | Value |
   | --- | --- |
   | `pThisDB` | This database's exact name, e.g. `GitDemo-Prod` |
   | `pBaseURL` | The tenant URL, e.g. `https://us-east-1.planninganalytics.saas.ibm.com` |
   | `pTenantID` | The tenant ID |
   | `pApiKey` | An API key for this tenant. It is only ever logged masked |
   | `pSourceDB`, `pTarget1DB` | Optional: database names for the Source and Target1 environments |

   The log should end with **"Settings ok"**. A blank parameter keeps the value already stored.
3. Run **`}Sys.Deploy.Setup.Rules`**. It first checks that the values above actually reach this database, and stops with an ERROR if they don't. It then sets the Deployment and picklist rules and creates the **Deploy_Admin** and **Deploy_User** groups. The changes appear a few seconds after it finishes.
4. Add users to Deploy_Admin (manage environments and keys) and Deploy_User (run deployments).
5. Register each target database (see below).

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
   | Subset | `dimension:subset` | |
   | Cube | cube | Structure and rules |
   | View | `cube:view` | |
   | Rule | cube | Rules only |
   | Process | process | Copies the process |
   | Chore | chore | |
   | RunTI | process | Runs it on the target with Parameter 1–10 and Timeout |

4. Choose Target 1–5 for each item.
5. Run **`Deployment - Deploy Items`** with `pDeployment`, e.g. `Deployment 1`.

Each item's **Deployment Status** shows `Success` or the reason it failed, per target. All Items shows the overall result, start, end and duration.

## Logging

Every process logs to the **Process Log** cube (and Process Log by User) through `PLOG`. The status cell (T08) shows **ERROR only for critical errors**, the ones that stop a run: missing rights, an inactive deployment, wrong or missing connection values. A single item that fails to copy doesn't stop the deployment. It's reported in that item's status and in the log, and T08 stays SUCCESS.

## Moving attribute values with Git

PAW Git moves objects, not data. To carry attribute values (such as `System - Admin Parameters`):

- **Before a push**, `}Sys.Attr.Export` writes the values into `}Sys.Attr.Load.<dimension>.NN` processes, which are then pushed like any other process. It runs as a PrePush task.
- **After a pull**, `}Sys.Attr.Load.All` runs every loader and restores the values.

## Notes and limitations

- **One item at a time:** items and targets run in sequence.
- **Same-database calls:** REST calls a process makes into its own database are sent asynchronously; a synchronous call would wait on the process's own lock.
- **PAW pushes:** a PAW Git push keeps `README.md` but removes any other non-TM1 file from the branch. Keep files like `LICENSE` on `main` and merge into it.
- **Pulls don't delete:** a pull never deletes objects from a database. Remove retired objects by hand.

## License

see LICENSE on this Branch
