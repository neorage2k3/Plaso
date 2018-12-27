# Cookbook Plaso 20181219

* This is a summary of what I've learned having to start from zero knownlege of the Plaso tool suite.
* At the time of this writing, the `--workers` and  `--worker_memory_limit` were not functioning on my real world test Windows 7 test image using fresh install of Ubuntu 18.04 in a VMWare Workstation with 16G RAM and 8 Processor cores. Using these options resulted in unhandled out of memory errors.  To successfully run log2timeline, the `--single_process` had to be utilzed. Plaso was installed from the GIFT repo.  Older version of Plaso ran find when using the offending options.  I was able to use plaso-20181219-py3.6-amd64.zip successfully on a bare metal Win10 installation without issue (so far).

# Suggestions
* Never run Log2timeline with out constraints.
* Create a VM dedicated to Plaso.  Using a dedicated VM keeps Plaso from hogging all the resources on the host OS.  
* If running on the host OS or if processing power for other applications is needed, use the `--workers` setting  option to something reasonable.
* Create an export filter for the artifacts that can or need to be processed by other tools; ie regripper.
* If running Plaso against a mounted drive image, run all commands against the mounted image as root/admin.
* Consider running Plaso in multiple targeted passes, each pass only getting what is needed.
* Output the help of log2timeline and psort to a text file for quick reference. 
* Each option used adds a certain amount of time, choose options carefully.

