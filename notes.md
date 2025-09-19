# Notes

## Prerequisites

- AAP operator 2.4 and 2.x available in the catalog (available for install)
- Persistent Storage
	- 1 PV for each AAP deployment backup
	- 1 PV for each Replica (1)
- Database
	- 1 DB for each new gateway component
	- 1 DB for each replica (1)
    - Confirm if all existing DBs have been upgraded to v15
- Enough capacity for running original & replicas (1)

(1) Side-by-side only

## Prep tasks

- Prerequisites

- Define the cut-over strategy (side by side only)

- Define procedure for replicating namespace/deployments (side by side only)

- Strategy testing & validation in LAB
    - [OCP Team] Configure the first operator to monitor existing namespaces with AAP deployments **ONLY**.
    - [OCP Team] Create replicated namespaces (rs1, rs2, etc.)
    - [OCP Team] Deploy second 2.4 operator monitoring rs1 **ONLY**

    - Backup an existing deployment
    - Restore the deployment in a replicated namespace (e.g. rs1)
    
    - Upgrade deployment in rs1 to 2.5/2.6 (Standard Approach)
    - Validate

- Document Team dependencies & responsibilities

- Document the step-by-step upgrade process

- Automate upgrade where possible
  - Backup/Rollback
  - CRDs creation and deployment
  - Disable/enable scheduled jobs (1)
  - etc.

- Review existing CRDs, Customization (nginx, redis, etc.) & Potential Impact

- Customers
  - REST endpoints and payload
  - Changes needed
  - Communication

- Sizing of new components


## Rollback

### In Place
- [operator backup/restore]: Downgrade operator and restore from backup.
- [DB snapshot]: ?

## Side by Side
Nothing beyond cleaning up replicated deployment and resources.
