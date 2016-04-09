# Exploratory Analysis of Weather Events with High Mortality and Economic Impact
Wayne Witzke  

## Synopsis
In this exploratory analysis, NOAA weather event data was processed and
examined to determine which weather events were most likely to have the largest
economic and health impact. Analysis focused on fatalities, injuries, and crop
and property damage data obtained after 1996. It was determined that
hurricanes, tsunamis, heat, and storm surge were the events with the highest
impact.

## Preliminaries

#### Globals
This analysis presumes that there is a specific directory structure in place.
That structure is defined here.

```r
knitr::opts_chunk$set( fig.path = "figure/" );
rawdata.dir = "raw";
tidydata.dir = "data";
rawfile = file.path( rawdata.dir, "RawData" );
tidyfile = file.path( tidydata.dir, "TidyData.rds" );
library(data.table);
library(stringr);
```

#### System Information
This analysis was performed using the hardware and software listed in this
section.


```r
sessionInfo();
```

```
## R version 3.2.2 (2015-08-14)
## Platform: x86_64-pc-linux-gnu (64-bit)
## Running under: Ubuntu 15.10
## 
## locale:
##  [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
##  [3] LC_TIME=en_US.UTF-8        LC_COLLATE=en_US.UTF-8    
##  [5] LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8   
##  [7] LC_PAPER=en_US.UTF-8       LC_NAME=C                 
##  [9] LC_ADDRESS=C               LC_TELEPHONE=C            
## [11] LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       
## 
## attached base packages:
## [1] stats     graphics  grDevices utils     datasets  methods   base     
## 
## other attached packages:
## [1] stringr_1.0.0    data.table_1.9.6
## 
## loaded via a namespace (and not attached):
##  [1] magrittr_1.5    formatR_1.3     tools_3.2.2     htmltools_0.3.5
##  [5] yaml_2.1.13     Rcpp_0.12.3     stringi_1.0-1   rmarkdown_0.9.5
##  [9] knitr_1.12.3    digest_0.6.9    chron_2.3-47    evaluate_0.8.3
```

## Data Processing

#### Data collection and processing overview
This analysis uses data from the NOAA storm database. This data is
programmatically downloaded from [this](https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2)
location as part of the analysis. Information about this data set can be found
at the National Weather Service [Storm Data Documentation](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf)
and the National Climatic Data Center Storm Events [FAQ](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2FNCDC%20Storm%20Events-FAQ%20Page.pdf).

Several parts of the data tidying process are packaged in whole or in part as
reusable and are replicated here as functions.

#### GetRawFile function
This function will fetch a file from an online location, timestamp it, and then
return the timestamp. It only does this if the `output.file` doesn't already
exist. If it does, then it will merely return the timestamp for the already
saved raw data set.

This function will also create the raw data directory, if necessary.

```r
GetRawFile = function( url, output.file = rawfile )
{
    if ( !dir.exists( rawdata.dir ) )
    {
        dir.create( rawdata.dir, recursive = TRUE );
    }

    timestamp = "";
    timestamp.file = paste( output.file, "timestamp", sep = "." );

    if ( !file.exists( output.file ) )
    {
        message( "Downloading raw data." );
        timestamp = date();
        download.file(
            url = url,
            destfile = output.file
        );
        writeLines( timestamp, timestamp.file );
    }
    else
    {
        message( "Using existing raw data." );
        timestamp = readLines( timestamp.file );
    }

    suppressWarnings(
        unzip( output.file, overwrite = FALSE, exdir = rawdata.dir )
    );

    return(timestamp);
}
```

We can use the GetRawFile function to reproducibly fetch the raw data file.
In this case, this will FAIL to unzip the target file.

```r
cat(
    "Raw data downloaded on",
    GetRawFile("https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2"),
    "\n"
);
```

```
## Downloading raw data.
```

```
## Raw data downloaded on Sat Apr  9 15:57:35 2016
```

#### Tidying the data
Because of the size of this data set, here we conditionally tidy the data
dependant upon whether a clean data file, `TidyData.rds`, already exists,
indicating that the data processing routines in this analysis have already been
run and the result of that processing has been saved for future use. If it
does, then that file is used instead of the tidying operation. Note that the
existence of the `TidyData.rds` file should have *no* impact on the analysis,
except for shortening this data processing step. Also note that `TidyData.rds`
will be produced as a result of the data processing steps in this analysis.


```r
weather.table = NA;
dirtydata = TRUE;

if ( file.exists( tidyfile ) )
{
    message( "Reading saved tidy data set." );
    weather.table = readRDS( tidyfile );
    str( weather.table );
    dirtydata = FALSE;
}
```

If there was no tidy data file, then the raw data must be processed to be used
in the analysis. 

The raw data is compressed using bzip2, which, unfortunately, cannot at this
time be natively read by `fread`, the file reading component of `data.table`.
The following code decompresses the file, but it is **not** generally portable.
It requires that bunzip2 be installed, and may require that the system be
unix-like. The alternative is to use R connections, but those will be much,
much slower than the file reading mechanism provided by `fread`.

Also, to avoid decompressing the data file every time, we first check to see
if the uncompressed data file already exists.


```r
message( "Creating tidy data set." );
```

```
## Creating tidy data set.
```

```r
rawfile.decompressed = paste( rawfile, ".out", sep = "" );

if ( !file.exists( rawfile.decompressed ) )
{
    system( paste( "bunzip2 -k", rawfile  ) );
}
```

When we read in the data, only the beginning date for the event, event type,
fatalities, injuries, property damage, property damage explanation, crop
damage, and crop damage explanation are kept, as they will be the only
variables contributing to the analysis.

Note that reading the decompressed data will generate warnings. This appears to
be due to certain columns containing multiline values. `fread` is capable of
handling these multiline values, but the mechanism it uses for estimating the
number of observations in the file does not appear to take these multiline
values into account. This results in an inflated estimate that does not match
the actual number of lines read.


```r
weather.table =
    fread(
        rawfile.decompressed,
        select = c(
            "BGN_DATE",
            "EVTYPE",
            "FATALITIES",
            "INJURIES",
            "PROPDMG",
            "PROPDMGEXP",
            "CROPDMG",
            "CROPDMGEXP"
        ),
        na.strings = "",
        showProgress = FALSE
    );
```

