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



