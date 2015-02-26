***************************
Analyze Apache Web Log Data
***************************

This script pipes the apache log data through a log converter to get the logs into a csv format, into the Essentia preprocessor, and then into the udbd database.

The preprocessor allows more efficient loading of data by ignoring the irrelevant columns in the web logs and creates a column to keep track of the number of records.

Attributes are applied in the database and the number of records corresponding to each unique referrer is counted.

Then the 25 referrers that corresponded to the most records in the web log data are output and the total number of unique referrers is displayed.

A Brief Description of What This Script Does
============================================

**Line 5** 

* Store a vector in the database apache that aggregates the values in the pagecount column for each unique referrer. 
* The pagecount column only contains the number '1' so this serves to count the number of times any one referrer was seen in the web logs.

**Line 13** 

* Create a new rule to take any files with '/2014' followed by another '/2014' in their name and put them in the 2014logs category.

**Line 18** 

* Pipe the files in the category 2014logs that were created between April 1st and 5th, 2014 to the aq_pp command. 
* In the aq_pp command, tell the preprocessor to take data from stdin, ignoring errors and not outputting any error messages. 
* Then define the incoming data's columns, skipping all of the columns except referrer, and create a column called pagecount that always contains the value 1. 
* Then import the data to the vector in the apache database so the attributes listed there can be applied.

**Line 20** 

* Export the aggregated data from the database, sorting by pagecount and limiting to the 25 most common referrers. Also export the total number of unique referrers.

..  code-block:: sh
    :linenos:
    :emphasize-lines: 5,13,18,20

    ess instance local
    ess spec drop database apache
    ess spec create database apache --ports=1
    
    ess spec create vector vector1 s,pkey:referrer i,+add:pagecount
    
    ess udbd start
    
    ess datastore select s3://*Bucket*/data/openid/ --aws_access_key=*AccessKey* --aws_secret_access_key=*SecretAccessKey*
    
    ess datastore scan
    
    ess datastore rule add "*/2014*/2014*" "2014logs" "/YYYYMMDD"
    
    ess datastore probe 2014logs --apply
    ess datastore summary
    
    ess task stream 2014logs "2014-04-01" "2014-04-05" "logcnv -f,eok - -d ip:ip sep:' ' s:rlog sep:' ' s:rusr sep:' [' i,tim:time sep:'] \"' s,clf,hl1:req_line1 sep:'\" ' i:res_status sep:' ' i:res_size sep:' \"' s,clf:cookie sep:'\" \"' s,clf:referrer sep:'\" \"' s,clf:user_agent sep:'\" ' i:dt sep:' ' s:url_base sep:' ' s:con_status sep:' ' x | aq_pp -f,qui,eok - -d X X X X X X X X X X s:referrer X X X X -evlc i:pagecount "1" -ddef -udb_imp apache:vector1" --debug
    
    ess task exec "aq_udb -exp apache:vector1 -sort pagecount -dec -top 25; aq_udb -cnt apache:vector1" --debug
    
    ess udbd stop
    
This script provided a very simple analysis of Apache Log Data using Essentia. To run the complete version of our
Apache Log Demo, including more advanced analysis using Essentia and R, please follow the steps in
`Getting Started with the R Integrator <http://www.auriq.net/documentation/source/usecases/r-format-requirements.html>`_.