# Suggested VM Setup
* As many processors possible.
* As much memory that can be spared that aligns with the number of processors given.  I suggest at least 2G per processor.
* Clean Ubuntu LTS or supported Linux Distro.
* Latest Plaso. (https://plaso.readthedocs.io/en/latest/sources/user/Ubuntu-Packaged-Release.html)

# Filtering
## Collection Filter
* A Collection filter is a user created filter that assists in targeting data for collection or processing into a timelime.
* Create a specific filter for exporting of data and one for ingesting into a master timeline.

## Sample Filters options (-f).  
 Here are some sample lines:
```
/[$]Boot
/[$]Extend/[$]UsnJrnl
/[$]LogFile
/[$]MFT
/[$]Secure
/[$]Recycle.Bin
/[$]Recycle.Bin/.+
/[$]Recycle.Bin/.+/.+
/Windows/Prefetch/.+
/Windows/System32/config/(SAM|SOFTWARE|SECURITY|SYSTEM)
/Windows/System32/winevt/Logs/.+[.]evtx
/Users/.+/NTUSER.DAT
/Users/.+/AppData/Local/Microsoft/Windows/Usrclass[.]dat
```
# General Walkthough
1. Create collection filter
2. Create the timeline filter
3. Use image_export to export the data needed for further analsys with other tools.
4. Use log2timeline to create the timeline plaso file
5. Use psort to enrich the data
6. Use psort to extract slices of data for analsys. 
7. Use pinfo to extract data for case notes.

# Export data for analysis with other tools
* Data collections from the volume/partitions and the volume shadow servers (VSS) cannot be collected in one pass or exported to the same folder. 
* Create a top level export directory "exports_path."  This will be the top level of the exported data. 
* Use the following to export the desired data to different top level structures. Plaso will keep the directory structure from the image.
```
image_export.py -f collection_filter.txt --partitions all --no_vss -w exports_partition_path image_file.e01
image_export.py -f collection_filter.txt --vss_only --vss_stores all -w exports_path_vss image_file.e01
```
# Timeline the Collected Artifacts 
* The following may work:
```
log2timeline.py -f collection_filter.txt --status_view window --partitions all --vss_stores all --hashers md5,sha1 --hasher_file_size_limit 0 timeline.plaso image_file.e01
```
* But use two passes like image_export to be sure:
```
log2timeline.py -f collection_filter.txt --status_view window --partitions all --no_vss --hashers md5,sha1 --hasher_file_size_limit 0 timeline.plaso image_file.e01
log2timeline.py -f collection_filter.txt --status_view window --vss_only --vss_stores all --hashers md5,sha1 --hasher_file_size_limit 0 timeline.plaso image_file.e01
```

# Adding more constraints 
## Understanding Parsers
* Supported parsers can be found by running `log2timeline.py --parsers list`
* One good method of creating timelines is around filters and/or parsers.
* Creating timelines based on case needs will be all be faster than ingesting all artifacts.
* Here is an example of using a specific parsers
```
log2timeline.py --status_view window -f collection_filter.txt --status_view window --partitions all --no_vss --parsers win7 -hashers md5,sha1 --hasher_file_size_limit 0 timeline.plaso image_file.e01
log2timeline.py --status_view window --parsers mactime timeline.plaso body_file.body
```
## Understanding Workers
* Use the `--workers` or `--single_process` to reduce the procressing power log2timeline can use.  This is important when processing power is needed for other activites on the host.
```
log2timeline.py --single_process -f timeline_filter.txt --status_view window --partitions all --vss_stores all --hashers md5,sha1 --hasher_file_size_limit 0 timeline.plaso image_file.e01
log2timeline.py --workers 6 --worker_memory_limit 1073741824 -f timeline_filter.txt --status_view window --partitions all --vss_stores all --hashers md5,sha1 --hasher_file_size_limit 0 timeline.plaso image_file.e01
```
#  Putting it all together using common options
```
log2timeline.py --no_dependencies_check --workers 6 -f timeline_filter.txt --status_view window ---partitions all ---no_vss --parsers win7 --hashers md5,sha1 --hasher_file_size_limit 0 timeline.plaso image_file.e01
log2timeline.py --no_dependencies_check --workers 6 -f timeline_filter.txt --status_view window --vss_only --vss_stores all --parsers win7 --hashers md5,sha1 --hasher_file_size_limit 0 timeline.plaso image_file.e01
```

# Processing Archive files
* The `--process_archives` switch can be used to process the files inside of archive files.
* Use a filter to reduce the files or directories processed.
```
log2timeline.py -f archive_filter.txt --process_archives --status_view window --partitions all ---no_vss --parsers win7 --hashers md5,sha1 --hasher_file_size_limit 0 timeline.plaso image_file.e01
log2timeline.py -f archive_filter.txt --process_archives --status_view window --vss_only --vss_stores all --parsers win7 --hashers md5,sha1 --hasher_file_size_limit 0 timeline.plaso image_file.e01
```
# Add Plaso pre-defined Tags
* At a minimum use the pre-defined tagging included with the Plaso install.
* Plaso's pre-defined taggs are OS specific and can be `/usr/share/plaso/` on Ubuntu or `plaso\data\` on Windows.
```
psort.py --analysis tagging --tagging-file /usr/share/plaso/tag_windows.txt -o null timeline.plaso
-or -
psort.py--analysis tagging --tagging-file plaso\data\tag_windows.txt -o null timeline.plaso
```
# Add NSRL Tags
* If a NSRLSVR is available use the following to add the NSRL tags.
* [Info on NSRLSVR](https://rjhansen.github.io/nsrlsvr/)
```
psort.py --analysis nsrlsvr --nsrlsvr-hash md5 --nsrlsvr-host ipaddress --nsrlsvr-port 9120 --nsrl-label NSRL262 -o null timeline.plaso
```

# Analysis
## Output file options
* Use `psort.py -o list` to find the available output options
* Using json as an output type is useful in determining additional fields.
* Sample JSON extract 24 hours period, 12 hours each side of a given time stamp.
```
psort.py --slice "2018-12-22 12:00:00 PM" --slice_size 720 -o json -w slicetime.json -q timelime.plaso
```
* Overall, I recommend the use of the xlsx with addtional fields on data sliced by time constraints.

# More Slicing examples
* Slice out a 24 hour period, 12 hour each side of a timestamp, using xlsx output and addtional fields
```
psort.py --slice "2018-04-16 12:00:00 PM" --slice_size 720 -o xlsx -w slice_20180416.xlsx -q --additional_fields hostname,computer_name, event_identifier, md5_hash, sha1_hash, location, username, user_sid, xml_string timeline.plaso
```
* Slice by date range
```
psort.py -o xlsx -w slice_20180227.xlsx -q --additional_fields hostname,computer_name,event_identifier,md5_hash,sha1_hash,location,username,user_sid,xml_string timeline.plaso "date < '2018-02-28 00:00:00' and date > '2018-2-27 00:00:00'"
```

# Other considerations when/if time is not a factor
* These commands will use all available processsing power and will automatically use all parsers.  
* All items will be hashed with MD5, SHA1 and SHA256.  
* These commands can take multiple days and is only recommended for testing and learning
```
log2timeline.py --no_dependencies_check --status_view window --partitions all --no_vss --process_archives --hashers all --hasher_file_size_limit 0 timeline.plaso image_file.e01
log2timeline.py --no_dependencies_check --status_view window --vss_only --vss_stores all --process_archives --hashers all --hasher_file_size_limit 0 timeline.plaso image_file.e01
```

# Export documentation
* The plaso file contains good information to add to case notes and to track what was ingested.
* Run this command after all ingesting of data is accomplished.
```
pinfo.py -v timeline.plaso > plaso.log
```

# Misc
* There is much more to Plaso than this document has covered.

# Noteable references:
* [https://plaso.readthedocs.io/en/latest/](https://plaso.readthedocs.io/en/latest/)
* [https://plaso.readthedocs.io/en/latest/sources/user/Ubuntu-Packaged-Release.html](https://plaso.readthedocs.io/en/latest/sources/user/Ubuntu-Packaged-Release.html)
* [https://digital-forensics.sans.org/media/Plaso-Cheat-Sheet.pdf](https://digital-forensics.sans.org/media/Plaso-Cheat-Sheet.pdf)
* [https://github.com/mark-hallman/plaso_filters/blob/master/filter_windows.txt](https://github.com/mark-hallman/plaso_filters/blob/master/filter_windows.txt)
* [https://github.com/mark-hallman/plaso_filters/blob/master/gather_evidence_of_query.txt](https://github.com/mark-hallman/plaso_filters/blob/master/gather_evidence_of_query.txt)
