# Plaso
Plaso Related Files

Run to log2timeline using the triage filter.
```
log2timeline.py --status_view window -z UTC --partitions all --vss_stores all -f triage_filter.txt --hashers all --hasher_file_size_limit 0 image_file.e01
```
Backup the timelime just in case something go wrong with the tagging.

```
copy NT_triage_timeline.plaso triage_timeline.plaso
```

Tag the plaso file using the triage tags.
```
psort.exe -z UTC -o null --analysis tagging --tagging-file triage_tags.txt triage_timeline.plaso
```
Export the data.
```
psort.exe -z UTC -o xlsx -w triage.xlsx --additional_fields data_type,hostname,computer_name,event_identifier,md5_hash,sha1_hash,username,user_sid,xml_string triage_timeline.plaso
```
