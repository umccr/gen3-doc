# Program & Project

Appropriate ACL permission required for creating a program and/or project. This task is usually carried out by ADMIN.

### Create New Program

_Creating new data submission program for your centre_

> Depends on your account permission, you may create new program.

- Login to Gen3 portal
- Go to `root` node at https://gen3.cloud.dev.umccr.org/_root
- Click "Use Form Submission"
- At dropdown, enter and select "program"
- Must fill as follows, for example:
    - **dbgap_accession_number**: _umccr123_
    - **name**: _umccr_
- Click "Generate submission JSON from form"
- You should see generated JSON (you may still edit JSON there)
- Once finalise, click **Submit**

> NOTE: Due to Portal UI bug, you won't see any screen update.

- Verify that program `umccr` created by visiting to program https://gen3.cloud.dev.umccr.org/umccr

### Create New Project

_Creating new Project under UMCCR program_

> Depends on your account permission, you may create new project under specify program.

- Login to Gen3 portal 
- Go to `umccr` submission program at https://gen3.cloud.dev.umccr.org/umccr
- Click "Use Form Submission"
- At dropdown, enter and select "project"
- Must fill as follows, for example:
    - **code**: _cup_
    - **dbgap_accession_number**: _umccr123.cup_
    - **name**: _cup_
- Click "Generate submission JSON from form"
- You should see generated JSON (you may still edit JSON there)
- Once finalise, click **Submit**

> You should see "Succeeded: 200" green bar.

- Verify that project `cup` is created under `umccr` program by visiting to 
  - Submission page at https://gen3.cloud.dev.umccr.org/submission
  - At "List of Projects" section 
  - Click "Submit Data" button for "umccr-cup"


## REF

- https://gen3.org/resources/operator/#6-programs-and-projects
