# Three layers

## Current situation

More details at https://github.com/repronim/reproin?tab=readme-ov-file#complete-setup-at-dbic


0. Initial "receiver" setup/configuration

- we rely on each scanning "accession" (could be multiple session UUIDs at DICOM level if interrupted) to reside in a separate folder
- we rely on ReproIn organization of the study protocols and naming of the sequences
- we rely on patient_ID to be set in DICOM
- we rely on "accessions to convert" to either start with A0 or be equal to "qa"

1. cron job to extract metadata from recent DICOMs to update listing of accessions (`lists-update`)

- ad-hoc text files - should be made machine readable
- only scouts -- should go through all
- only locator, subject, session -- should be much more (may be full heudiconv record)

Also this one
- updates ad-hoc shell script per study with what was already done and what to be done, and what warnings
- sends out email with summary across studies

problem:
- "the window" of view -- only since beginning of the month

2. create study dataset

links repronim/containers with current/specific version of the conversion container

problem:
- just BIDS, not project level

3. run conversion

- if all is good (no warnings from step 1), and it is recent dataset with container:  reproin study-convert STUDY-NAME  in the study directory - does the job

problems:
- slow:
    - it rescans all listings from step 1, and looks inside the dataset to decide if particular subject/session was converted already
    - heudiconv re-extracts dicom metadata
- maintains the list of what accessions to not convert within .heudiconv/sid-skip

4. mriqc, defacing, ...

not there

problems:
- fMRIPrep in particular is "session aware" (aggregates across sessions for certain products); thus, if adding a new session for a particular subject, that subject's processing should be rerun, which would require some kind of detection/triggering, or alternatively some way of delaying fMRIPrep until collection is finished)


## Typical screw ups

Some of those might be "detected" and prevent automatic conversion requiring human decisision on how to proceed. Ideally -- should figure out annotation for those at MRI console level.

TODO:
- check that "Additional info" textedit box on "Patient registration":
  - where in DICOM?
  - could be edited for resubmission e.g. upon fixin accession #? 

- Rescanning of previously scanned
  - ses X was rescanned again (and may be already converted)
      - TODO: see how to annotate at MRI console level to "automate" 
  - partial rescanning: only some sequences from new session to be added to old one
  - ~~partial rescanning within another session~~
- Human entry issues
  - wrong accession number
    - resolution: fix on MRI console, resubmit to PACS, gets a new folder, old folder deleted by admin some time
  - session indicator is absent in first scans but then introduced
  - session indicator is wrong 
      - should be 02 but was 01 and 01 was already collected - we can detect
      - should be 02 but was 03 -- is 02 missing?
- Magnet screw up
    - needs restart of the scanning session. That resets the sequence counter, "sorting" should be done by time, not seq number

## Features kinda missing but would be nice to add "by default"

- automated mapping of DBIC Subject IDs and per-study IDs

  we have ad-hoc script: https://github.com/nipy/heudiconv/blob/master/utils/anon-cmd 
  
# Modes of operation

- Convert only what project curator approved for conversion (may be with fixups)
- Convert all until any ambiguity
- Convert all which could be converted without ambiguity

Trickier use-cases

- **Re**-convert what was "corrected":
    - then prior ones need to be removed first. And because heudiconv has "memory" of stuff he was told to do/done under .heudiconv/sub/ses/ -- we better redo the entire session, not just few scans.

# Redesign ideas

## Potential "alternative" modularization