```
## Warning in fread(rawfile.decompressed, select = c("BGN_DATE", "EVTYPE", :
## Read less rows (902297) than were allocated (967216). Run again with
## verbose=TRUE and please report.
```

```r
str(weather.table);
```

```
## Classes 'data.table' and 'data.frame':	902297 obs. of  8 variables:
##  $ BGN_DATE  : chr  "4/18/1950 0:00:00" "4/18/1950 0:00:00" "2/20/1951 0:00:00" "6/8/1951 0:00:00" ...
##  $ EVTYPE    : chr  "TORNADO" "TORNADO" "TORNADO" "TORNADO" ...
##  $ FATALITIES: num  0 0 0 0 0 0 0 0 1 0 ...
##  $ INJURIES  : num  15 0 2 2 2 6 1 0 14 0 ...
##  $ PROPDMG   : num  25 2.5 25 2.5 2.5 2.5 2.5 2.5 25 25 ...
##  $ PROPDMGEXP: chr  "K" "K" "K" "K" ...
##  $ CROPDMG   : num  0 0 0 0 0 0 0 0 0 0 ...
##  $ CROPDMGEXP: chr  NA NA NA NA ...
##  - attr(*, ".internal.selfref")=<externalptr>
```

As part of the data processing, we do some basic maintenance on the raw data
that was just read.

We change column names to lowercase (for ease of use), and we convert
`bgn_date` to a Date variable.


```r
names( weather.table ) = tolower( names( weather.table ) );
names( weather.table );
```

```
## [1] "bgn_date"   "evtype"     "fatalities" "injuries"   "propdmg"   
## [6] "propdmgexp" "cropdmg"    "cropdmgexp"
```

```r
weather.table[ , bgn_date := as.Date(bgn_date, "%m/%d/%Y %H:%M:%S") ];
```

We also note, from the data documentation, that the only correct values for the
`propdmgexp` and `cropdmgexp` columns are "K", "M", and "B", denoting
"thousands", "millions" and "billions" of dollars, respectively. If we take a
look at the actual values `propdmgexp` and `cropdmgexp`, though, we see that
there are a large number of other values in observations for this variable,
including lowercase "k" and "m".


```r
table( weather.table$propdmgexp );
```

```
## 
##      -      ?      +      0      1      2      3      4      5      6 
##      1      8      5    216     25     13      4      4     28      4 
##      7      8      B      h      H      K      m      M 
##      5      1     40      1      6 424665      7  11330
```

```r
table( weather.table$cropdmgexp );
```

```
## 
##      ?      0      2      B      k      K      m      M 
##      7     19      1      9     21 281832      1   1994
```

The lowercase values appear to be simple coding errors that can be fixed. The
other values, however, cannot be fixed by a simple coding change. Similarly,
if the explanation for damage is `NA`, then the damage amounts become
ambiguous. However, there are only 342 observations in this data set where
there is a problematic damage explanation value *and* the damage is not equal
to zero. Furthermore, these problematic values only occur in the years 1993,
1994, and 1995.


```r
weirdpropl = (
    !(weather.table$propdmgexp %in% c( "K","M","B","k","m","b" ))
    & weather.table$propdmg != 0
);

weirdprop = which( weirdpropl );

length(weirdprop);
```

```
## [1] 327
```

```r
table( weather.table[ weirdprop, propdmgexp ] );
```

```
## 
##   -   +   0   2   3   4   5   6   7   h   H 
##   1   5 209   1   1   4  18   3   2   1   6
```

```r
table( weather.table[ weirdprop, format(bgn_date, "%Y") ] );
```

```
## 
## 1993 1994 1995 
##    7   32  288
```

```r
weirdcropl = (
    !(weather.table$cropdmgexp %in% c( "K","M","B","k","m","b" ))
    & weather.table$cropdmg != 0
);

weirdcrop = which( weirdcropl );
length(weirdcrop);
```

```
## [1] 15
```

```r
table( weather.table[ weirdcrop, cropdmgexp ] );
```

```
## 
##  0 
## 12
```

```r
table( weather.table[ weirdcrop, format(bgn_date, "%Y") ] );
```

```
## 
## 1994 1995 
##   11    4
```

Because the number of problematic observations is much smaller than the total
number of observations, and because they occur in years that are less relevant
to later analytical results, we choose to exclude them. We also clean up the
remaining values in those variables, and turn them into factors.


```r
weather.table[ , propdmgexp := toupper( propdmgexp ) ];
weather.table[ , cropdmgexp := toupper( cropdmgexp ) ];

notweirdl = !( weirdcropl | weirdpropl );

weather.table = weather.table[ notweirdl, ];

weather.table[
    !( propdmgexp %in% c( "K","M","B" ) ),
    propdmgexp := NA
];
weather.table[
    !( cropdmgexp %in% c( "K","M","B" ) ),
    cropdmgexp := NA
];

weather.table[ , propdmgexp := factor( propdmgexp ) ];
weather.table[ , cropdmgexp := factor( cropdmgexp ) ];

str(weather.table);
```

```
## Classes 'data.table' and 'data.frame':	901955 obs. of  8 variables:
##  $ bgn_date  : Date, format: "1950-04-18" "1950-04-18" ...
##  $ evtype    : chr  "TORNADO" "TORNADO" "TORNADO" "TORNADO" ...
##  $ fatalities: num  0 0 0 0 0 0 0 0 1 0 ...
##  $ injuries  : num  15 0 2 2 2 6 1 0 14 0 ...
##  $ propdmg   : num  25 2.5 25 2.5 2.5 2.5 2.5 2.5 25 25 ...
##  $ propdmgexp: Factor w/ 3 levels "B","K","M": 2 2 2 2 2 2 2 2 2 2 ...
##  $ cropdmg   : num  0 0 0 0 0 0 0 0 0 0 ...
##  $ cropdmgexp: Factor w/ 3 levels "B","K","M": NA NA NA NA NA NA NA NA NA NA ...
##  - attr(*, ".internal.selfref")=<externalptr> 
##  - attr(*, "index")= int
```

