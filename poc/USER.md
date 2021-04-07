## User Guide


### Create New Project

_aka Creating a new Project Resource Path under UMCCR program_

- Login to Gen3 portal > Go to `umccr` submission program at https://gen3.dev.umccr.org/umccr
- Click "Use Form Submission"
- Select "project"
- Must fill as follows, for example:
    - **code**: _vic_
    - **dbgap_accession_number**: _phs001110.v2.p1111_
    - **name**: _vic_
- Click "Upload submission json from form"
- You should see generate JSON (you may edit those JSON there if you wish)
- Once finalise, click **Submit**


### Perform Graph Query

- Login > Go to **Query** menu i.e https://gen3.dev.umccr.org/query
- Switch to Graph Model (If it is in Graph mode, you shall see "Switch to Flat Model")
- Clear left panel and enter as follows to list all projects:
    ```
    {
      project {
        project_id
        name
        code
        dbgap_accession_number
      }
    }
    ```

- List all programs:
    ```
    {
      program {
        name
        dbgap_accession_number
        project_id
      }
    }
    ```

### How Resource Paths Authorization Work

- Gen3 **_Resource Paths_** are access controlled by [Fence](https://github.com/uc-cdis/fence) and [Arborist](https://github.com/uc-cdis/arborist) AuthZ services. e.g. UMCCR program with `vic` project resource path as follows:
    ```
    /programs/umccr/projects/vic
    ```

- All further resource (file upload, metadata, samples, analysis reports, etc) will be submitted into a particular project, hence all AuthZ permission bind under a project path.

- User ACL on resource path is configured in `user.yaml` (Fence service) on how / who should have access at what permissions or role. Please see [`user.yaml` guide](https://github.com/uc-cdis/fence/blob/master/docs/user.yaml_guide.md) to get the idea.


### Perform Data Submission

_Resource node creation (graph model) on given default Data Dictionary (DD)_

> 🙋‍♂️ Please follow along the following video tutorial:

> Please create all resources as Jim does in Step 1, under the project you created from ☝️ **New Project** section.

**Step 1:** Jim Henegan tutorial: [Submitting Data to a Gen3 Commons](https://www.youtube.com/watch?v=F2EOtHPg6g8)

<details>
  <summary>Click to expand!</summary>

  [![Submitting Data to a Gen3 Commons](https://img.youtube.com/vi/F2EOtHPg6g8/0.jpg)](https://www.youtube.com/watch?v=F2EOtHPg6g8)
</details>

**Step 2:** Gen3 Data Commons - [Data Upload Tutorial](https://www.youtube.com/watch?v=QxQKXlbFt00)

> This video goes along with the following sections next. Specifically, please follow up to creating `core_metadata_collection` resource as described in the video. You can skip (or just watch through) the rest, so that to connect with the **custom Upload Flow** section explain in next.

<details>
  <summary>Click to expand!</summary>

  [![Gen3 Data Commons - Data Upload Tutorial](https://img.youtube.com/vi/QxQKXlbFt00/0.jpg)](https://www.youtube.com/watch?v=QxQKXlbFt00)
</details>


### Using Gen3 CLI Client

- [Download](https://github.com/uc-cdis/cdis-data-client/releases/tag/2021.04) and install
```
wget https://github.com/uc-cdis/cdis-data-client/releases/download/2021.04/dataclient_osx.zip
unzip dataclient_osx.zip
mv gen3-client /usr/local/bin
chmod +x /usr/local/bin/gen3-client
gen3-client --help
```

- Go to your [Profile](https://gen3.dev.umccr.org/identity) > Create API key > download `credentials.json`

- Configure
```
cd ~/Download/
gen3-client configure --profile=gen3 --cred=credentials.json --apiendpoint=https://gen3.dev.umccr.org/
```

- Check auth
```
gen3-client auth --profile=gen3
2020/08/19 14:27:20
You have access to the following project(s) at https://gen3.dev.umccr.org:
...
...
...
```


### Upload Flow

- We will need `Python >= 3.6` environment. Optionally, you may wish to use Conda environment. Please create one and activate it.
    ```
    conda create python=3.8 -n gen3
    conda activate gen3
    ```

- Install `g3po` as follows
    ```
    pip install g3po
    g3po version
    ``` 

- Prepare staging location for uploading task
    ```
    mkdir -p /tmp/gen3
    cd /tmp/gen3
    ```

- Copy over the downloaded `credentials.json` to staging location
    ```
    mv ~/Download/credentials.json /tmp/gen3
    ```

- Copy and prepare your data file to staging location, e.g. create text file with some content
    ```
    touch victor_test1.txt
    echo "LOREM IPSUM" > victor_test1.txt
    ```

- Upload file using `gen3-client` as follows:
    ```
    gen3-client upload --profile=gen3 --upload-path=vic_test1.txt
    2020/10/02 04:17:00 Finish parsing all file paths for "/tmp/gen3/submit-data/upload_flow/vic_test1.txt"
    
    The following file(s) has been found in path "/tmp/gen3/submit-data/upload_flow/vic_test1.txt" and will be uploaded:
        /gen3/submit-data/upload_flow/vic_test1.txt
    
    2020/10/02 04:17:00 Uploading data ...
    vic_test1.txt  35 B / 35 B [==============================================================================] 100.00% 0s
    2020/10/02 04:17:01 Successfully uploaded file "/tmp/gen3/submit-data/upload_flow/vic_test1.txt" to GUID f5f52160-d995-4c8b-8131-a149e5a12069.
    2020/10/02 04:17:01 Local succeeded log file updated
    
    
    Submission Results
    Finished with 0 retries | 1
    Finished with 1 retry   | 0
    Finished with 2 retries | 0
    Finished with 3 retries | 0
    Finished with 4 retries | 0
    Finished with 5 retries | 0
    Failed                  | 0
    TOTAL                   | 1
    ```

- Generate md5 checksum
    ```
    md5sum vic_test1.txt
    64503f07db17f16d48cfb9d8e0553d7b  vic_test1.txt
    ```

- Determine file size in bytes
    ```
    wc -c vic_test1.txt
          35 vic_test1.txt
    ```

- Query Gen3 _indexd_ service using GUID (from output of `gen3-client upload` step ☝️)
    ```
    g3po index get f5f52160-d995-4c8b-8131-a149e5a12069 | jq
    {
      "acl": [],
      "authz": [],
      "baseid": "6f6f27fa-f81c-40b8-bb0e-751b9c425f52",
      "created_date": "2020-10-01T18:17:00.797405",
      "did": "f5f52160-d995-4c8b-8131-a149e5a12069",
      "file_name": "vic_test1.txt",
      "form": null,
      "hashes": {},
      "metadata": {},
      "rev": "88a8688b",
      "size": null,
      "updated_date": "2020-10-01T18:17:00.797411",
      "uploader": "san.lin@umccr.org",
      "urls": [],
      "urls_metadata": {},
      "version": null
    }
    ```

- Please go to https://gen3.dev.umccr.org/submission/files i.e. _Login > Submit Data > Map My Files_

- There, you should see the uploaded file with status **"Generating..."**

- Use `g3po` to update hash, size and urls to the blank record using the GUID
    ```
    g3po index blank update \
      --guid f5f52160-d995-4c8b-8131-a149e5a12069 \
      --rev 88a8688b \
      --hash_type md5 \
      --hash_value 64503f07db17f16d48cfb9d8e0553d7b \
      --size 35 \
      --urls s3://umccr-gen3-dev/f5f52160-d995-4c8b-8131-a149e5a12069/vic_test1.txt \
      --authz /programs/umccr/projects/vic \
      | jq
    
    {
      "baseid": "6f6f27fa-f81c-40b8-bb0e-751b9c425f52",
      "did": "f5f52160-d995-4c8b-8131-a149e5a12069",
      "rev": "a546bf88"
    }
    ```

- Please refresh the https://gen3.dev.umccr.org/submission/files

- Now the uploaded file should change to **Ready status**. Select and continue with Map Files in WindMill Data Portal UI there.


## REF

- Please refer to [`g3po` README](https://github.com/umccr/g3po) for more ad-hoc CLI commands to work with Gen3 services, such as out-of-band data ingesting using manifest indexing and mapping to graph data dictionary model, and so on. Also, refer [its wiki entry](https://github.com/umccr/g3po/wiki) for technical details about blank index record update and why the need of this.

- Now please review and contrast all ☝️ steps with:
    - https://gen3.org/resources/operator/#6-programs-and-projects
    - https://gen3.org/resources/user/access-data/
    - https://gen3.org/resources/user/submit-data/
    - https://gen3.org/resources/user/gen3-client/
    - https://gen3.org/resources/user/submit-data/sower/
