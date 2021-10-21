# Manifest-based Data Submission

> Q. How do I submit pre-existing backlog of big data, like 1 PetaByte?
> 
> A. You will need file object manifest, minimally. Continue reading for how.

> Q. Do we _physically_ need to move pre-existing data buckets into Gen3 instance?
> 
> A. No. Your existing data can be kept as-is where they are. This includes some data volume in HPC. You just index them using manifest. Continue reading for how.

### Context

- In Gen3, there are couple ways to make data submission.
- Typical end-user data submission for smaller dataset (couple of files) from your desktop/workstation follow [normal data submission](../user-guide/submit-data.md) procedure. 
- However, this is not always the case such that there exists backlog of data stores in Cloud Storage buckets or data volumes on HPC cluster.
- You can drive all these data that sit in elsewhere into Gen3 Indexing and catalogue metadata _Graph_ model.
- This kind of data submission in Gen3 is known as couple names:
  - _manifest-based_ data submission
  - _out-of-band_ data ingestion
  - _DIIRM_ indexing (DIIRM - Data Ingestion, Integration, and Release Management)

### Notes

Each sub-directory contains demo example on this _manifest-based_ data submission with some mock data or public dataset for exploration; including **Consent & Data Access** (ACL), **interoperability** (DRS & Htsget), external bucket, so on.

### REF

For more details on technical:

- https://github.com/uc-cdis/indexd#use-cases-for-indexing-data
- https://github.com/uc-cdis/cloud-automation/tree/master/doc/data_upload
- https://github.com/uc-cdis/gen3sdk-python/blob/master/docs/howto/diirmIndexing.md

Bucket manifest:

> Note that, this essentially entails how you generate manifest file for existing Big data. This process can be outside Gen3 with some _Batch Job_ running in Cloud (AWS/GCP) or HPC. 

Or, you could also utilise Gen3 EKS Kubernetes cluster. If so, technical pointers are as follows.

- https://gen3.org/resources/user/submit-data/sower/
- https://github.com/uc-cdis/cloud-automation/blob/master/doc/bucket-manifest.md
- https://github.com/uc-cdis/cloud-automation/blob/master/doc/gcp-bucket-manifest.md
- https://github.com/jacquayj/gen3-s3indexer-extramural