We would also like to turn the event type variable into a factor.
Unfortunately, there are many problems with this variable. For instance, there
are over 900 different values in this column, but the data documentation only
specifies 48 possible valid values for the even type. 


```r
length( levels( factor( weather.table$evtype ) ) );
```

```
## [1] 982
```

There is some ambiguity in the documentation. "Hurricane/Typhoon" is also
referenced as "Hurricane (Typhoon)", and "Marine Tstm Wind" is also referenced
as "Marine Thunderstorm Wind". In addition, for all "Hail", "Wind" and
"Tornado" events, the data documentation specifies that additional
parenthetical values be included in the event type specifying details about
those events. It is interesting to note that the rules regarding those
parenthetical values are not followed for any observation in the NOAA data set.

Before we do any more intensive cleanup, we should remove observations that
contain no data, for example any row with with an event type containing the
word "summary", which aggregates other discrete event types, or "rogue wave",
which is an event that does not have a type and which, presumably, this
database does not officially track. We also change the case of each event type
to facilitate future processing here as well.


```r
weather.table[ , evtype := tolower( evtype ) ];

#   Summaries appear to be completely impossible to use.
weather.table = weather.table[ !str_detect( evtype, "summary" ), ];

#   This one has no matching evtype code, even after examining the description
#   of the observation to which it applies.
weather.table = weather.table[ !str_detect( evtype, "marine accident" ), ];

#   This one had reported large hail and strong winds, which alone might cause
#   me to exclude it due to it being in two categories, but it also had no
#   apparent fatalities, injuries, property or crop damage. It appears to be a
#   bad entry.
weather.table = weather.table[ !str_detect( evtype, "metro storm" ), ];

#   Entries with the word "monthly" in them are summaries, and cannot be
#   considered.
weather.table = weather.table[ !str_detect( evtype, "monthly" ), ];

#   Yet another summary entry that cannot be considered. Also, why record a
#   lack of snow as an event?
weather.table = weather.table[ !str_detect( evtype, "lack of snow" ), ];

#   Why is this next one even in the database?
weather.table = weather.table[ !str_detect( evtype, "no severe weather" ), ];

#   What? Why?!?
weather.table = weather.table[ evtype != "none", ];

#   I can't even . . . >.<
weather.table = weather.table[ !str_detect( evtype, "northern lights" ), ];

#   The following "record" entries do not appear to match any valid event type.
#   There may be a few that do, but the problem is that they might be
#   duplicated by valid events, and there are too many to check each one
#   individually in a reasonable amount of time. All "record" events have to
#   go.
weather.table = weather.table[ !str_detect( evtype, "record" ), ];

#   These appear to be reporting the chance of wildfires, or wildfires that are
#   already recorded by other observations.
weather.table = weather.table[ !str_detect( evtype, "red flag" ), ];

#   These have no matching valid event type.
weather.table = weather.table[ !str_detect( evtype, "rogue wave" ), ];
weather.table = weather.table[ !str_detect( evtype, "heavy seas" ), ];
weather.table = weather.table[ !str_detect( evtype, "heavy swells" ), ];
weather.table = weather.table[ !str_detect( evtype, "high\\s*swells" ), ];
weather.table = weather.table[ !str_detect( evtype, "high seas" ), ];
weather.table = weather.table[ !str_detect( evtype, "rough seas" ), ];
weather.table = weather.table[ !str_detect( evtype, "severe turbulence" ), ];

#   The word "unseasonable" is not specific enough to denote an actual event
#   type, and there are too many such entries to check each one individually.
weather.table = weather.table[ !str_detect( evtype, "unseason" ), ];

#   Similarly, "unusual" elements are not specific enough, either.
weather.table = weather.table[ !str_detect( evtype, "unusual" ), ];

#   As appears to be the case with many of the adverbs and adjectives appearing
#   in the event type column, "abnormal" observations appear to contain summary
#   information that is not usable.
weather.table = weather.table[ !str_detect( evtype, "abnormal" ), ];

#   The various "accumulated" entries not only are not properly coded as event
#   types, but they report damage in their remarks without having any damage
#   specified in the proper variables.
weather.table = weather.table[ !str_detect( evtype, "accumulate" ), ];

#   No really information in the entry with this one.
weather.table = weather.table[ !str_detect( evtype, "driest" ), ];

#   The word pattern seems to indicate summary information as well.
weather.table = weather.table[ !str_detect( evtype, "pattern" ), ];

#   Same with "spell".
weather.table = weather.table[ !str_detect( evtype, "spell" ), ];

#   There are a few observations with the "other" even type. These end up being
#   mixtures of dust devils, winds, and winter weather. It may be possible to
#   recode these all individually, but it is not worth the time investment to
#   do it for this exploratory analysis, so they have to go.
weather.table = weather.table[ evtype !=  "other", ];

#   This next one was coded incorrectly and no detail in the remarks to help.
weather.table = weather.table[ !str_detect( evtype, "beach erosin" ), ];

#   This next invalid event type descriptor denotes summaries, like most others
#   do.
weather.table = weather.table[ !str_detect( evtype, "normal" ), ];

#   It is unfortunately impossible to determine if "dry" events are droughts
#   from the information available. The website that is supposed to maintain
#   those definitions is no longer available, and there isn't enough
#   information in the remarks to make a determination anyway (probably).
weather.table = weather.table[ ! (evtype %in% c( "dry", 
                                                 "dry conditions",
                                                 "dry weather",
                                                 "dryness",
                                                 "excessively dry",
                                                 "very dry"
                                               )
                                 ),
                             ];

#   "Early rain" cannot be matched to a valid event type.
weather.table = weather.table[ evtype != "early rain", ];

#   "Excessive precipitation" also cannot be matched to a valid event type.
weather.table = weather.table[ evtype != "excessive precipitation", ];

#   "extremely wet" is a summary observation of no value to us.
weather.table = weather.table[ evtype != "extremely wet", ];

#   "flood watch" appears to be a summary event.
weather.table = weather.table[ !str_detect( evtype, "flood watch" ), ];

#   No idea what this is supposed to be.
weather.table = weather.table[ !( evtype %in% c( "high",
                                                 "large wall cloud",
                                                 "wall cloud",
                                                 "rotating wall cloud"
                                               )
                                ),
                             ];


#   More summary entries.
weather.table = weather.table[ !( evtype %in% c( "wet month", "wet year" ) ) ];

#   Get rid of the obvious one, finally.
weather.table = weather.table[ evtype != "?" ];
```

