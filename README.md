[![Ingest Archiver Build Status](https://travis-ci.org/HumanCellAtlas/ingest-archiver.svg?branch=master)](https://travis-ci.org/HumanCellAtlas/ingest-archiver)
[![Maintainability](https://api.codeclimate.com/v1/badges/8ce423001595db4e6de7/maintainability)](https://codeclimate.com/github/HumanCellAtlas/ingest-archiver/maintainability)
[![codecov](https://codecov.io/gh/HumanCellAtlas/ingest-archiver/branch/master/graph/badge.svg)](https://codecov.io/gh/HumanCellAtlas/ingest-archiver)

# Ingest Archiver
The archiver service is an ingest component that:
- Submits metadata to the appropriate external accessioning authorities. These are currently only EBI authorities (e.g. Biosamples).
- Converts metadata into the format accepted by each external authority

In the future it will:
- Update HCA metadata with accessions provided by external authorities

At the moment it consists of a minimum of 2 steps.
1. A metadata archiver (MA) script (the one in this repository) which archives the metadata of a submission through the USI. This script also checks the submission of the files by the file uploader (see below).
1. A file uploader (FIU) of the archive data to the USI which runs on the EBI cluster. This will need access to the file submission JSON instructions generated by the metadata archiver.

This component is currently invoked manually after an HCA submission.

# How to run
1. Install the script requirements
`sudo pip3 install -r requirements.txt`

2. Get the project UUID of the project you want to archive. For example rsatija dataset `5f256182-5dfc-4070-8404-f6fa71d37c73`

3. Set environment variables
```
# Required variables
# If you don’t know the EBI Authentication and Authorization Profile (AAP) password we’re using for 
# archiving please ask an ingest dev or another EBI wrangler. This password will be different
# depending on whether you're upload to test archives through USI or production ones.
$ export AAP_API_PASSWORD=password
$ export INGEST_API_URL=http://api.ingest.data.humancellatlas.org/

# https://submission-dev-ebi.ac.uk - USI test archiving. This only gets wiped every 24 hours so if you
# want to keep testing you may have to do so with a different dataset.
# https://submission.ebi.ac.uk - USI production archiving.
$ export USI_API_URL=https://submission-dev.ebi.ac.uk
# $ export USI_API_URL=https://submission.ebi.ac.uk

# Optional variables
# If archiving to test you can keep this as subs.test-team-21. Otherwise you will need to
# set this to subs.team-2 as below.
$ export AAP_API_DOMAIN=subs.team-21
# $ export AAP_API_DOMAIN=subs.team-2

# If archiving to test you can keep this as https://explore.api.aai.ebi.ac.uk/auth
# Otherwise you will need to set it to https://api.aai.ebi.ac.uk/auth
$ export AAP_API_URL=https://explore.api.aai.ebi.ac.uk/auth
# $ export AAP_API_URL=https://api.aai.ebi.ac.uk/auth

```

4. Run the metadata archiver
`./cli.py --project_uuid=2a0faf83-e342-4b1c-bb9b-cf1d1147f3bb`

Add `--submit` flag to let the script running while the files are being uploaded. Please note that file uploading might take long.

`./cli.py --project_uuid="2a0faf83-e342-4b1c-bb9b-cf1d1147f3bb" --submit`

Note, if you are doing this in debug on anything other than the first time then you will need to add a custom 'alias' prefix. An alias is a technical artifact that USI uses to identify an entity for local linking purposes. However, even in test USI these are permanent, so subsequent attempts to test the same data set, even if they come from different projects, will need a unique alias.

This alias prefix can be anything though you may want to stick with something along the lines of HCA_2019-01-07_
`./cli.py --alias_prefix=HCA_2019-01-07-13-53_ --project_uuid=2a0faf83-e342-4b1c-bb9b-cf1d1147f3bb`


You should get output like:
```
Processing 6 bundles:
0d172fd7-f5af-4307-805b-3a421cdabd76
9526f387-bb5a-4a1b-9fd1-8ff977c62ffd
4d07290e-8bcc-4060-9b67-505133798ab0
b6d096f4-239a-476d-9685-2a03c86dc06b
985a9cb6-3665-4c04-9b93-8f41e56a2c71
19f1a1f8-d563-43a8-9eb3-e93de1563555

* PROCESSING BUNDLE 1/6: 0d172fd7-f5af-4307-805b-3a421cdabd76
Finding project entities in bundle...
1
Finding study entities in bundle...
1
Finding sample entities in bundle...
17
Finding sequencingExperiment entities in bundle...
1
Finding sequencingRun entities in bundle...
1
...
Entities to be converted: {
    "project": 1,
    "study": 1,
    "sample": 19,
    "sequencingExperiment": 6,
    "sequencingRun": 6
}
Saving Report file...
Saved to /home/me/ingest-archiver/ARCHIVER_2019-01-04T115615/REPORT.json!
##################### FILE ARCHIVER NOTIFICATION
Saved to /home/me/ingest-archiver/ARCHIVER_2019-01-04T115615/FILE_UPLOAD_INFO.json!
```

5. In your current directory, the MA will have generated a directory with the name `ARCHIVER_<timestamp>` containing two files, `REPORT.json` and `FILE_UPLOAD_INFO.json`. Inspect `REPORT.json` for errors.

6. `FILE_UPLOAD_INFO.json` contains the instructions necessary for the file uploader to convert and upload submission data to the USI. Copy this file to the HCA NFS namespace via the cluster. For this step you must be connected to the EBI internal network

```scp FILE_UPLOAD_INFO.json ebi-cli.ebi.ac.uk:/nfs/production/hca```

7. Login to EBI CLI with your EBI password
`ssh ebi-cli.ebi.ac.uk`

8. Run the file uploader with this command (this is for test, for production you would need to replace the `-l` switch with `-l=https://api.aai.ebi.ac.uk/auth`

If the job fails during bundle download simply re-run the above command

`bsub 'singularity run -B /nfs/production/hca:/data docker://quay.io/humancellatlas/ingest-file-archiver -d=/data -f=/data/FILE_UPLOAD_INFO.json -l=https://explore.api.aai.ebi.ac.uk/auth -p=<ebi-aap-password> -u=hca-ingest'`

This will now take a very long time to run if you have a large dataset where data needs format conversion. The job may also remain in pending if the cluster is busy. You can check whether your job is running in the cluster with the command

`bjobs -W`

Here are some further useful links about using the cluster and associated commands.

https://sysinf.ebi.ac.uk/doku.php?id=sysinf_computing
https://sysinf.ebi.ac.uk/doku.php?id=ebi_cluster_good_computing_guide
https://sysinf.ebi.ac.uk/doku.php?id=introducing_singularity

Once the data upload has finished or failed the cluster will send you an e-mail.

9. If the metadata archiver is still running because you set `--submit` flag above then wait for that to report success. If it isn't or you stopped it, run it with this switch to validate and submit the archive entries.

`./cli.py --submission_url="https://submission-dev.ebi.ac.uk/api/submissions/<submission-uuid>"`

The submission uuid can be found either in the output of the initial metadata archiver run, e.g.

`USI SUBMISSION: https://submission-dev.ebi.ac.uk/api/submissions/b729f228-d587-440c-ae5b-d0c1f34b8766`

or in the `REPORT.json` in the `submission-url` field (there will be several). For example,

`"submission_url": "https://submission-dev.ebi.ac.uk/api/submissions/b729f228-d587-440c-ae5b-d0c1f34b8766"`

On success you will get the message `SUCCESSFULLY SUBMITTED`. You're done!

## Running the data uploader outside of singularity.
For test purposes you can run the data uploader outside of singularity with the command

`docker run --rm -v $PWD:/data quay.io/humancellatlas/ingest-file-archiver -d=/data -f=/data/FILE_UPLOAD_INFO.json -l=https://api.aai.ebi.ac.uk/auth -p=<password> -u=hca-ingest`

# How to run the tests

```
python -m unittest discover -s tests -t tests

```

# Versioning

For the versions available, see the [tags on this repository](https://github.com/HumanCellAtlas/ingest-archiver/tags).

# License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details
