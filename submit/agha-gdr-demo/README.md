# AGHA GDR Demo Submission with Consent AuthZ

> https://gen3.cloud.dev.umccr.org/agha-gdr

![agha_gdr_demo_files_explorer.png](img/agha_gdr_demo_files_explorer.png)

This demo contains scenario:
- there exists pre-existing bucket with data
- create indexd records of these data files using bucket manifest
- submit (DD graph) metadata for these data files
- following [granular access to data files](https://gen3.org/resources/operator/#7-how-to-upload-and-control-file-access-via-authz) -- that in-turn, in alignment with [data file metadata `consent_codes` properties](img/agha_gdr_data_file_consent_codes.png)

## 0. Initiate AGHA Program & GDR Submission Project

- Follow User Guide's [Program & Project](../../user-guide/program-project.md) 

## 1. Mock Pre-existing Data

```
dd bs=1024 count=256 </dev/urandom > demo_mocked_somatic.vcf.gz

wc -c demo_mocked_somatic.vcf.gz
  262144 demo_mocked_somatic.vcf.gz

md5sum demo_mocked_somatic.vcf.gz
fbbd092bf77294b638befe84ab33f6dc  demo_mocked_somatic.vcf.gz

uuid v4
65df1736-82cb-498c-8c57-7321901a0de9

aws s3 cp demo_mocked_somatic.vcf.gz s3://umccr-test-data-bucket/SBJ00001/demo_mocked_somatic.vcf.gz
```

```
dd bs=1024 count=1024 </dev/urandom > demo_mocked_tumor.bam

wc -c demo_mocked_tumor.bam
 1048576 demo_mocked_tumor.bam

md5sum demo_mocked_tumor.bam
12d0530416d935b8f0a26e210d18c39c  demo_mocked_tumor.bam

uuid v4
30fc4617-d666-47da-9f22-99bf8f1ccf30

aws s3 cp demo_mocked_tumor.bam s3://umccr-test-data-bucket/SBJ00001/demo_mocked_tumor.bam
```

```
dd bs=1024 count=512 </dev/urandom > demo_mocked_germline.bam

wc -c demo_mocked_germline.bam
  524288 demo_mocked_germline.bam

md5sum demo_mocked_germline.bam
ed6170fd2d45c6779f1555922946d2b3  demo_mocked_germline.bam

uuid v4
618c3d68-038c-4dc7-a306-e3f053db7dde

aws s3 cp demo_mocked_germline.bam s3://umccr-test-data-bucket/SBJ00001/demo_mocked_germline.bam
```

```
aws s3 ls s3://umccr-test-data-bucket --recursive > list.stdout
```

## 2. Prepare Bucket Manifest
```
wget https://raw.githubusercontent.com/umccr/g3po/dev/sample/manifest.tsv

edit manifest.tsv
```

## 3. Perform Indexing using Manifest

```
export GEN3_URL=https://gen3.cloud.dev.umccr.org/
g3po index health
g3po index manifest
```

## 4. Graph Metadata

### 4.1 Download Metadata Template

- https://gen3.cloud.dev.umccr.org/DD
- Download template > JSON

### 4.2 Fill Metadata

- Complete metadata information in template JSON files

### 4.3 Submit Metadata

- https://gen3.cloud.dev.umccr.org/agha-gdr
- Upload File > template JSON files

## 5. Admin Follow up

- You may need to rerun Tube ETL to sync ElasticSearch indexes
- You may need to update `user.yaml` to take effect on new indexes' authz, if any
- You may need to update [Fence config](https://github.com/umccr/gen3-doc/blob/main/workshop/fence-config.yaml#L558) to include the new bucket; provide S3 compliant endpoint if it is not native bucket