We are not interested in the size of the hail or the strength of tornados and
winds, and no digits appear in any of the valid entries for `evtype` in the
data set, so we will first remove all digits and parentheticals, We also are
not interested in most dashes, so we will remove them, and we can change all
instances of blackslashes or double-backslashes to forward slashes.


```r
#   I really wish we had perl regular expression syntax here. Would simplify
#   the horrible \\\\\\\\ (which matches \\ in the actual string) and \\\\.
weather.table[ , evtype := str_replace_all( evtype, "\\\\\\\\|\\\\", "/" ) ];
weather.table[ ,
    evtype := str_replace_all(
        evtype,
        "\\(|[0-9.]+|\\)|/$|^/",
        ""
    )
];
weather.table[ , evtype := str_replace_all( evtype, "-|\\s+", " " ) ];
```

There are a number of simple text replacements that can be performed that
should be safe. These arise from typos, inconsistent text formatting, and
left-over characters from previous processing steps. Correcting these should
have no negative impact on our results, and will provide additional
observations that might otherwise be lost.


```r
weather.table[ , evtype := str_replace_all( evtype, "wildfires", "wildfire" ) ];
weather.table[ , evtype := str_replace_all( evtype, "wild fires", "wildfire" ) ];
weather.table[ , evtype := str_replace_all( evtype, "water spout", "waterspout" ) ];
weather.table[ , evtype := str_replace_all( evtype, "waterspouts", "waterspout" ) ];
weather.table[ , evtype := str_replace_all( evtype, "wayterspout", "waterspout" ) ];
weather.table[ , evtype := str_replace_all( evtype, "ashfall", "ash" ) ];
weather.table[ , evtype := str_replace_all( evtype, "ash plume", "ash" ) ];
weather.table[ , evtype := str_replace_all( evtype, "tornadoes", "tornado" ) ];
weather.table[ , evtype := str_replace_all( evtype, "tornadoe debris", "tornado" ) ];
weather.table[ , evtype := str_replace_all( evtype, "tornados", "tornado" ) ];
weather.table[ , evtype := str_replace_all( evtype, "torndao", "tornado" ) ];
weather.table[ , evtype := str_replace_all( evtype, "thuderstorm", "thunderstorm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "thundeerstorm", "thunderstorm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "thunderestorm", "thunderstorm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "thunderstorms", "thunderstorm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "thundertorm", "thunderstorm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "thundertsorm", "thunderstorm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "thundestorm", "thunderstorm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "thunerstorm", "thunderstorm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "tunderstorm", "thunderstorm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "thunderstormwind", "thunderstorm wind" ) ];
weather.table[ , evtype := str_replace_all( evtype, "currents", "current" ) ];
weather.table[ , evtype := str_replace_all( evtype, "snowfall", "snow" ) ];
weather.table[ , evtype := str_replace_all( evtype, "lighting", "lightning" ) ];
weather.table[ , evtype := str_replace_all( evtype, "ligntning", "lightning" ) ];
weather.table[ , evtype := str_replace_all( evtype, "rainfall", "rain" ) ];
weather.table[ , evtype := str_replace_all( evtype, "rains", "rain" ) ];
weather.table[ , evtype := str_replace_all( evtype, "hvy", "heavy" ) ];
weather.table[ , evtype := str_replace_all( evtype, "torrential", "heavy" ) ];
weather.table[ , evtype := str_replace_all( evtype, "showers*", "rain" ) ];
weather.table[ , evtype := str_replace_all( evtype, "cstl", "coastal" ) ];
weather.table[ , evtype := str_replace_all( evtype, "devel", "devil" ) ];
weather.table[ , evtype := str_replace_all( evtype, "flashflood", "flash flood" ) ];
weather.table[ , evtype := str_replace_all( evtype, "flooding", "flood" ) ];
weather.table[ , evtype := str_replace_all( evtype, "floooding", "flood" ) ];
weather.table[ , evtype := str_replace_all( evtype, "floodin", "flood" ) ];
weather.table[ , evtype := str_replace_all( evtype, "flooodin", "flood" ) ];
weather.table[ , evtype := str_replace_all( evtype, "floods", "flood" ) ];
weather.table[ , evtype := str_replace_all( evtype, "clouds", "cloud" ) ];
weather.table[ , evtype := str_replace_all( evtype, "wnd", "wind" ) ];
weather.table[ , evtype := str_replace_all( evtype, "wins", "wind" ) ];
weather.table[ , evtype := str_replace_all( evtype, "w inds", "wind" ) ];
weather.table[ , evtype := str_replace_all( evtype, "w wind", " wind" ) ];
weather.table[ , evtype := str_replace_all( evtype, "winds", "wind" ) ];
weather.table[ , evtype := str_replace_all( evtype, "avalance", "avalanche" ) ];
weather.table[ , evtype := str_replace_all( evtype, "tstm", "thunderstorm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "coastalflood", "coastal flood" ) ];
weather.table[ , evtype := str_replace_all( evtype, "duststorm", "dust storm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "icestorm", "ice storm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "funnel cloudss", "funnel clouds" ) ];
weather.table[ , evtype := str_replace_all( evtype, "hail\\s*storms*", "hail" ) ];
weather.table[ , evtype := str_replace_all( evtype, " g$", "" ) ];
weather.table[ , evtype := str_replace_all( evtype, " f$", "" ) ];
weather.table[ , evtype := str_replace_all( evtype, " le cen$", "" ) ];
weather.table[ , evtype := str_replace_all( evtype, " mph$", "" ) ];
weather.table[ , evtype := str_replace_all( evtype, "small hail", "hail" ) ];
weather.table[ , evtype := str_replace_all( evtype, "^typhoon$", "hurricane/typhoon" ) ];
```

These next replacements are, perhaps, slightly more dangerous, but should be
relatively safe. These transformations involve removing or changing irrelevant
information in the event type string to help match valid event type values.


