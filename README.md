TODOs:

- Model needs to be fixed up from our blunt typing it in

    ❯ linkml lint --validate dicomsmodel.yaml
    /home/yoh/proj/dbic/dicom-workflow/dicomsmodel.yaml
      error    In classes > DICOMAccession > attributes: [{'series': {'range': 'DICOMSeries', 'multivalued': True}}, {'tags': {'description': 'Mostly computed based on DICOMSeries[].study_session[].tags', 'range': 'Tag', 'multivalued': True}}] is not of type 'object', 'null'  (valid-schema)
      error    In classes > StudyParticipant > slots > [0]: {'anon_id': {'description': "ID of the subject in the study (e.g. '01')", 'recommended': True}} is not of type 'string'  (valid-schema)
      error    In classes > StudySession > slots > [3]: {'tags': {'range': 'Tag', 'multivalued': True}} is not of type 'string'  (valid-schema)
      error    In classes > DICOMSeries > slots > [0]: {'paths': {'multivalued': True}} is not of type 'string'  (valid-schema)
      error    In classes > Skippable > attributes > skip: Additional properties are not allowed ('default' was unexpected)  (valid-schema)
