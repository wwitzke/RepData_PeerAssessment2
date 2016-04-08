# Exploratory Analysis of Weather Events with High Mortality and Economic Impact
Wayne Witzke  

## Synopsis

**TODO**

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
## [1] data.table_1.9.6 rmarkdown_0.9.5 
## 
## loaded via a namespace (and not attached):
##  [1] magrittr_1.5    formatR_1.3     htmltools_0.3.5 tools_3.2.2    
##  [5] yaml_2.1.13     Rcpp_0.12.3     stringi_1.0-1   knitr_1.12.3   
##  [9] stringr_1.0.0   digest_0.6.9    chron_2.3-47    evaluate_0.8.3
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
## Using existing raw data.
```

```
## Raw data downloaded on Thu Apr  7 15:13:35 2016
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
values into account, resulting in an inflated estimate that does not match the
actual number of lines read.


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
Unfortunately, there are many problems with this variable. As an example, there
are over 900 different values in this column, but the data documentation only
has 48 possible values for the even type. 


```r
levels( factor( weather.table$evtype ) );
```

```
##   [1] "?"                              "ABNORMALLY DRY"                
##   [3] "ABNORMALLY WET"                 "ABNORMAL WARMTH"               
##   [5] "ACCUMULATED SNOWFALL"           "AGRICULTURAL FREEZE"           
##   [7] "APACHE COUNTY"                  "ASTRONOMICAL HIGH TIDE"        
##   [9] "ASTRONOMICAL LOW TIDE"          "AVALANCE"                      
##  [11] "AVALANCHE"                      "BEACH EROSIN"                  
##  [13] "Beach Erosion"                  "BEACH EROSION"                 
##  [15] "BEACH EROSION/COASTAL FLOOD"    "BEACH FLOOD"                   
##  [17] "BELOW NORMAL PRECIPITATION"     "BITTER WIND CHILL"             
##  [19] "BITTER WIND CHILL TEMPERATURES" "Black Ice"                     
##  [21] "BLACK ICE"                      "BLIZZARD"                      
##  [23] "BLIZZARD AND EXTREME WIND CHIL" "BLIZZARD AND HEAVY SNOW"       
##  [25] "BLIZZARD/FREEZING RAIN"         "BLIZZARD/HEAVY SNOW"           
##  [27] "BLIZZARD/HIGH WIND"             "Blizzard Summary"              
##  [29] "BLIZZARD WEATHER"               "BLIZZARD/WINTER STORM"         
##  [31] "BLOWING DUST"                   "blowing snow"                  
##  [33] "Blowing Snow"                   "BLOWING SNOW"                  
##  [35] "BLOWING SNOW & EXTREME WIND CH" "BLOWING SNOW- EXTREME WIND CHI"
##  [37] "BLOWING SNOW/EXTREME WIND CHIL" "BLOW-OUT TIDE"                 
##  [39] "BLOW-OUT TIDES"                 "BRUSH FIRE"                    
##  [41] "BRUSH FIRES"                    "COASTAL EROSION"               
##  [43] "Coastal Flood"                  "COASTALFLOOD"                  
##  [45] " COASTAL FLOOD"                 "COASTAL FLOOD"                 
##  [47] "coastal flooding"               "Coastal Flooding"              
##  [49] "COASTAL FLOODING"               "COASTAL  FLOODING/EROSION"     
##  [51] "COASTAL FLOODING/EROSION"       "Coastal Storm"                 
##  [53] "COASTALSTORM"                   "COASTAL STORM"                 
##  [55] "COASTAL SURGE"                  "COASTAL/TIDAL FLOOD"           
##  [57] "Cold"                           "COLD"                          
##  [59] "COLD AIR FUNNEL"                "COLD AIR FUNNELS"              
##  [61] "COLD AIR TORNADO"               "Cold and Frost"                
##  [63] "COLD AND FROST"                 "COLD AND SNOW"                 
##  [65] "COLD AND WET CONDITIONS"        "Cold Temperature"              
##  [67] "COLD TEMPERATURES"              "COLD WAVE"                     
##  [69] "COLD WEATHER"                   "COLD/WIND CHILL"               
##  [71] "COLD WIND CHILL TEMPERATURES"   "COLD/WINDS"                    
##  [73] "COOL AND WET"                   "COOL SPELL"                    
##  [75] "CSTL FLOODING/EROSION"          "Damaging Freeze"               
##  [77] "DAMAGING FREEZE"                "DAM BREAK"                     
##  [79] "DAM FAILURE"                    "DEEP HAIL"                     
##  [81] "DENSE FOG"                      "DENSE SMOKE"                   
##  [83] "DOWNBURST"                      "DOWNBURST WINDS"               
##  [85] "DRIEST MONTH"                   "Drifting Snow"                 
##  [87] "DROUGHT"                        "DROUGHT/EXCESSIVE HEAT"        
##  [89] "DROWNING"                       "DRY"                           
##  [91] "DRY CONDITIONS"                 "DRY HOT WEATHER"               
##  [93] "DRY MICROBURST"                 "DRY MICROBURST 50"             
##  [95] "DRY MICROBURST 53"              "DRY MICROBURST 58"             
##  [97] "DRY MICROBURST 61"              "DRY MICROBURST 84"             
##  [99] "DRY MICROBURST WINDS"           "DRY MIRCOBURST WINDS"          
## [101] "DRYNESS"                        "DRY PATTERN"                   
## [103] "DRY SPELL"                      "DRY WEATHER"                   
## [105] "DUST DEVEL"                     "Dust Devil"                    
## [107] "DUST DEVIL"                     "DUST DEVIL WATERSPOUT"         
## [109] "DUSTSTORM"                      "DUST STORM"                    
## [111] "DUST STORM/HIGH WINDS"          "EARLY FREEZE"                  
## [113] "Early Frost"                    "EARLY FROST"                   
## [115] "EARLY RAIN"                     "EARLY SNOW"                    
## [117] "Early snowfall"                 "EARLY SNOWFALL"                
## [119] "Erosion/Cstl Flood"             "EXCESSIVE"                     
## [121] "Excessive Cold"                 "EXCESSIVE HEAT"                
## [123] "EXCESSIVE HEAT/DROUGHT"         "EXCESSIVELY DRY"               
## [125] "EXCESSIVE PRECIPITATION"        "EXCESSIVE RAIN"                
## [127] "EXCESSIVE RAINFALL"             "EXCESSIVE SNOW"                
## [129] "EXCESSIVE WETNESS"              "Extended Cold"                 
## [131] "Extreme Cold"                   "EXTREME COLD"                  
## [133] "EXTREME COLD/WIND CHILL"        "EXTREME HEAT"                  
## [135] "EXTREMELY WET"                  "EXTREME/RECORD COLD"           
## [137] "EXTREME WINDCHILL"              "EXTREME WIND CHILL"            
## [139] "EXTREME WIND CHILL/BLOWING SNO" "EXTREME WIND CHILLS"           
## [141] "EXTREME WINDCHILL TEMPERATURES" "FALLING SNOW/ICE"              
## [143] "FIRST FROST"                    "FIRST SNOW"                    
## [145] " FLASH FLOOD"                   "FLASH FLOOD"                   
## [147] "FLASH FLOOD/"                   "FLASH FLOOD/FLOOD"             
## [149] "FLASH FLOOD/ FLOOD"             "FLASH FLOOD FROM ICE JAMS"     
## [151] "FLASH FLOOD - HEAVY RAIN"       "FLASH FLOOD/HEAVY RAIN"        
## [153] "FLASH FLOODING"                 "FLASH FLOODING/FLOOD"          
## [155] "FLASH FLOODING/THUNDERSTORM WI" "FLASH FLOOD/LANDSLIDE"         
## [157] "FLASH FLOOD LANDSLIDES"         "FLASH FLOODS"                  
## [159] "FLASH FLOOD/ STREET"            "FLASH FLOOODING"               
## [161] "Flood"                          "FLOOD"                         
## [163] "FLOOD FLASH"                    "FLOOD/FLASH"                   
## [165] "Flood/Flash Flood"              "FLOOD/FLASHFLOOD"              
## [167] "FLOOD/FLASH FLOOD"              "FLOOD/FLASH/FLOOD"             
## [169] "FLOOD/FLASH FLOODING"           "FLOOD FLOOD/FLASH"             
## [171] "FLOOD & HEAVY RAIN"             "FLOODING"                      
## [173] "FLOOD/RAIN/WIND"                "FLOOD/RAIN/WINDS"              
## [175] "FLOOD/RIVER FLOOD"              "FLOODS"                        
## [177] "Flood/Strong Wind"              "FLOOD WATCH/"                  
## [179] "FOG"                            "FOG AND COLD TEMPERATURES"     
## [181] "FOREST FIRES"                   "Freeze"                        
## [183] "FREEZE"                         "Freezing drizzle"              
## [185] "Freezing Drizzle"               "FREEZING DRIZZLE"              
## [187] "FREEZING DRIZZLE AND FREEZING"  "Freezing Fog"                  
## [189] "FREEZING FOG"                   "Freezing rain"                 
## [191] "Freezing Rain"                  "FREEZING RAIN"                 
## [193] "FREEZING RAIN AND SLEET"        "FREEZING RAIN AND SNOW"        
## [195] "FREEZING RAIN/SLEET"            "FREEZING RAIN SLEET AND"       
## [197] "FREEZING RAIN SLEET AND LIGHT"  "FREEZING RAIN/SNOW"            
## [199] "Freezing Spray"                 "Frost"                         
## [201] "FROST"                          "Frost/Freeze"                  
## [203] "FROST/FREEZE"                   "FROST\\FREEZE"                 
## [205] "FUNNEL"                         "Funnel Cloud"                  
## [207] "FUNNEL CLOUD"                   "FUNNEL CLOUD."                 
## [209] "FUNNEL CLOUD/HAIL"              "FUNNEL CLOUDS"                 
## [211] "FUNNELS"                        "Glaze"                         
## [213] "GLAZE"                          "GLAZE ICE"                     
## [215] "GLAZE/ICE STORM"                "gradient wind"                 
## [217] "Gradient wind"                  "GRADIENT WIND"                 
## [219] "GRADIENT WINDS"                 "GRASS FIRES"                   
## [221] "GROUND BLIZZARD"                "GUSTNADO"                      
## [223] "GUSTNADO AND"                   "GUSTY LAKE WIND"               
## [225] "GUSTY THUNDERSTORM WIND"        "GUSTY THUNDERSTORM WINDS"      
## [227] "Gusty Wind"                     "GUSTY WIND"                    
## [229] "GUSTY WIND/HAIL"                "GUSTY WIND/HVY RAIN"           
## [231] "Gusty wind/rain"                "Gusty winds"                   
## [233] "Gusty Winds"                    "GUSTY WINDS"                   
## [235] "HAIL"                           "Hail(0.75)"                    
## [237] "HAIL 075"                       "HAIL 0.75"                     
## [239] "HAIL 088"                       "HAIL 0.88"                     
## [241] "HAIL 100"                       "HAIL 1.00"                     
## [243] "HAIL 125"                       "HAIL 150"                      
## [245] "HAIL 175"                       "HAIL 1.75"                     
## [247] "HAIL 1.75)"                     "HAIL 200"                      
## [249] "HAIL 225"                       "HAIL 275"                      
## [251] "HAIL 450"                       "HAIL 75"                       
## [253] "HAIL 80"                        "HAIL 88"                       
## [255] "HAIL ALOFT"                     "HAIL DAMAGE"                   
## [257] "HAIL FLOODING"                  "HAIL/ICY ROADS"                
## [259] "HAILSTORM"                      "HAIL STORM"                    
## [261] "HAILSTORMS"                     "HAIL/WIND"                     
## [263] "HAIL/WINDS"                     "HARD FREEZE"                   
## [265] "HAZARDOUS SURF"                 "HEAT"                          
## [267] "Heatburst"                      "HEAT DROUGHT"                  
## [269] "HEAT/DROUGHT"                   "Heat Wave"                     
## [271] "HEAT WAVE"                      "HEAT WAVE DROUGHT"             
## [273] "HEAT WAVES"                     "HEAVY LAKE SNOW"               
## [275] "HEAVY MIX"                      "HEAVY PRECIPATATION"           
## [277] "Heavy Precipitation"            "HEAVY PRECIPITATION"           
## [279] "Heavy rain"                     "Heavy Rain"                    
## [281] "HEAVY RAIN"                     "HEAVY RAIN AND FLOOD"          
## [283] "Heavy Rain and Wind"            "HEAVY RAIN EFFECTS"            
## [285] "HEAVY RAINFALL"                 "HEAVY RAIN/FLOODING"           
## [287] "Heavy Rain/High Surf"           "HEAVY RAIN/LIGHTNING"          
## [289] "HEAVY RAIN/MUDSLIDES/FLOOD"     "HEAVY RAINS"                   
## [291] "HEAVY RAIN/SEVERE WEATHER"      "HEAVY RAINS/FLOODING"          
## [293] "HEAVY RAIN/SMALL STREAM URBAN"  "HEAVY RAIN/SNOW"               
## [295] "HEAVY RAIN/URBAN FLOOD"         "HEAVY RAIN; URBAN FLOOD WINDS;"
## [297] "HEAVY RAIN/WIND"                "HEAVY SEAS"                    
## [299] "HEAVY SHOWER"                   "HEAVY SHOWERS"                 
## [301] "HEAVY SNOW"                     "HEAVY SNOW AND"                
## [303] "HEAVY SNOW ANDBLOWING SNOW"     "HEAVY SNOW AND HIGH WINDS"     
## [305] "HEAVY SNOW AND ICE"             "HEAVY SNOW AND ICE STORM"      
## [307] "HEAVY SNOW AND STRONG WINDS"    "HEAVY SNOW/BLIZZARD"           
## [309] "HEAVY SNOW/BLIZZARD/AVALANCHE"  "HEAVY SNOW/BLOWING SNOW"       
## [311] "HEAVY SNOW   FREEZING RAIN"     "HEAVY SNOW/FREEZING RAIN"      
## [313] "HEAVY SNOW/HIGH"                "HEAVY SNOW/HIGH WIND"          
## [315] "HEAVY SNOW/HIGH WINDS"          "HEAVY SNOW/HIGH WINDS & FLOOD" 
## [317] "HEAVY SNOW/HIGH WINDS/FREEZING" "HEAVY SNOW & ICE"              
## [319] "HEAVY SNOW/ICE"                 "HEAVY SNOW/ICE STORM"          
## [321] "HEAVY SNOWPACK"                 "Heavy snow shower"             
## [323] "HEAVY SNOW/SLEET"               "HEAVY SNOW SQUALLS"            
## [325] "HEAVY SNOW-SQUALLS"             "HEAVY SNOW/SQUALLS"            
## [327] "HEAVY SNOW/WIND"                "HEAVY SNOW/WINTER STORM"       
## [329] "Heavy Surf"                     "HEAVY SURF"                    
## [331] "Heavy surf and wind"            "HEAVY SURF COASTAL FLOODING"   
## [333] "HEAVY SURF/HIGH SURF"           "HEAVY SWELLS"                  
## [335] "HEAVY WET SNOW"                 "HIGH"                          
## [337] "HIGH SEAS"                      "High Surf"                     
## [339] "HIGH SURF"                      "HIGH SURF ADVISORIES"          
## [341] "   HIGH SURF ADVISORY"          "HIGH SURF ADVISORY"            
## [343] "HIGH SWELLS"                    "HIGH  SWELLS"                  
## [345] "HIGH TEMPERATURE RECORD"        "HIGH TIDES"                    
## [347] "HIGH WATER"                     "HIGH WAVES"                    
## [349] "HIGHWAY FLOODING"               "High Wind"                     
## [351] "HIGH WIND"                      "HIGH WIND 48"                  
## [353] "HIGH WIND 63"                   "HIGH WIND 70"                  
## [355] "HIGH WIND AND HEAVY SNOW"       "HIGH WIND AND HIGH TIDES"      
## [357] "HIGH WIND AND SEAS"             "HIGH WIND/BLIZZARD"            
## [359] "HIGH WIND/ BLIZZARD"            "HIGH WIND/BLIZZARD/FREEZING RA"
## [361] "HIGH WIND DAMAGE"               "HIGH WIND (G40)"               
## [363] "HIGH WIND/HEAVY SNOW"           "HIGH WIND/LOW WIND CHILL"      
## [365] "HIGH WINDS"                     "HIGH  WINDS"                   
## [367] "HIGH WINDS/"                    "HIGH WINDS 55"                 
## [369] "HIGH WINDS 57"                  "HIGH WINDS 58"                 
## [371] "HIGH WINDS 63"                  "HIGH WINDS 66"                 
## [373] "HIGH WINDS 67"                  "HIGH WINDS 73"                 
## [375] "HIGH WINDS 76"                  "HIGH WINDS 80"                 
## [377] "HIGH WINDS 82"                  "HIGH WINDS AND WIND CHILL"     
## [379] "HIGH WINDS/COASTAL FLOOD"       "HIGH WINDS/COLD"               
## [381] "HIGH WINDS DUST STORM"          "HIGH WIND/SEAS"                
## [383] "HIGH WINDS/FLOODING"            "HIGH WINDS/HEAVY RAIN"         
## [385] "HIGH WINDS HEAVY RAINS"         "HIGH WINDS/SNOW"               
## [387] "HIGH WIND/WIND CHILL"           "HIGH WIND/WIND CHILL/BLIZZARD" 
## [389] "Hot and Dry"                    "HOT/DRY PATTERN"               
## [391] "HOT PATTERN"                    "HOT SPELL"                     
## [393] "HOT WEATHER"                    "HURRICANE"                     
## [395] "Hurricane Edouard"              "HURRICANE EMILY"               
## [397] "HURRICANE ERIN"                 "HURRICANE FELIX"               
## [399] "HURRICANE-GENERATED SWELLS"     "HURRICANE GORDON"              
## [401] "HURRICANE OPAL"                 "HURRICANE OPAL/HIGH WINDS"     
## [403] "HURRICANE/TYPHOON"              "HVY RAIN"                      
## [405] "HYPERTHERMIA/EXPOSURE"          "HYPOTHERMIA"                   
## [407] "Hypothermia/Exposure"           "HYPOTHERMIA/EXPOSURE"          
## [409] "ICE"                            "ICE AND SNOW"                  
## [411] "ICE FLOES"                      "Ice Fog"                       
## [413] "ICE JAM"                        "ICE JAM FLOODING"              
## [415] "Ice jam flood (minor"           "ICE ON ROAD"                   
## [417] "ICE PELLETS"                    "ICE ROADS"                     
## [419] "Ice/Snow"                       "ICE/SNOW"                      
## [421] "ICE STORM"                      "ICE STORM AND SNOW"            
## [423] "Icestorm/Blizzard"              "ICE STORM/FLASH FLOOD"         
## [425] "ICE/STRONG WINDS"               "Icy Roads"                     
## [427] "ICY ROADS"                      "LACK OF SNOW"                  
## [429] "Lake Effect Snow"               "LAKE EFFECT SNOW"              
## [431] "LAKE-EFFECT SNOW"               "LAKE FLOOD"                    
## [433] "LAKESHORE FLOOD"                "LANDSLIDE"                     
## [435] "LANDSLIDES"                     "LANDSLIDE/URBAN FLOOD"         
## [437] "Landslump"                      "LANDSLUMP"                     
## [439] "LANDSPOUT"                      "LARGE WALL CLOUD"              
## [441] "LATE FREEZE"                    "LATE SEASON HAIL"              
## [443] "LATE SEASON SNOW"               "Late-season Snowfall"          
## [445] "Late Season Snowfall"           "LATE SNOW"                     
## [447] "LIGHT FREEZING RAIN"            "LIGHTING"                      
## [449] "LIGHTNING"                      " LIGHTNING"                    
## [451] "LIGHTNING."                     "LIGHTNING AND HEAVY RAIN"      
## [453] "LIGHTNING AND THUNDERSTORM WIN" "LIGHTNING AND WINDS"           
## [455] "LIGHTNING DAMAGE"               "LIGHTNING FIRE"                
## [457] "LIGHTNING/HEAVY RAIN"           "LIGHTNING INJURY"              
## [459] "LIGHTNING THUNDERSTORM WINDS"   "LIGHTNING THUNDERSTORM WINDSS" 
## [461] "LIGHTNING  WAUSEON"             "Light snow"                    
## [463] "Light Snow"                     "LIGHT SNOW"                    
## [465] "LIGHT SNOW AND SLEET"           "Light Snowfall"                
## [467] "Light Snow/Flurries"            "LIGHT SNOW/FREEZING PRECIP"    
## [469] "LIGNTNING"                      "LOCAL FLASH FLOOD"             
## [471] "LOCAL FLOOD"                    "LOCALLY HEAVY RAIN"            
## [473] "LOW TEMPERATURE"                "LOW TEMPERATURE RECORD"        
## [475] "LOW WIND CHILL"                 "MAJOR FLOOD"                   
## [477] "Marine Accident"                "MARINE HAIL"                   
## [479] "MARINE HIGH WIND"               "MARINE MISHAP"                 
## [481] "MARINE STRONG WIND"             "MARINE THUNDERSTORM WIND"      
## [483] "MARINE TSTM WIND"               "Metro Storm, May 26"           
## [485] "Microburst"                     "MICROBURST"                    
## [487] "MICROBURST WINDS"               "Mild and Dry Pattern"          
## [489] "MILD/DRY PATTERN"               "MILD PATTERN"                  
## [491] "MINOR FLOOD"                    "Minor Flooding"                
## [493] "MINOR FLOODING"                 "MIXED PRECIP"                  
## [495] "Mixed Precipitation"            "MIXED PRECIPITATION"           
## [497] "MODERATE SNOW"                  "MODERATE SNOWFALL"             
## [499] "MONTHLY PRECIPITATION"          "Monthly Rainfall"              
## [501] "MONTHLY RAINFALL"               "Monthly Snowfall"              
## [503] "MONTHLY SNOWFALL"               "MONTHLY TEMPERATURE"           
## [505] "Mountain Snows"                 "MUD/ROCK SLIDE"                
## [507] "Mudslide"                       "MUDSLIDE"                      
## [509] "MUD SLIDE"                      "MUDSLIDE/LANDSLIDE"            
## [511] "Mudslides"                      "MUDSLIDES"                     
## [513] "MUD SLIDES"                     "MUD SLIDES URBAN FLOODING"     
## [515] "NEAR RECORD SNOW"               "NONE"                          
## [517] "NON SEVERE HAIL"                "NON-SEVERE WIND DAMAGE"        
## [519] "NON TSTM WIND"                  "NON-TSTM WIND"                 
## [521] "NORMAL PRECIPITATION"           "NORTHERN LIGHTS"               
## [523] "No Severe Weather"              "Other"                         
## [525] "OTHER"                          "PATCHY DENSE FOG"              
## [527] "PATCHY ICE"                     "Prolong Cold"                  
## [529] "PROLONG COLD"                   "PROLONG COLD/SNOW"             
## [531] "PROLONGED RAIN"                 "PROLONG WARMTH"                
## [533] "RAIN"                           "RAIN AND WIND"                 
## [535] "Rain Damage"                    "RAIN (HEAVY)"                  
## [537] "RAIN/SNOW"                      "RAINSTORM"                     
## [539] "RAIN/WIND"                      "RAPIDLY RISING WATER"          
## [541] "Record Cold"                    "RECORD COLD"                   
## [543] "RECORD  COLD"                   "RECORD COLD AND HIGH WIND"     
## [545] "RECORD COLD/FROST"              "RECORD COOL"                   
## [547] "Record dry month"               "RECORD DRYNESS"                
## [549] "RECORD/EXCESSIVE HEAT"          "RECORD/EXCESSIVE RAINFALL"     
## [551] "Record Heat"                    "RECORD HEAT"                   
## [553] "RECORD HEAT WAVE"               "Record High"                   
## [555] "RECORD HIGH"                    "RECORD HIGH TEMPERATURE"       
## [557] "RECORD HIGH TEMPERATURES"       "RECORD LOW"                    
## [559] "RECORD LOW RAINFALL"            "Record May Snow"               
## [561] "RECORD PRECIPITATION"           "RECORD RAINFALL"               
## [563] "RECORD SNOW"                    "RECORD SNOW/COLD"              
## [565] "RECORD SNOWFALL"                "Record temperature"            
## [567] "RECORD TEMPERATURE"             "Record Temperatures"           
## [569] "RECORD TEMPERATURES"            "RECORD WARM"                   
## [571] "RECORD WARM TEMPS."             "Record Warmth"                 
## [573] "RECORD WARMTH"                  "Record Winter Snow"            
## [575] "RED FLAG CRITERIA"              "RED FLAG FIRE WX"              
## [577] "REMNANTS OF FLOYD"              "RIP CURRENT"                   
## [579] "RIP CURRENTS"                   "RIP CURRENTS HEAVY SURF"       
## [581] "RIP CURRENTS/HEAVY SURF"        "RIVER AND STREAM FLOOD"        
## [583] "RIVER FLOOD"                    "River Flooding"                
## [585] "RIVER FLOODING"                 "ROCK SLIDE"                    
## [587] "ROGUE WAVE"                     "ROTATING WALL CLOUD"           
## [589] "ROUGH SEAS"                     "ROUGH SURF"                    
## [591] "RURAL FLOOD"                    "Saharan Dust"                  
## [593] "SAHARAN DUST"                   "Seasonal Snowfall"             
## [595] "SEICHE"                         "SEVERE COLD"                   
## [597] "SEVERE THUNDERSTORM"            "SEVERE THUNDERSTORMS"          
## [599] "SEVERE THUNDERSTORM WINDS"      "SEVERE TURBULENCE"             
## [601] "SLEET"                          "SLEET & FREEZING RAIN"         
## [603] "SLEET/FREEZING RAIN"            "SLEET/ICE STORM"               
## [605] "SLEET/RAIN/SNOW"                "SLEET/SNOW"                    
## [607] "SLEET STORM"                    "small hail"                    
## [609] "Small Hail"                     "SMALL HAIL"                    
## [611] "SMALL STREAM"                   "SMALL STREAM AND"              
## [613] "SMALL STREAM AND URBAN FLOOD"   "SMALL STREAM AND URBAN FLOODIN"
## [615] "SMALL STREAM FLOOD"             "SMALL STREAM FLOODING"         
## [617] "SMALL STREAM URBAN FLOOD"       "SMALL STREAM/URBAN FLOOD"      
## [619] "Sml Stream Fld"                 "SMOKE"                         
## [621] "Snow"                           "SNOW"                          
## [623] "Snow Accumulation"              "SNOW ACCUMULATION"             
## [625] "SNOW ADVISORY"                  "SNOW AND COLD"                 
## [627] "SNOW AND HEAVY SNOW"            "Snow and Ice"                  
## [629] "SNOW AND ICE"                   "SNOW AND ICE STORM"            
## [631] "Snow and sleet"                 "SNOW AND SLEET"                
## [633] "SNOW AND WIND"                  "SNOW/ BITTER COLD"             
## [635] "SNOW/BLOWING SNOW"              "SNOW/COLD"                     
## [637] "SNOW\\COLD"                     "SNOW DROUGHT"                  
## [639] "SNOWFALL RECORD"                "SNOW FREEZING RAIN"            
## [641] "SNOW/FREEZING RAIN"             "SNOW/HEAVY SNOW"               
## [643] "SNOW/HIGH WINDS"                "SNOW- HIGH WIND- WIND CHILL"   
## [645] "SNOW/ICE"                       "SNOW/ ICE"                     
## [647] "SNOW/ICE STORM"                 "SNOWMELT FLOODING"             
## [649] "SNOW/RAIN"                      "SNOW/RAIN/SLEET"               
## [651] "SNOW SHOWERS"                   "SNOW SLEET"                    
## [653] "SNOW/SLEET"                     "SNOW/SLEET/FREEZING RAIN"      
## [655] "SNOW/SLEET/RAIN"                "SNOW SQUALL"                   
## [657] "Snow squalls"                   "Snow Squalls"                  
## [659] "SNOW SQUALLS"                   "SNOWSTORM"                     
## [661] "SOUTHEAST"                      "STORM FORCE WINDS"             
## [663] "STORM SURGE"                    "STORM SURGE/TIDE"              
## [665] "STREAM FLOODING"                "STREET FLOOD"                  
## [667] "STREET FLOODING"                "Strong Wind"                   
## [669] "STRONG WIND"                    "STRONG WIND GUST"              
## [671] "Strong winds"                   "Strong Winds"                  
## [673] "STRONG WINDS"                   "Summary August 10"             
## [675] "Summary August 11"              "Summary August 17"             
## [677] "Summary August 21"              "Summary August 2-3"            
## [679] "Summary August 28"              "Summary August 4"              
## [681] "Summary August 7"               "Summary August 9"              
## [683] "Summary Jan 17"                 "Summary July 23-24"            
## [685] "Summary June 18-19"             "Summary June 5-6"              
## [687] "Summary June 6"                 "Summary: Nov. 16"              
## [689] "Summary: Nov. 6-7"              "Summary: Oct. 20-21"           
## [691] "Summary: October 31"            "Summary of April 12"           
## [693] "Summary of April 13"            "Summary of April 21"           
## [695] "Summary of April 27"            "Summary of April 3rd"          
## [697] "Summary of August 1"            "Summary of July 11"            
## [699] "Summary of July 2"              "Summary of July 22"            
## [701] "Summary of July 26"             "Summary of July 29"            
## [703] "Summary of July 3"              "Summary of June 10"            
## [705] "Summary of June 11"             "Summary of June 12"            
## [707] "Summary of June 13"             "Summary of June 15"            
## [709] "Summary of June 16"             "Summary of June 18"            
## [711] "Summary of June 23"             "Summary of June 24"            
## [713] "Summary of June 3"              "Summary of June 30"            
## [715] "Summary of June 4"              "Summary of June 6"             
## [717] "Summary of March 14"            "Summary of March 23"           
## [719] "Summary of March 24"            "SUMMARY OF MARCH 24-25"        
## [721] "SUMMARY OF MARCH 27"            "SUMMARY OF MARCH 29"           
## [723] "Summary of May 10"              "Summary of May 13"             
## [725] "Summary of May 14"              "Summary of May 22"             
## [727] "Summary of May 22 am"           "Summary of May 22 pm"          
## [729] "Summary of May 26 am"           "Summary of May 26 pm"          
## [731] "Summary of May 31 am"           "Summary of May 31 pm"          
## [733] "Summary of May 9-10"            "Summary: Sept. 18"             
## [735] "Summary Sept. 25-26"            "Summary September 20"          
## [737] "Summary September 23"           "Summary September 3"           
## [739] "Summary September 4"            "Temperature record"            
## [741] "THUDERSTORM WINDS"              "THUNDEERSTORM WINDS"           
## [743] "THUNDERESTORM WINDS"            "THUNDERSNOW"                   
## [745] "Thundersnow shower"             "THUNDERSTORM"                  
## [747] "THUNDERSTORM DAMAGE"            "THUNDERSTORM DAMAGE TO"        
## [749] "THUNDERSTORM HAIL"              "THUNDERSTORMS"                 
## [751] "THUNDERSTORMS WIND"             "THUNDERSTORMS WINDS"           
## [753] "THUNDERSTORMW"                  "THUNDERSTORMW 50"              
## [755] "Thunderstorm Wind"              "THUNDERSTORM WIND"             
## [757] "THUNDERSTORM WIND."             "THUNDERSTORM WIND 50"          
## [759] "THUNDERSTORM WIND 52"           "THUNDERSTORM WIND 56"          
## [761] "THUNDERSTORM WIND 59"           "THUNDERSTORM WIND 59 MPH"      
## [763] "THUNDERSTORM WIND 59 MPH."      "THUNDERSTORM WIND 60 MPH"      
## [765] "THUNDERSTORM WIND 65MPH"        "THUNDERSTORM WIND 65 MPH"      
## [767] "THUNDERSTORM WIND 69"           "THUNDERSTORM WIND 98 MPH"      
## [769] "THUNDERSTORM WIND/AWNING"       "THUNDERSTORM WIND (G40)"       
## [771] "THUNDERSTORM WIND G50"          "THUNDERSTORM WIND G51"         
## [773] "THUNDERSTORM WIND G52"          "THUNDERSTORM WIND G55"         
## [775] "THUNDERSTORM WIND G60"          "THUNDERSTORM WIND G61"         
## [777] "THUNDERSTORM WIND/HAIL"         "THUNDERSTORM WIND/LIGHTNING"   
## [779] "THUNDERSTORMWINDS"              "THUNDERSTORM WINDS"            
## [781] "THUNDERSTORM  WINDS"            "THUNDERSTORM W INDS"           
## [783] "THUNDERSTORM WINDS."            "THUNDERSTORM WINDS 13"         
## [785] "THUNDERSTORM WINDS 2"           "THUNDERSTORM WINDS 50"         
## [787] "THUNDERSTORM WINDS 52"          "THUNDERSTORM WINDS53"          
## [789] "THUNDERSTORM WINDS 53"          "THUNDERSTORM WINDS 60"         
## [791] "THUNDERSTORM WINDS 61"          "THUNDERSTORM WINDS 62"         
## [793] "THUNDERSTORM WINDS 63 MPH"      "THUNDERSTORM WINDS AND"        
## [795] "THUNDERSTORM WINDS/FLASH FLOOD" "THUNDERSTORM WINDS/ FLOOD"     
## [797] "THUNDERSTORM WINDS/FLOODING"    "THUNDERSTORM WINDS FUNNEL CLOU"
## [799] "THUNDERSTORM WINDS/FUNNEL CLOU" "THUNDERSTORM WINDS G"          
## [801] "THUNDERSTORM WINDS G60"         "THUNDERSTORM WINDSHAIL"        
## [803] "THUNDERSTORM WINDS HAIL"        "THUNDERSTORM WINDS/HAIL"       
## [805] "THUNDERSTORM WINDS/ HAIL"       "THUNDERSTORM WINDS HEAVY RAIN" 
## [807] "THUNDERSTORM WINDS/HEAVY RAIN"  "THUNDERSTORM WINDS      LE CEN"
## [809] "THUNDERSTORM WINDS LIGHTNING"   "THUNDERSTORM WINDSS"           
## [811] "THUNDERSTORM WINDS SMALL STREA" "THUNDERSTORM WINDS URBAN FLOOD"
## [813] "THUNDERSTORM WIND/ TREE"        "THUNDERSTORM WIND TREES"       
## [815] "THUNDERSTORM WIND/ TREES"       "THUNDERSTORM WINS"             
## [817] "THUNDERSTORMW WINDS"            "THUNDERSTROM WIND"             
## [819] "THUNDERSTROM WINDS"             "THUNDERTORM WINDS"             
## [821] "THUNDERTSORM WIND"              "THUNDESTORM WINDS"             
## [823] "THUNERSTORM WINDS"              "TIDAL FLOOD"                   
## [825] "Tidal Flooding"                 "TIDAL FLOODING"                
## [827] "TORNADO"                        "TORNADO DEBRIS"                
## [829] "TORNADOES"                      "TORNADOES, TSTM WIND, HAIL"    
## [831] "TORNADO F0"                     "TORNADO F1"                    
## [833] "TORNADO F2"                     "TORNADO F3"                    
## [835] "TORNADOS"                       "TORNADO/WATERSPOUT"            
## [837] "TORNDAO"                        "TORRENTIAL RAIN"               
## [839] "Torrential Rainfall"            "TROPICAL DEPRESSION"           
## [841] "TROPICAL STORM"                 "TROPICAL STORM ALBERTO"        
## [843] "TROPICAL STORM DEAN"            "TROPICAL STORM GORDON"         
## [845] "TROPICAL STORM JERRY"           "TSTM"                          
## [847] "TSTM HEAVY RAIN"                "TSTMW"                         
## [849] "Tstm Wind"                      " TSTM WIND"                    
## [851] "TSTM WIND"                      "TSTM WIND 40"                  
## [853] "TSTM WIND (41)"                 "TSTM WIND 45"                  
## [855] "TSTM WIND 50"                   "TSTM WIND 51"                  
## [857] "TSTM WIND 52"                   "TSTM WIND 55"                  
## [859] "TSTM WIND 65)"                  "TSTM WIND AND LIGHTNING"       
## [861] "TSTM WIND DAMAGE"               "TSTM WIND (G35)"               
## [863] "TSTM WIND (G40)"                " TSTM WIND (G45)"              
## [865] "TSTM WIND G45"                  "TSTM WIND  (G45)"              
## [867] "TSTM WIND (G45)"                "TSTM WIND G58"                 
## [869] "TSTM WIND/HAIL"                 "TSTM WINDS"                    
## [871] "TSTM WND"                       "TSUNAMI"                       
## [873] "TUNDERSTORM WIND"               "TYPHOON"                       
## [875] "Unseasonable Cold"              "UNSEASONABLY COLD"             
## [877] "UNSEASONABLY COOL"              "UNSEASONABLY COOL & WET"       
## [879] "UNSEASONABLY DRY"               "UNSEASONABLY HOT"              
## [881] "UNSEASONABLY WARM"              "UNSEASONABLY WARM AND DRY"     
## [883] "UNSEASONABLY WARM & WET"        "UNSEASONABLY WARM/WET"         
## [885] "UNSEASONABLY WARM YEAR"         "UNSEASONABLY WET"              
## [887] "UNSEASONAL LOW TEMP"            "UNSEASONAL RAIN"               
## [889] "UNUSUALLY COLD"                 "UNUSUALLY LATE SNOW"           
## [891] "UNUSUALLY WARM"                 "UNUSUAL/RECORD WARMTH"         
## [893] "UNUSUAL WARMTH"                 "URBAN AND SMALL"               
## [895] "URBAN AND SMALL STREAM"         "URBAN AND SMALL STREAM FLOOD"  
## [897] "URBAN AND SMALL STREAM FLOODIN" "Urban flood"                   
## [899] "Urban Flood"                    "URBAN FLOOD"                   
## [901] "Urban Flooding"                 "URBAN FLOODING"                
## [903] "URBAN FLOOD LANDSLIDE"          "URBAN FLOODS"                  
## [905] "URBAN SMALL"                    "URBAN/SMALL"                   
## [907] "URBAN/SMALL FLOODING"           "URBAN/SMALL STREAM"            
## [909] "URBAN SMALL STREAM FLOOD"       "URBAN/SMALL STREAM FLOOD"      
## [911] "URBAN/SMALL STREAM  FLOOD"      "URBAN/SMALL STREAM FLOODING"   
## [913] "URBAN/SMALL STRM FLDG"          "URBAN/SML STREAM FLD"          
## [915] "URBAN/SML STREAM FLDG"          "URBAN/STREET FLOODING"         
## [917] "VERY DRY"                       "VERY WARM"                     
## [919] "VOG"                            "Volcanic Ash"                  
## [921] "VOLCANIC ASH"                   "VOLCANIC ASHFALL"              
## [923] "Volcanic Ash Plume"             "VOLCANIC ERUPTION"             
## [925] "WAKE LOW WIND"                  "WALL CLOUD"                    
## [927] "WALL CLOUD/FUNNEL CLOUD"        "WARM DRY CONDITIONS"           
## [929] "WARM WEATHER"                   "WATERSPOUT"                    
## [931] " WATERSPOUT"                    "WATER SPOUT"                   
## [933] "WATERSPOUT-"                    "WATERSPOUT/"                   
## [935] "WATERSPOUT FUNNEL CLOUD"        "WATERSPOUTS"                   
## [937] "WATERSPOUT TORNADO"             "WATERSPOUT-TORNADO"            
## [939] "WATERSPOUT/TORNADO"             "WATERSPOUT/ TORNADO"           
## [941] "WAYTERSPOUT"                    "wet micoburst"                 
## [943] "WET MICROBURST"                 "Wet Month"                     
## [945] "WET SNOW"                       "WET WEATHER"                   
## [947] "Wet Year"                       "Whirlwind"                     
## [949] "WHIRLWIND"                      "WILDFIRE"                      
## [951] "WILDFIRES"                      "WILD FIRES"                    
## [953] "WILD/FOREST FIRE"               "WILD/FOREST FIRES"             
## [955] "Wind"                           "WIND"                          
## [957] " WIND"                          "WIND ADVISORY"                 
## [959] "WIND AND WAVE"                  "WIND CHILL"                    
## [961] "WIND CHILL/HIGH WIND"           "Wind Damage"                   
## [963] "WIND DAMAGE"                    "WIND GUSTS"                    
## [965] "WIND/HAIL"                      "WINDS"                         
## [967] "WIND STORM"                     "WINTER MIX"                    
## [969] "WINTER STORM"                   "WINTER STORM/HIGH WIND"        
## [971] "WINTER STORM HIGH WINDS"        "WINTER STORM/HIGH WINDS"       
## [973] "WINTER STORMS"                  "Winter Weather"                
## [975] "WINTER WEATHER"                 "WINTER WEATHER MIX"            
## [977] "WINTER WEATHER/MIX"             "WINTERY MIX"                   
## [979] "Wintry mix"                     "Wintry Mix"                    
## [981] "WINTRY MIX"                     "WND"
```