```r
weather.table[ , evtype := str_replace_all( evtype, "thunderstormw", "thunderstorm wind" ) ];
weather.table[ , evtype := str_replace_all( evtype, "^hurricane.*", "hurricane/typhoon" ) ];
weather.table[ , evtype := str_replace_all( evtype, "^tropical storm.*", "tropical storm" ) ];
weather.table[ , evtype := str_replace_all( evtype, "locally|local", "" ) ];
weather.table[ , evtype := str_replace_all( evtype, "major|minor", "" ) ];
weather.table[ , evtype := str_replace_all( evtype, "/*\\s*trees", "" ) ];
weather.table[ , evtype := str_replace_all( evtype, "late season ", "" ) ];
weather.table[ , evtype := str_replace_all( evtype, "rural", "" ) ];
weather.table[ , evtype := str_replace_all( evtype, "agricultural ", "" ) ];
weather.table[ , evtype := str_replace_all( evtype, "near record", "heavy" ) ];
```

Next, we do some safe cleanup to remove extra spaces, in preparation for the
massive categorization to come.


```r
weather.table[ , evtype := str_replace_all( evtype, "\\s+", " " ) ];
weather.table[ , evtype := str_replace_all( evtype, "^\\s+|\\s+$", "" ) ];
```

Finally, this next series uses the definitions provided in the data
documentation to attempt to classify various remaining errant `evtype` values
as a valid values.  Many of the event type values can be reclassified based on
those guidlines, but occasionally interpretation beyond those guidelines is
required. Any `evtype` values that could not possibly be classified have
already been removed from the dataset. The categorization here is not perfect,
but is reasonable for an exploratory study.


