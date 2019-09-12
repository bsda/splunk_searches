# Random splunk search ideas

##### 2 Search ideas to to try to find if we are receiving the correct amount of data based on averages of same hour and same day for multiple weeks in the past

**Using streamstats:**
```
|tstats count index, _time span=1h
| sort 1-_time
| eval weekday=strftime(_time,"%a")
| eval week_hour=strftime(_time, "%H")
| eval today=strftime(now(), "%a")
| eval hour=strftime(relative_time(now(), "-1h"), "%H")
| eval sameDay=if(today=weekday AND hour = week_hour ,1,0)
| search sameDay=1
| streamstats avg(count) AS avg, median(count) AS median, stdev(count) AS stdev BY index
| streamstats current=f global=f window=1 latest(avg) as last_avg, latest(median) AS latest_median, latest(stdev) AS latest_stdev by index
```


**Using map:**
```
| tstats count where index!="_*" index!="*summary" by index
| map [|tstats count where index=$index$ by index, _time span=1h 
    | timechart span=1h sum(count) AS count by index 
    | timewrap 1week 
    | rename *_latest_week AS *_current 
    | fillnull value=0 
    | addtotals *week*_before fieldname=total col=true 
    | eval weeks=0 
    | foreach *week* 
        [ eval weeks=if(isnotnull('<<FIELD>>'), weeks + 1, weeks) ] 
    | eval avg=round(total/weeks) 
    | sort - _time | head 1
    | eval index="$index$"
    | foreach *_current [ eval latest='<<FIELD>>' ] 
 | table index avg latest total  weeks ] maxsearches=100
```


### Get size of lookups  from search
**My simple version counting only lines (which is not useful)**
```
| rest /services/data/lookup-table-files splunk_server=local  
| fields eai:data 
| rename eai:data AS file
| eval file=mvindex(split(file, "/"), -1), filename=file 
|  map maxsearches=10000 search="|inputlookup $file$ | eval filename=$file$ | stats c by filename"
```

**The really good version found in splunk answers**
https://answers.splunk.com/answers/701881/is-there-a-search-to-show-bundle-size-in-the-dispa.html

```
|rest/services/data/lookup-table-files splunk_server=local
 
 | rename dispatch.* AS *
 | rename eai:acl.* AS *
 | map maxsearches=9999 search="
 | inputlookup $title$
 | rename COMMENT1of3 AS \"Some field names have single-quotes which will cause this error:\"
 | rename COMMENT3of3 AS \"{map}: Failed to parse templatized search for field 'Bad Field's Name Here'\"
 | rename COMMENT3of3 AS \"So rename those fields before we process them to replace ' with _\"
 | rename *'*'*'*'* AS *_*_*_*_*, *'*'*'* AS *_*_*_*, *'*'* AS *_*_*, *'* AS *_*
 | eval T3MpJuNk_bytes=0, T3MpJuNk_cols=0, T3MpJuNk_field_names=\",\"
 | foreach _*
 [ eval T3MpJuNk_bytes = T3MpJuNk_bytes + coalesce(len('<<FIELD>>'), 0)
 | eval T3MpJuNk_cols = T3MpJuNk_cols + 1
 | eval T3MpJuNk_field_names = T3MpJuNk_field_names . \"<<FIELD>>\"]
 | rename _* AS *, T3MpJuNk_* AS _T3MpJuNk_*
 | foreach *
 [ eval _T3MpJuNk_bytes = _T3MpJuNk_bytes + coalesce(len('<<FIELD>>'), 0)
 | eval _T3MpJuNk_cols = _T3MpJuNk_cols + 1
 | eval _T3MpJuNk_field_names = _T3MpJuNk_field_names . \"<<FIELD>>\"]
 | rename COMMENT AS \"Account for the commas, too!\"
 | eval bytes = bytes + (cols - 1)
 | stats sum(_T3MpJuNk_bytes) AS bytes count AS lines first(_T3MpJuNk_cols) AS cols first(_T3MpJuNk_field_names) AS field_names
 | rename COMMENT AS \"Account for the header line, too!\"
 | eval bytes = bytes + (len(field_names) - 1)
 | eval title=\"$$title$$\"
 | eval owner=\"$$owner$$\""
 | eval bytes = coalesce(bytes, 0)
 | addtotals row=false col=true labelfield=title label="$TOTAL_FIELD_VALUE$"
 | eval "bytes/line" = if(title=="$TOTAL_FIELD_VALUE$", "N/A", round(coalesce(bytes/lines, 0), 2))
 | eval owner = if(title=="$TOTAL_FIELD_VALUE$", "N/A", owner)
 | eval cols  = if(title=="$TOTAL_FIELD_VALUE$", "N/A", coalesce(cols, "N/A"))
 | eval MB = round(bytes / 1024 / 1024, 2)
 | eval bundlePct = round(100 * bytes / 838860800, 2)
 | eval status=case(
    title=="$TOTAL_FIELD_VALUE$", if((bundlePct < 90),                         "OK", "DANGEROUS TERRITORY"),
    true(),                       if((bundlePct < 25 AND lines < 10000000), "OK", "Consider KVStore"))
 | sort 0 - bytes
 | table title status bundlePct owner bytes MB lines cols bytes*line
 | eval _drilldown  = if(title=="$TOTAL_FIELD_VALUE$", "*", title)
```