Not that valid values are:

|                         |                     |                   |
|-------------------------|---------------------|-------------------|
| Astronomical Low Tide   | Avalanche           | Blizzard          |
| Coastal Flood           | Cold/Wind Chill     | Debris Flow       |
| Dense Fog               | Dense Smoke         | Drought           |
| Dust Devil              | Dust Storm          | Excessive Heat    |
| Extreme Cold/Wind Chill | Flash Flood         | Flood             |
| Frost/Freeze            | Funnel Cloud        | Freezing Fog      |
| Hail                    | Heat                | Heavy Rain        |
| Heavy Snow              | High Surf           | High Wind         |
| Hurricane (Typhoon)     | Ice Storm           | Lake-Effect Snow  |
| Lakeshore Flood         | Lightning           | Marine Hail       |
| Marine High Wind        | Marine Strong Wind  | Marine Tstm Wind  |
| Rip Current             | Seiche              | Sleet             |
| Storm Surge/Tide        | Strong Wind         | Thunderstorm Wind |
| Tornado                 | Tropical Depression | Tropical Storm    |
| Tsunami                 | Volcanic Ash        | Waterspout        |
| Wildfire                | Winter Storm        | Winter Weather    |


A majority of the discrepancy is caused by typos and similar transcription
errors.


    if ( !dir.exists( tidydata.dir ) )
    {
        dir.create( tidydata.dir, recursive = TRUE );
    }
    saveRDS( retval, retfile );