```r
weather.table[
    evtype %in% c( "blow out tide", "blow out tides" ),
    evtype := "astronomical low tide"
];
weather.table[
    evtype == "heavy snow/blizzard/avalanche",
    evtype := "avalanche"
];
weather.table[
    evtype %in% c(
        "blizzard and extreme wind chil", "blizzard and heavy snow",
        "blizzard weather", "blizzard/freezing rain", "blizzard/heavy snow",
        "blizzard/high wind", "blizzard/winter storm", "ground blizzard",
        "heavy snow/blizzard", "high wind/ blizzard", "high wind/blizzard",
        "high wind/blizzard/freezing ra", "high wind/wind chill/blizzard",
        "ice storm/blizzard"
    ),
    evtype := "blizzard"
];
weather.table[
    evtype %in% c(
        "beach erosion/coastal flood", "beach flood", "coastal flood/erosion",
        "coastal/tidal flood", "erosion/coastal flood",
        "heavy surf coastal flood", "high wind/coastal flood"
    ),
    evtype := "coastal flood"
];
weather.table[
    evtype %in% c(
        "cold", "cold temperature", "cold temperatures", "cold wave",
        "cold weather", "cold wind chill temperatures", "cold/wind",
        "excessive cold", "extended cold", "fog and cold temperatures",
        "high wind and wind chill", "high wind/cold",
        "high wind/lo wind chill", "high wind/wind chill",
        "hyperthermia/exposure", "hypothermia", "hypothermia/exposure",
        "lo wind chill", "low temperature", "prolong cold", "wind chill",
        "wind chill/high wind"
    ),
    evtype := "cold/wind chill"
];
weather.table[
    evtype %in% c(
        "landslide", "landslide/urban flood", "landslides", "landslump",
        "mud slide", "mud slides", "mud slides urban flood", "mud/rock slide",
        "mudslide", "mudslide/landslide", "mudslides", "rock slide",
        "urban flood landslide"
    ),
    evtype := "debris flow"
];
weather.table[
    evtype %in% c( "fog", "patchy dense fog" ),
    evtype := "dense fog"
];
weather.table[
    evtype == "smoke",
    evtype := "dense smoke"
];
weather.table[
    evtype %in% c(
        "drought/excessive heat", "excessive", "excessive heat/drought",
        "heat drought", "heat wave drought", "heat/drought", "snow drought"
    ),
    evtype := "drought"
];
weather.table[
    evtype %in% c(
        "blowing dust", "dust storm/high wind", "high wind dust storm",
        "saharan dust"
    ),
    evtype := "dust storm"
];
weather.table[
    evtype == "extreme heat",
    evtype := "excessive heat"
];
weather.table[
    evtype %in% c(
        "bitter wind chill", "bitter wind chill temperatures", "extreme cold",
        "extreme wind chill", "extreme wind chills", "extreme windchill",
        "extreme windchill temperatures", "severe cold"
    ),
    evtype := "extreme cold/wind chill"
];
weather.table[
    evtype %in% c(
        "flash flood from ice jams", "flash flood heavy rain",
        "flash flood landslides", "flash flood/ flood", "flash flood/ street",
        "flash flood/flood", "flash flood/heavy rain", "flash flood/landslide",
        "flash flood/thunderstorm wi", "flood flash", "flood flood/flash",
        "flood/flash", "flood/flash flood", "flood/flash/flood",
        "rapidly rising water", "thunderstorm wind/flash flood"
    ),
    evtype := "flash flood"
];
weather.table[
    evtype %in% c(
        "flood & heavy rain", "flood/rain/wind", "flood/river flood",
        "flood/strong wind", "hail flood", "heavy rain and flood",
        "heavy rain/flood", "heavy rain/mudslides/flood",
        "heavy snow/high wind & flood", "high wind/flood", "ice jam flood",
        "river and stream flood", "river flood", "snowmelt flood",
        "thunderstorm wind/ flood", "thunderstorm wind/flood"
    ),
    evtype := "flood"
];
weather.table[
    evtype == "ice fog",
    evtype := "freezing fog"
];
weather.table[
    evtype %in% c(
        "cold and frost", "damaging freeze", "early freeze", "early frost",
        "first frost", "freeze", "frost", "hard freeze", "late freeze"
    ),
    evtype := "frost/freeze"
];
weather.table[
    evtype %in% c(
        "cold air funnel", "cold air funnels", "funnel", "funnels",
        "thunderstorm wind funnel clou", "thunderstorm wind/funnel clou",
        "wall cloud/funnel cloud"
    ),
    evtype := "funnel cloud"
];
weather.table[
    evtype %in% c(
        "deep hail", "funnel cloud/hail", "gusty wind/hail", "hail aloft",
        "hail damage", "hail/wind", "non severe hail", "thunderstorm hail",
        "thunderstorm wind hail", "thunderstorm wind/ hail",
        "thunderstorm wind/hail", "thunderstorm windhail", "wind/hail"
    ),
    evtype := "hail"
];
weather.table[
    evtype %in% c(
        "dry hot weather", "heat wave", "heat waves", "hot and dry",
        "hot weather", "prolong warmth", "very warm", "warm dry conditions",
        "warm weather"
    ),
    evtype := "heat"
];
weather.table[
    evtype %in% c(
        "coastal storm", "coastalstorm", "cold and wet conditions",
        "cool and wet", "dam break", "dam failure", "drowning",
        "excessive rain", "excessive wetness", "gusty wind/heavy rain",
        "heavy precipatation", "heavy precipitation", "heavy rain and wind",
        "heavy rain effects", "heavy rain; urban flood wind;",
        "heavy rain/lightning", "heavy rain/severe weather",
        "heavy rain/small stream urban", "heavy rain/urban flood",
        "heavy rain/wind", "high wind heavy rain", "high wind/heavy rain",
        "highway flood", "lightning and heavy rain", "lightning/heavy rain",
        "prolonged rain", "rain", "rain and wind", "rain damage", "rain heavy",
        "rain/wind", "raintorm", "small stream", "small stream and",
        "small stream and urban flood", "small stream flood",
        "small stream urban flood", "small stream/urban flood",
        "sml stream fld", "stream flood", "street flood",
        "thunderstorm heavy rain", "thunderstorm wind heavy rain",
        "thunderstorm wind small strea", "thunderstorm wind urban flood",
        "thunderstorm wind/heavy rain", "urban and small",
        "urban and small stream", "urban and small stream flood",
        "urban flood", "urban small", "urban small stream flood",
        "urban/small", "urban/small flood", "urban/small stream",
        "urban/small stream flood", "urban/small strm fldg",
        "urban/sml stream fld", "urban/sml stream fldg", "urban/street flood",
        "wet weather"
    ),
    evtype := "heavy rain"
];
weather.table[
    evtype %in% c(
        "excessive snow", "falling snow/ice", "heavy snow and",
        "heavy snow and high wind", "heavy snow and strong wind",
        "heavy snow squalls", "heavy snow/high", "heavy snow/high wind",
        "heavy snow/squalls", "heavy snow/wind", "heavy snowpack",
        "heavy wet snow", "high wind and heavy snow", "high wind/heavy snow",
        "snow accumulation", "snow and heavy snow", "snow/heavy snow"
    ),
    evtype := "heavy snow"
];
weather.table[
    evtype %in% c(
        "astronomical high tide", "beach erosion", "hazardous surf",
        "heavy rain/high surf", "heavy surf", "heavy surf and wind",
        "heavy surf/high surf", "high surf advisories", "high surf advisory",
        "high tides", "high water", "high waves", "high wind and high tides",
        "rip current heavy surf", "rip current/heavy surf", "rough surf",
        "wind and wave"
    ),
    evtype := "high surf"
];
weather.table[
    evtype %in% c(
        "gradient wind", "high wind damage", "high wind/seas",
        "non thunderstorm wind", "storm force wind"
    ),
    evtype := "high wind"
];
weather.table[
    evtype == "remnants of floyd",
    evtype := "hurricane/typhoon"
];
weather.table[
    evtype %in% c(
        "glaze/ice storm", "heavy snow/ice storm", "ice storm and snow",
        "ice storm/flash flood", "snow and ice storm", "snow/ice storm"
    ),
    evtype := "ice storm"
];
weather.table[
    evtype %in% c(
        "heavy lake snow", "lake effect snow", "thundersnow",
        "thundersnow rain"
    ),
    evtype := "lake-effect snow"
];
weather.table[
    evtype == "lake flood",
    evtype := "lakeshore flood"
];
weather.table[
    evtype %in% c(
        "lightning damage", "lightning fire", "lightning injury",
        "lightning wauseon"
    ),
    evtype := "lightning"
];
weather.table[
    evtype == "high wind and seas",
    evtype := "marine high wind"
];
weather.table[
    evtype == "gusty lake wind",
    evtype := "marine strong wind"
];
weather.table[
    evtype == "marine mishap",
    evtype := "marine strong wind"
];
weather.table[
    evtype == "sleet storm",
    evtype := "sleet"
];
weather.table[
    evtype %in% c(
        "coastal erosion", "coastal surge", "storm surge", "tidal flood"
    ),
    evtype := "storm surge/tide"
];
weather.table[
    evtype %in% c(
        "gusty wind", "gusty wind/rain", "non severe wind damage", "southeast",
        "strong wind gust", "wake lo wind", "wind", "wind advisory",
        "wind damage", "wind gusts", "wind storm"
    ),
    evtype := "strong wind"
];
weather.table[
    evtype %in% c(
        "apache county", "downburst", "downburst wind", "dry microburst",
        "dry microburst wind", "dry mircoburst wind", "gustnado",
        "gustnado and", "gusty thunderstorm wind", "heatburst",
        "lightning and thunderstorm win", "lightning and wind",
        "lightning thunderstorm wind", "lightning thunderstorm winds",
        "microburst", "microburst wind", "severe thunderstorm",
        "severe thunderstorm wind", "thunderstorm", "thunderstorm damage",
        "thunderstorm damage to", "thunderstorm wind and",
        "thunderstorm wind and lightning", "thunderstorm wind damage",
        "thunderstorm wind lightning", "thunderstorm wind/ tree",
        "thunderstorm wind/awning", "thunderstorm wind/lightning",
        "thunderstorm winds", "thunderstrom wind", "wet micoburst",
        "wet microburst"
    ),
    evtype := "thunderstorm wind"
];
weather.table[
    evtype %in% c(
        "cold air tornado", "landspout", "tornado debris",
        "tornado, thunderstorm wind, hail", "whirlwind"
    ),
    evtype := "tornado"
];
weather.table[
    evtype %in% c( "vog", "volcanic eruption" ),
    evtype := "volcanic ash"
];
weather.table[
    evtype %in% c(
        "dust devil waterspout", "tornado/waterspout",
        "waterspout funnel cloud", "waterspout tornado", "waterspout/ tornado",
        "waterspout/tornado"
    ),
    evtype := "waterspout"
];
weather.table[
    evtype %in% c(
        "brush fire", "brush fires", "forest fires", "grass fires",
        "wild/forest fire", "wild/forest fires"
    ),
    evtype := "wildfire"
];
weather.table[
    evtype %in% c(
        "freezing rain and sleet", "freezing rain and snow",
        "freezing rain sleet and", "freezing rain sleet and light",
        "freezing rain/sleet", "freezing rain/snow", "hail/icy roads",
        "heavy mix", "heavy snow & ice", "heavy snow and ice",
        "heavy snow and ice storm", "heavy snow andblowing snow",
        "heavy snow freezing rain", "heavy snow/blowing snow",
        "heavy snow/freezing rain", "heavy snow/high wind/freezing",
        "heavy snow/ice", "heavy snow/sleet", "heavy snow/winter storm",
        "ice and snow", "ice floes", "ice/snow", "light snow and sleet",
        "light snow/freezing precip", "mixed precip", "mixed precipitation",
        "sleet & freezing rain", "sleet/freezing rain", "sleet/ice storm",
        "sleet/rain/snow", "sleet/snow", "snow and ice", "snow and sleet",
        "snow freezing rain", "snow sleet", "snow/ ice", "snow/freezing rain",
        "snow/ice", "snow/rain/sleet", "snow/sleet",
        "snow/sleet/freezing rain", "snow/sleet/rain", "winter mix",
        "winter storm high wind", "winter storm/high wind", "winter storms",
        "winter weather mix", "winter weather/mix", "wintery mix", "wintry mix"
    ),
    evtype := "winter storm"
];
weather.table[
    evtype %in% c(
        "black ice", "blowing snow", "blowing snow & extreme wind ch",
        "blowing snow extreme wind chi", "blowing snow/extreme wind chil",
        "cold and snow", "drifting snow", "early snow",
        "extreme wind chill/blowing sno", "first snow", "freezing drizzle",
        "freezing drizzle and freezing", "freezing rain", "freezing spray",
        "glaze", "glaze ice", "heavy rain/snow", "heavy snow rain",
        "high wind/snow", "ice", "ice jam", "ice on road", "ice pellets",
        "ice roads", "ice/strong wind", "icy roads", "late snow",
        "light freezing rain", "light snow", "light snow/flurries",
        "moderate snow", "mountain snows", "patchy ice", "prolong cold/snow",
        "rain/snow", "seasonal snow", "snow", "snow advisory", "snow and cold",
        "snow and wind", "snow high wind wind chill", "snow rain",
        "snow squall", "snow squalls", "snow/ bitter cold",
        "snow/blowing snow", "snow/cold", "snow/high wind", "snow/rain",
        "snowstorm", "wet snow"
    ),
    evtype := "winter weather"
];

valid.evtype = c(
    "Astronomical Low Tide", "Avalanche", "Blizzard",
    "Coastal Flood", "Cold/Wind Chill", "Debris Flow",
    "Dense Fog", "Dense Smoke", "Drought",
    "Dust Devil", "Dust Storm", "Excessive Heat",
    "Extreme Cold/Wind Chill", "Flash Flood", "Flood",
    "Frost/Freeze", "Funnel Cloud", "Freezing Fog",
    "Hail", "Heat", "Heavy Rain",
    "Heavy Snow", "High Surf", "High Wind",
    "Hurricane/Typhoon", "Ice Storm", "Lake-Effect Snow",
    "Lakeshore Flood", "Lightning", "Marine Hail",
    "Marine High Wind", "Marine Strong Wind", "Marine Thunderstorm Wind",
    "Rip Current", "Seiche", "Sleet",
    "Storm Surge/Tide", "Strong Wind", "Thunderstorm Wind",
    "Tornado", "Tropical Depression", "Tropical Storm",
    "Tsunami", "Volcanic Ash", "Waterspout",
    "Wildfire", "Winter Storm", "Winter Weather"
);

weather.table = weather.table[ evtype %in% tolower(valid.evtype), ];
weather.table$evtype = factor( weather.table$evtype );

#   This is from the ?tolower help information:
proper = function(ss)
{
    cap =
        function(st) paste(
            toupper( substring( st, 1, 1 ) ),
            tolower( substring( st, 2 ) ),
            sep = "",
            collapse = " "
        );

    sapply(
        strsplit( ss, split = " " ),
        cap,
        USE.NAMES = !is.null( names( ss ) )
    );
}

#   Capitalization for nice summarization later.
levels( weather.table$evtype ) =
    proper( levels( weather.table$evtype ) );
```

