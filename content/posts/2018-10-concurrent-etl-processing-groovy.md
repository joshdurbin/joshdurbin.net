+++
title = "Concurrent eTl processing in groovy"
description = "Converting and structuring government NPI entries for direct import into Mongo Collections"
date = "2018-10-31"
tags = ["etl", "groovy", "mongo" ]
+++

This is a throw back to a PoC [project](https://github.com/joshdurbin/cms-npi-rest) I developed a few years back for a leading health insurance
provider; a REST API for pulling out provider details with batched updates as available from 
the [Centers for Medicare & Medicaid Services (CMS)](https://www.cms.gov/). CMS keeps track of 
health care providers in a [National Provider Identifier](https://en.wikipedia.org/wiki/National_Provider_Identifier)
(NPI) database. CMS doesn't make this data available via an API, but does allow for the [download](http://download.cms.gov/nppes/NPI_Files.html)
 of the entire dataset of roughly 5.7 million entries or incremental updates.
 
This post is not so much focused on the project, but the conversion steps and process used to convert
that 5.7 million, 6.4 gigabyte file into the necessary data structures required for direct import into
Mongo. The [original script](https://github.com/joshdurbin/cms-npi-rest/blob/data_cleaning_scripts/FormatNPIData.groovy)
took a number of minutes to run, scan through the file, process each row gathering
statistics on organizations and codes, and transform the output into cleaner JSON in the form of both
organizations and individuals.

Each line in the CSV looks something like the following and in:

`[1619971173, 2, null, <UNAVAIL>, MID ERIE FAMILY PRACTICE LLC, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, 1611 PEACH ST, STE 320, ERIE, PA, 165012122, US, 8144562003, 8144564098, 1611 PEACH ST, STE 320, ERIE, PA, 165012122, US, 8144562003, 8144564098, 06/11/2005, 11/10/2009, null, null, null, null, LEEMHUIS, RONALD, PAUL, LLC MEMBER, 8144562003, 207Q00000X, null, null, Y, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, 007547490, 05, PA, null, 767424, 01, PA, BLUE CROSS/BLUE SHIELD, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, N, null, null, DR., null, M.D., 193200000X MULTI-SPECIALTY GROUP, null, null, null, null, null, null, null, null, null, null, null, null, null, null]`

...and the time of this writing has a mapping guide that is roughly equivalent to the following containing:

- NPI code
- Entity type (individual v. organization)
- Names
- Address (two)
- Other NPI Data, Dates
- 15 Taxonomy Switches  
- 15 Taxonomy Groups
- 50 "Other" Provider Identifiers 

More information on the mapping can be found on the [CMS NPI Data Dissemination page](https://www.cms.gov/Regulations-and-Guidance/Administrative-Simplification/NationalProvIdentStand/DataDissemination.html).