Convert each accession into independent BIDS dataset to later be collated into
a composit "study" dataset.  This is the approach of A2CPS and somewhat of the [Allen Institute of Neural Dynamics](https://github.com/bids-standard/bids-2-devel/issues/60).  This aligns with potential developments in BIDS

- https://github.com/bids-standard/bids-2-devel/issues/59 

Pros:

- simplifies session processing (no need to mess with filtering for fmriprep execution etc)
  - fmriprep needs ideally multiple sessions at once 
- allows for composition into multiple datasets

Cons:

- needs additional "aggregation" step
- no immediate bids-validator across all subject/sessions
- if DataLad'ified
  - might have merge conflicts in top level files etc so would require 
    additional tooling to do merge correctly

## Reuse of accessions or sequences in multiple datasets

E.g.

- Yarik's phantom study collected anatomicals from some studies
  - pretty much conversion script needs to be given a set of scans,
    without full accession folders
- Converting/importing a particualr anatomical from another study/session
  into another where it was not collected
  - allow for specific sequence(s) to be converted into a specific study
  
## More elaborate DICOMs listing (former step 1)

```yaml=
id: https://w3id.org/dbic/dicomsmodel
name: dicomsmodel
description: |-
  Information about DICOMS and studies they are associated with
license: https://creativecommons.org/publicdomain/zero/1.0/
imports:
  - linkml:types
prefixes:
  personinfo: https://w3id.org/linkml/examples/personinfo/
  linkml: https://w3id.org/linkml/
  schema: http://schema.org/
  xsd: http://www.w3.org/2001/XMLSchema#

types:
  string:
    uri: xsd:string
  integer:
    uri: xsd:integer
  float:
    uri: xsd:float
  boolean:
    uri: xsd:boolean

classes:
    Skippable:
        attributes:
          skip:
            range: boolean
            default: False

    Person:
        class_uri: schema:Person
        slots: 
        - first_name
        - last_name
        - alias  # typically just last name?
        - email
        # - role ?
      
    PI:
      description: Who pays for the experiment
      is_a: Person

    Experimenter:
      description: Who leads the experiment (e.g. a student)
      is_a: Person
    
    ## TODO: may be make it "project"
    Study:
      description: The actual experiment study
      slots:
      - id
      - codename
      attributes:
        pi:
          range: PI
          required: True
        experimenter:
          range: Experimenter
        participants:
          range: StudyParticipant
          multivalued: True
        sessions:
          range: StudySession
          multivalued: True
          
    Participant:
      slots:
      - id   # DBIC id (e.g. sid000323)
      - gender
          
    StudyParticipant:
      is_a: Participant
      slots:
      - anon_id:
          ## TODO: might need more clarification
          description: ID of the subject in the study (e.g. '01')
          recommended: True  
      attributes:
        tags:
          range: Tag
          multivalued: True
        sessions:
          range: string
          multivalued: True
    
    StudySession:
      slots:
      - age
      - weight
      - date
      - tags:
          range: Tag
          multivalued: True
      attributes:
        participant:
          range: StudyParticipant

    DICOMAccession:
      description: >-
        Collection of DICOM series, such as /inbox/DICOM/2025/03/11/qa
      is_a: Skippable
      slots:
        - path
        - date
        - study  # a study this set was associated with at DICOM level, could be edited 
        - subject
      attributes:
        - series:
            range: DICOMSeries
            multivalued: True
        - tags:
            description: Mostly computed based on DICOMSeries[].study_session[].tags
            range: Tag
            multivalued: True

    DICOMSeries:
      description: >-
        Collection of DICOM file(s), such as /inbox/DICOM/2025/03/11/qa/005-func-bold_task-rest_acq-p2
      is_a: Skippable
      slots:
        - paths:  # directory or a list of files (if flat listing)
            multivalued: True
        ## may be in dicom_metadata - date
        - session
        - dicom_metadata:
            description: auto-extracted/not-changed, likely what heudiconv extracts now
            range: SeqInfo
        - study_sessions:
            range: StudySession
            multivalued: True
    
  SeqInfo:
    description: Information about an MRI sequence, based on HeuDiConv SeqInfo
    attributes:
      total_files_till_now:
        description: Total number of files processed till now
        range: integer
      example_dcm_file:
        description: Example DICOM file
        range: string
      series_id:
        description: Series identifier
        range: string
      dcm_dir_name:
        description: DICOM directory name
        range: string
      series_files:
        description: Number of files in the series
        range: integer
      unspecified:
        description: Unspecified field
        range: string
      dim1:
        description: First dimension
        range: integer
      dim2:
        description: Second dimension
        range: integer
      dim3:
        description: Third dimension
        range: integer
      dim4:
        description: Fourth dimension
        range: integer
      TR:
        description: Repetition Time
        range: float
      TE:
        description: Echo Time
        range: float
      protocol_name:
        description: Protocol name
        range: string
      is_motion_corrected:
        description: Whether motion correction was applied
        range: boolean
      is_derived:
        description: Whether this is a derived dataset
        range: boolean
      patient_id:
        description: Patient identifier
        range: string
        required: false
      study_description:
        description: Description of the study
        range: string
      referring_physician_name:
        description: Name of the referring physician
        range: string
      series_description:
        description: Description of the series
        range: string
      sequence_name:
        description: Name of the sequence
        range: string
      image_type:
        description: Type of image
        multivalued: true
        range: string
      accession_number:
        description: Accession number
        range: string
      patient_age:
        description: Age of the patient
        range: string
        required: false
      patient_sex:
        description: Sex of the patient
        range: string
        required: false
      date:
        description: Date of acquisition
        range: string
        required: false
      series_uid:
        description: Unique identifier for the series
        range: string
        required: false
      time:
        description: Time of acquisition
        range: string
        required: false
      custom:
        description: Custom data field
        range: string
        required: false
      patient_weight:
        description: Weight of the patient
        range: float
        required: false

enums:
  Tag:
    permissible_values:
      RECEIVED:
      BIDSIFIED:
        description: Was already converted into a BIDS dataset
      REMOVED:
      RENAMED:
      # May be if to also model the submission to MRI
      TO_SUBMIT_TO_MRI:
      SUBMITTED_TO_MRI:
  Action:
    permissible_values:
      BIDSIFY:
      REBIDSIFY:
      REMOVE_BIDSIFIED:
      
```


    

## Study/datasets level configurations

- subject_id anonymization
- mapping of protocol names into reproin (done for a2cps, spacetop etc)
    - nice would be to be able to assess if changes to mapping change any already converted names (thus might be redone or mapping fixed up)
- decisions on IntendedFor
    - for a specific subj/session to relax/change intendedfor assignment
    - some groups have other ideas on how to assign fieldmaps (e.g. lower noise etc)
- BIDS-apps etc -- for later. May be in conjunction with https://github.com/nipoppy/nipoppy 

## Helpers to develop to "lay it out" over our data

- given collected DICOMs, populate the DB (so replace our adhoc lists)
  - if already a known accession - ERROR for now. Alternatives: is it a duplicate or partial duplicate etc 
- given a converted BIDS dataset, adjust records for its accessions etc within our "DB"
  - we need to allow for "old style" (project less organization) and new, so explicit **path** might be needed 