Now we check for `NA` values. Generally, `NA` values will prevent an
observation from contributing to the analysis. `NA` values in `propdmgexp` and
`cropdmgexp` are valid only if the accompanying `propdmg` and `cropdmg` are 0.
We remove any observations that do not conform.


```r
weather.table = weather.table[ 
    !is.na(bgn_date) &
    !is.na(evtype) &
    !is.na(fatalities) &
    !is.na(injuries) &
    !is.na(propdmg) &
    !is.na(cropdmg) &
    !(is.na(propdmgexp) & propdmg != 0) &
    !(is.na(cropdmgexp) & cropdmg != 0),
];
```

Since `propdmg` and `propdmgexp` do not actually represent different variables,
but rather a single variable spread across two columns, we combine them back
into a single numerical value of the proper magnitude and store that back in
`propdmg`, dropping `propdmgexp` from the data set.  We do the same for
`cropdmg` and `cropdmgexp`. Finally, since we are only interested in overall
economic impact, we combine `cropdmg` and `propdmg` into a single new variable,
`totaldmg`, and drop `cropdmg` and `propdmg` from the data set.

```r
weather.table[
    !is.na(propdmgexp),
    propdmg :=
        ifelse(
            propdmgexp == 'K',
            propdmg * 1000,
            ifelse(
                propdmgexp == 'M',
                propdmg * 1000000,
                propdmg * 1000000000
            )
        )
];
weather.table[ , propdmgexp := NULL ];

weather.table[
    !is.na(cropdmgexp),
    cropdmg :=
        ifelse(
            cropdmgexp == 'K',
            cropdmg * 1000,
            ifelse(
                cropdmgexp == 'M',
                cropdmg * 1000000,
                cropdmg * 1000000000
            )
        )
];

weather.table[ , cropdmgexp := NULL ];

weather.table[ , totaldmg := propdmg + cropdmg ];
weather.table[ , cropdmg := NULL ];
weather.table[ , propdmg := NULL ];

str( weather.table );
```

