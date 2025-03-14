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
  - could be edited for resubmission e.g. upon fixing accession #?

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
a composite "study" dataset.  This is the approach of A2CPS and somewhat of the [Allen Institute of Neural Dynamics](https://github.com/bids-standard/bids-2-devel/issues/60).  This aligns with potential developments in BIDS

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
- Converting/importing a particular anatomical from another study/session
  into another where it was not collected
  - allow for specific sequence(s) to be converted into a specific study

## More elaborate DICOMs listing (former step 1)

```yaml=

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