#```{r make_tidy_data}
#
#
#
#```

### What is mean total number of steps taken per day?
#
#To calculate the total steps taken per day, we can group our `steps.table` by
#date. We can also take a look at the distribution of the total steps by
#generating a histogram.
#
#```{r steps_histogram}
#steps.by.date =
#    steps.table[, list(total.steps = sum(steps, na.rm = TRUE)), by = date];
#
#head( steps.by.date );
#
#summary( steps.by.date$total.steps, na.rm = TRUE );
#
#steps.mean.by.date = steps.by.date[ , mean(total.steps, na.rm = TRUE) ];
#steps.median.by.date = steps.by.date[ , median(total.steps, na.rm = TRUE) ];
#
#library( ggplot2 );
#
#gg.hist.steps =
#    ggplot( steps.by.date, aes( total.steps ), show.legend = TRUE ) +
#    geom_histogram( bins = 20, col="black", aes( fill=..count.. ) ) +
#    labs( x = "Total Steps per Day", y = "Count" ) +
#    labs( title = "Occurances of Total Steps Taken in One Day" ) +
#    geom_vline(
#        aes(
#            xintercept = steps.mean.by.date,
#            color = "Mean"
#        ),
#        size = 1,
#        show.legend = TRUE
#    ) +
#    geom_vline(
#        aes(
#            xintercept = steps.median.by.date,
#            color = "Median"
#        ),
#        size = 1,
#        show.legend = TRUE
#    ) +
#    scale_color_manual(
#        name = "",
#        values = c( Mean = "red", Median = "green" )
#    );
#               
#
#print( gg.hist.steps );
#
#```
#
#The mean and median total steps per day are:
#
#```{r steps_mean_median, results = 'hold'}
#cat( "The mean total steps per day is", steps.mean.by.date, "\n" );
#cat( "The median total steps per day is", steps.median.by.date, "\n" );
#```
#
### What is the average daily activity pattern?
#
#To determine average daily activity patterns, we can simply group by interval.
#
#```{r steps_time_series}
#steps.by.interval =
#    steps.table[, list(mean.steps = mean(steps, na.rm = TRUE)), by = interval];
#
#head( steps.by.interval );
#
#summary( steps.by.interval$mean.steps, na.rm = TRUE );
#
#steps.max.interval =
#    steps.by.interval[ mean.steps == max(mean.steps, na.rm = TRUE), interval ];
#
#library( ggplot2 );
#
#gg.mean.steps =
#    ggplot( steps.by.interval, aes( interval, mean.steps ) ) +
#    geom_line( color = "cyan3" ) +
#    labs( x = "Interval", y = "Number of Steps" ) +
#    labs( title = "Average Daily Activity Pattern" );
#
#print( gg.mean.steps );
#
#cat(
#    "The maximum average steps occurs during the interval at minute",
#    steps.max.interval,
#    "\n"
#);
#
#```
#
### Imputing missing values
#
#There may be some bias introduced into the dataset due to missing values. So,
#how many missing values do we have?
#
#```{r missing_values, results = 'hold'}
#cat( "Missing steps", sum(is.na( steps.table$steps )), "\n");
#cat( "Missing dates", sum(is.na( steps.table$date )), "\n");
#cat( "Missing intervals", sum(is.na( steps.table$interval )), "\n");
#
#cat(
#    "Proportion of missing steps is",
#    sum( is.na( steps.table$steps ) )/nrow( steps.table ),
#    "\n"
#);
#
#```
#
#So, only steps are missing from the dataset, which is good, but over 13% of
#entries for steps are `NA`, which may be very bad. We can estimate what these
#missing values might be by replacing them with the median steps for that
#interval. Apparently both mean and median are not good choices for imputation,
#but in this assignment we are allowed to use simple imputed values. I choose
#median because this dataset appears to have some skew in it.
#
#```{r fix_missing_steps}
#
#steps.table.fixed = data.table::copy(steps.table);
#steps.table.fixed[ ,
#    steps := ifelse(is.na(steps), median(steps, na.rm = TRUE), steps),
#    by = interval
#];
#
#str(steps.table.fixed);
#
#summary(steps.table$steps);
#summary(steps.table.fixed$steps);
#
#```
#
#Now, we create a histogram similar to the one created earlier.
#
#```{r fixed_steps_histogram}
#steps.by.date.fixed =
#    steps.table.fixed[ , list(total.steps = sum(steps)), by = date ];
#
#head( steps.by.date.fixed );
#
#summary( steps.by.date.fixed$total.steps );
#
#steps.mean.by.date.fixed = steps.by.date.fixed[ , mean(total.steps) ];
#steps.median.by.date.fixed = steps.by.date.fixed[ , median(total.steps) ];
#
#library( ggplot2 );
#
#gg.hist.steps.fixed =
#    ggplot( steps.by.date.fixed, aes( total.steps ), show.legend = TRUE ) +
#    geom_histogram( bins = 20, col="black", aes( fill=..count.. ) ) +
#    labs( x = "Total Steps per Day", y = "Count" ) +
#    labs( title = "Occurances of Total Steps Taken in One Day (with imputation)" ) +
#    geom_vline(
#        aes(
#            xintercept = steps.mean.by.date.fixed,
#            color = "Mean"
#        ),
#        size = 1,
#        show.legend = TRUE
#    ) +
#    geom_vline(
#        aes(
#            xintercept = steps.median.by.date.fixed,
#            color = "Median"
#        ),
#        size = 1,
#        show.legend = TRUE
#    ) +
#    scale_color_manual(
#        name = "",
#        values = c( Mean = "red", Median = "green" )
#    );
#               
#
#print( gg.hist.steps.fixed );
#
#```
#
#Our new mean and median are:
#
#```{r steps_mean_median_fixed, results = 'hold'}
#
#cat( "The mean fixed total steps per day is", steps.mean.by.date.fixed, "\n" );
#cat(
#    "The median fixed total steps per day is",
#    steps.median.by.date.fixed,
#    "\n"
#);
#```
#
#To remind ourselves of the original values:
#```{r steps_mean_median_reminder, results = 'hold'}
#
#cat( "The mean total steps per day is", steps.mean.by.date, "\n" );
#cat( "The median total steps per day is", steps.median.by.date, "\n" );
#```
#
#So, imputation using the median by interval changes the mean total daily steps
#so that it is slightly higher, by about 200 steps, and does not change the
#median number of steps per day.
#
### Are there differences in activity patterns between weekdays and weekends?
#
#We can determine this by creating a new factor for `steps.table.fixed`. 
#
#```{r factor_steps_fixed_time_series}
#steps.table.fixed[ ,
#    daytype :=
#        ifelse(
#            weekdays(date) %in% c("Saturday","Sunday"),
#            "Weekend",
#            "Weekday"
#        )
#];
#steps.table.fixed[ , daytype := factor(daytype) ];
#
#str(steps.table.fixed);
#print(steps.table.fixed);
#
#steps.by.interval.daytype =
#    steps.table.fixed[ ,
#        list(mean.steps = mean(steps)),
#        by = "interval,daytype"
#];
#
#head( steps.by.interval.daytype );
#
#summary( steps.by.interval.daytype$mean.steps );
#
#library( ggplot2 );
#
#gg.mean.steps.fixed =
#    ggplot( steps.by.interval.daytype, aes( interval, mean.steps ) ) +
#    geom_line( aes( color = daytype), show.legend = FALSE ) +
#    facet_grid( daytype ~ . ) +
#    labs( x = "Interval", y = "Number of Steps" ) +
#    labs( title = "Average Daily Activity Pattern" );
#
#print( gg.mean.steps.fixed );
#
#```
#