```
## Classes 'data.table' and 'data.frame':	900917 obs. of  5 variables:
##  $ bgn_date  : Date, format: "1950-04-18" "1950-04-18" ...
##  $ evtype    : Factor w/ 48 levels "Astronomical Low Tide",..: 40 40 40 40 40 40 40 40 40 40 ...
##  $ fatalities: num  0 0 0 0 0 0 0 0 1 0 ...
##  $ injuries  : num  15 0 2 2 2 6 1 0 14 0 ...
##  $ totaldmg  : num  25000 2500 25000 2500 2500 2500 2500 2500 25000 25000 ...
##  - attr(*, ".internal.selfref")=<externalptr>
```

```r
if ( !dir.exists( tidydata.dir ) )
{
    dir.create( tidydata.dir, recursive = TRUE );
}
saveRDS( weather.table, tidyfile );
```

## Results

The present impact of weather events on the economy and individual health
depends largely on the fatilities, injuries, and damage done by those events.
From the data provided by the NOAA, we can study this information to get an
idea of which events are most damaging.  However, due to advances in
technology (such as the advent of early warning systems), changes in culture
(such as education), changes in the environment (such as global warming), and
changes in data collection techniques, not all data provided by NOAA will be
relevant to a measure of the current impact of weather events.

The following figure shows total yearly fatalities, injuries, and property
damage for each weather event. While specific details about individual weather
events cannot be determined from this graph, it is clear that there is
significant variation in the collected data over time.


```r
library(lubridate);
```

```
## 
## Attaching package: 'lubridate'
```

```
## The following objects are masked from 'package:data.table':
## 
##     hour, mday, month, quarter, wday, week, yday, year
```

```r
library(ggplot2);
weather.impact.by.year =
    weather.table[ , year := year(bgn_date) ];

weather.impact.by.year =
    weather.impact.by.year[ ,
        list(
            fatalities = sum(fatalities),
            injuries = sum(injuries),
            totaldmg = sum(totaldmg)
        ),
        by="year,evtype"
    ];

weather.impact.by.year =
    melt(
         weather.impact.by.year,
         c( "year", "evtype" ),
         variable.name = "impact"
    );

wiby =
    ggplot( weather.impact.by.year, aes( year, value ) ) +
    geom_line( aes( col = evtype ) ) +
    facet_grid( impact ~ ., scales = "free_y" ) +
    theme( legend.position = "bottom" ) +
    xlab( "Year" ) +
    ylab( "" ) +
    ggtitle( "Summary of Total Weather Impact Over Time" ) +
    scale_x_continuous(
        breaks =
            round(
                seq(
                    min(weather.impact.by.year$year),
                    max(weather.impact.by.year$year),
                    by = 5 
                ),
                0
            )
    ) +
    scale_color_discrete( name = "Event Title" );

print(wiby);
```

![](figure/weather_impact_by_year-1.png)

The plot shows that data collected earlier than 1993 is, at best, incomplete.
In addition, the drop in fatalities for the data that does exist prior to 1993
signals changes in at least one of the elements mentioned earlier. Also, from
the data processing step, we know there were significant problems with data
collected prior to 1996.

However, we can focus on data after 1996 to get the following two plots. The
first shows the average health impact of each weather event per year after
1996. From this plot, it appears as though tsunamis, hurricanes, and
heat/excessive heat have the largest impact on health.


```r
weather.impact =
    weather.table[ , year := year(bgn_date) ];

weather.impact =
    weather.impact[
        year >= 1996,
        list(
            injuries = mean(injuries),
            fatalities = mean(fatalities),
            totaldmg = mean(totaldmg)
        ),
        by = "evtype"
    ];

weather.impact =
    melt(
         weather.impact,
         c( "evtype", "totaldmg" ),
         variable.name = "health"
    );

healthplot =
    ggplot( weather.impact, aes( x = evtype, y = value, fill = health ) ) +
    geom_bar( stat = "identity", position = position_dodge() ) +
    theme(
        axis.text.x = element_text( angle = -60, hjust = 0 )
    ) +
    scale_y_continuous( expand = c(0, 0), limits = c(0, 6.8) ) +
    ggtitle(
        "Average Health Impact of Weather Effects Per Year (after 1996)"
    ) +
    xlab( "Weather Event Type" ) +
    ylab( "Number of People" ) +
    scale_fill_discrete( name = "" );

print( healthplot );
```

![](figure/weather_health_impact_after_1996-1.png)

Similarly, the following figure shows average economic impact per year after
1996 for each weather event. From this plot, it appears as though hurricanes
and storm surges have by far the largest economic impact.


```r
econplot =
    ggplot( weather.impact, aes( x = evtype, y = totaldmg, fill = evtype ) ) +
    geom_bar( stat = "identity", show.legend = FALSE ) +
    theme(
        axis.text.x = element_text( angle = -60, hjust = 0 )
    ) +
    scale_y_continuous( expand = c(0, 0) ) +
    ggtitle(
        "Average Economic Impact of Weather Effects Per Year (after 1996)"
    ) +
    xlab( "Weather Event Type" ) +
    ylab( "Damage (in dollars)" ) +
    scale_fill_discrete( name = "" );

print( econplot );
```

![](figure/weather_economic_impact_after_1996-1.png)

Future investigation should probably work to resolve issues with the NOAA data
set in a way that is robust. In addition, focus should be given to
examining the impact of hurricanes, heat, storm surge (which is related to
hurricanes), and tsunamis.

***
***
