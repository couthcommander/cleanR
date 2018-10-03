How to Clean Your Dataset in R
========================================================
author: Cole Beck
date: October 5, 2018
autosize: true
css: myrules.css

Data Preparation
========================================================

- Data Preprocessing
- Data Cleaning
- Data Wrangling

## Popular R packages
- tidyr, dplyr, purrr, stringr, lubridate

But we'll focus on base R

Raw Data
========================================================

TB data from WHO (http://www.who.int/tb/country/data/download/en/)




```r
url <- "https://extranet.who.int/tme/generateCSV.asp?ds=notifications"
tb <- read.csv(url, stringsAsFactors = FALSE)
```


```r
dim(tb)
```

```
[1] 8070  163
```

By default `stringsAsFactors` turns character strings into categorical variables. Setting it to `FALSE` ignores this behaviour.

Understand the Data
========================================================

Sample columns|Description
----|----
newrel_m014|new case, relapse, male, age 0-14
new_ep_m1524|new case, extrapulmonary TB, male, age 15-24
new_sn_f5564|new case, pulmonary TB - smear negative, female, age 55-64
new_sp_f65|new case, pulmonary TB - smear positive, female, age 65+
newrel_sexunk15plus|new case, relapse, sex unknown, age 15+
newrel_sexunkageunk|new case, relapse, sex unknown, age unknown

These column names contain 4 values (new, TB case, sex, age group)

USA 2008 - 2017
========================================================


```r
(ix <- which(tb[,'iso3'] == 'USA' & tb[,'year'] >= 2008 & tb[,'year'] <= 2017))
```

```
 [1] 7681 7682 7683 7684 7685 7686 7687 7688 7689 7690
```

```r
(us0817 <- tb[ix, c('iso3','year','newrel_m014','new_ep_m1524','new_sn_f5564','new_sp_f65')])
```

```
     iso3 year newrel_m014 new_ep_m1524 new_sn_f5564 new_sp_f65
7681  USA 2008          NA          146          233        300
7682  USA 2009          NA          121          225        247
7683  USA 2010          NA          114          193        223
7684  USA 2011          NA           94          212        269
7685  USA 2012          NA           88          200        243
7686  USA 2013         249           NA           NA         NA
7687  USA 2014         231           NA           NA         NA
7688  USA 2015         215           NA           NA         NA
7689  USA 2016         185           NA           NA         NA
7690  USA 2017         216           NA           NA         NA
```

USA 2008 - 2017
========================================================

Todo list

1. Rename "newrel" to "new_rel" (for consistency)
1. Go from wide to long data (10x6 => 40x4)
1. Extract values from column names (40x4 => 40x6)

Simple text substitution
========================================================


```r
# fix column name
names(us0817) <- sub('newrel', 'new_rel', names(us0817), fixed = TRUE)
```

Setting `fixed = TRUE` turns off regular expression pattern matching.

Several characters have special meanings in the context of a regular expression. For example, `^` can be used to indicate the given pattern should start at the beginning of a string.


```r
# verify column name changed
grep('^new_rel', names(us0817), value = TRUE)
```

```
[1] "new_rel_m014"
```

Re-shaping data
========================================================

Longitudinal data can be formatted as either "long" or "wide". The `reshape` function converts from one format to the other.


```r
(vcols <- grep('^new_', names(us0817), value = TRUE))
```

```
[1] "new_rel_m014" "new_ep_m1524" "new_sn_f5564" "new_sp_f65"  
```

```r
uslong <- reshape(us0817, varying = list(vcols), v.names = "cases", timevar = 'newvals',
        idvar = c('iso3', 'year'), times = vcols, direction = 'long')
dim(uslong)
```

```
[1] 40  4
```

Re-shaping data
========================================================


```r
uscnts <- uslong[!is.na(uslong[,'cases']),]
print(uscnts[1:10,], row.names = FALSE)
```

```
 iso3 year      newvals cases
  USA 2013 new_rel_m014   249
  USA 2014 new_rel_m014   231
  USA 2015 new_rel_m014   215
  USA 2016 new_rel_m014   185
  USA 2017 new_rel_m014   216
  USA 2008 new_ep_m1524   146
  USA 2009 new_ep_m1524   121
  USA 2010 new_ep_m1524   114
  USA 2011 new_ep_m1524    94
  USA 2012 new_ep_m1524    88
```

String extraction, with string-split
========================================================


```r
(newvals <- unique(uslong[,'newvals']))
```

```
[1] "new_rel_m014" "new_ep_m1524" "new_sn_f5564" "new_sp_f65"  
```

```r
strsplit(newvals, '_')
```

```
[[1]]
[1] "new"  "rel"  "m014"

[[2]]
[1] "new"   "ep"    "m1524"

[[3]]
[1] "new"   "sn"    "f5564"

[[4]]
[1] "new" "sp"  "f65"
```

String extraction, complex substitutions
========================================================

There are some good web sites to test regular expressions, such as https://regex101.com/


```r
type <- sub('.*_(.*)_.*', '\\1', newvals)
sex <- sub('.*_([mf]).*', '\\1', newvals)
age <- sub('.*_[mf](.*)', '\\1', newvals)
(newdata <- data.frame(newvals, type, sex, age))
```

```
       newvals type sex  age
1 new_rel_m014  rel   m  014
2 new_ep_m1524   ep   m 1524
3 new_sn_f5564   sn   f 5564
4   new_sp_f65   sp   f   65
```

Data merge
========================================================

Combine `uslong` and `newdata` by `newvals` column


```r
merge(uslong, newdata, by = 'newvals')[1:5,-1]
```

```
  iso3 year cases type sex  age
1  USA 2008   146   ep   m 1524
2  USA 2009   121   ep   m 1524
3  USA 2010   114   ep   m 1524
4  USA 2011    94   ep   m 1524
5  USA 2012    88   ep   m 1524
```

```r
# possible to do manual merge with `match`
match(uslong[,'newvals'], newdata[,'newvals'])
```

```
 [1] 1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3 3 3 3 3 3 3 3 3 4 4 4 4 4
[36] 4 4 4 4 4
```

Missing Data & Imputation
========================================================

Talk to a statistician about missing data!

* remove observations with missingness
* mean
* use same subject (longitudinal)
* sample similar observations
* regression

Removing Incomplete Records
========================================================

Smart pill data comes from TSHS (https://www.causeweb.org/tshs/smart-pill/)


```r
load('smartpill.Rdata')
sp <- smartpill[,1:10]
head(sp)
```

```
  Group Gender Race Height    Weight Age GE.Time SB.Time C.Time WG.Time
1     0      1   NA 182.88 102.05820  25    74.3     8.4     NA     816
2     0      1   NA 180.34 102.05820  39    73.3    13.8     NA     168
3     0      1   NA 180.34  68.03880  44     4.3     6.7     NA     240
4     0      1   NA 175.26  69.85317  53      NA      NA     NA     216
5     0      0   NA 152.40  44.90561  57    13.9     5.1     NA     120
6     0      1   NA 185.42  94.80073  43    23.3     8.7     NA     384
```

Understand the Data
========================================================

Sample columns|Description
----|----
Group|0 = Critically Ill, 1 = Healthy
GE.Time|gastric emptying time (in hours)
SB.Time|small bowel emptying time
C.Time|colonic transit time
WG.Time|whole gut time

Removing Incomplete Records
========================================================


```r
nrow(sp[!is.na(sp[,'GE.Time']),])
```

```
[1] 92
```

```r
nrow(sp[is.na(sp[,'GE.Time']),])
```

```
[1] 3
```

```r
nrow(sp[complete.cases(sp[,grep('Time$', names(sp))]),])
```

```
[1] 82
```

```r
allsp <- sp[complete.cases(sp),]
```

Group Mean
========================================================


```r
nsp <- sp
timecols <- grep('Time$', names(nsp), value = TRUE)
(gm <- sapply(split(nsp[, timecols], sp[,'Group']), colMeans, na.rm = TRUE))
```

```
                 0         1
GE.Time  28.885714  3.532471
SB.Time   7.114286  4.058916
C.Time         NaN 27.507073
WG.Time 309.000000 34.793953
```

```r
for(tc in timecols) {
  nsp[is.na(nsp[,tc]) & nsp[,'Group'] == 0, tc] <- gm[tc,1]
  nsp[is.na(nsp[,tc]) & nsp[,'Group'] == 1, tc] <- gm[tc,2]
}
```

(in general I wouldn't recommend this method of imputation)

Regression
========================================================


```r
mod <- lm(GE.Time ~ Group + Gender + Age + WG.Time, data = sp)
ix <- is.na(sp[,'GE.Time'])
sp[ix,'GE.Time'] <- predict(mod, sp[ix,])
sp[ix,]
```

```
   Group Gender Race  Height   Weight Age   GE.Time SB.Time C.Time WG.Time
4      0      1   NA 175.260 69.85317  53 22.720393      NA     NA  216.00
61     1      0    1 157.480 51.70949  66  6.793628      NA     NA   72.08
86     1      1    3 145.542 77.06528  35  3.044958      NA     NA   23.29
```

Within Subject
========================================================

Simulated longitudinal data


```r
ld <- data.frame(
  id = sample(1001:1010, 150, replace = TRUE),
  visitDate = as.Date(sample(1461, 150, replace = TRUE), origin = '2010-01-01'),
  testScore = round(rnorm(150, 75, 10))
)
height <- round(runif(10, 60, 75))
ld[,'height'] <- height[ld[,'id']-1000]
ld <- ld[order(ld[,'id'], ld[,'visitDate']),]
ld[sample(150, 10), 'testScore'] <- NA
ld[sample(150, 25), 'height'] <- NA
```

Mean test score
========================================================

Impute test score by taking mean test score for each subject


```r
mt <- tapply(ld[,'testScore'], ld[,'id'], mean, na.rm = TRUE)
ix <- which(is.na(ld[,'testScore']))
ld[ix,'testScore'] <- mt[as.character(ld[ix,'id'])]
head(ld[ix,])
```

```
      id  visitDate testScore height
126 1001 2012-06-16  74.23810     NA
22  1001 2012-12-19  74.23810     67
123 1002 2010-08-12  73.26667     65
65  1004 2010-05-24  72.46667     74
70  1005 2012-07-27  74.30769     67
26  1005 2013-11-03  74.30769     67
```

Last observation carried forward
========================================================


```r
locf <- function(x) {
  ix <- which(is.na(x))
  for(i in ix) {
    if(i > 1 && !is.na(x[i-1])) {
      x[i] <- x[i-1]
    }
  }
  x
}
sld <- split(ld[,'height'], ld[,'id'])
ld[,'height'] <- unsplit(lapply(sld, locf), ld[,'id'])
sum(is.na(ld[,'height']))
```

```
[1] 2
```

Data Validation
========================================================

Once data is structured, determine what should be validated

* global (min/max/missing)
* within observation
* within subject

Jennifer Thompson has an excellent presentation tailored to REDCap and project workflow:

https://github.com/jenniferthompson/DataCleanExample/

Global Validation
========================================================

Define some simple functions


```r
lowrange <- function(x, thr) x < thr
highrange <- function(x, thr) x > thr
badrange <- function(x, low, high) lowrange(x, low) | highrange(x, high)
badrange(c(1, 5, -3, 25, NA, 42), 0, 10)
```

```
[1] FALSE FALSE  TRUE  TRUE    NA  TRUE
```


```r
badage <- badrange(sp[,'Age'], 20, 40)
noage <- is.na(sp[,'Age'])
sum(badage)
```

```
[1] 37
```

Global Validation
========================================================


```r
badbmi <- highrange(sp[,'Weight'] / (sp[,'Height'] / 100)^2, 30)
isbad <- cbind(badage, noage, badbmi)
head(isbad)
```

```
     badage noage badbmi
[1,]  FALSE FALSE   TRUE
[2,]  FALSE FALSE   TRUE
[3,]   TRUE FALSE  FALSE
[4,]   TRUE FALSE  FALSE
[5,]   TRUE FALSE  FALSE
[6,]   TRUE FALSE  FALSE
```

```r
anybad <- rowSums(isbad) > 0
gooddat <- sp[!anybad,]
baddat <- sp[anybad,]
```

Validation within an observation
========================================================

Require `C.Time > SB.Time` if SB.Time is not missing


```r
ix <- !is.na(sp[,'SB.Time']) & (is.na(sp[,'C.Time']) | sp[,'C.Time'] <= sp[,'SB.Time'])
tail(sp[ix,])
```

```
   Group Gender Race Height    Weight Age GE.Time SB.Time C.Time WG.Time
21     1      0    2 154.94  79.37860  50    5.68   13.45  13.20   32.33
29     1      1    2 175.26  86.18248  44    2.72    3.71   1.19    7.62
63     1      1    1 191.77 127.00576  43    2.07    3.95   2.45    8.47
70     1      0    1 167.64  68.03880  38    2.39    4.88   2.48    9.75
80     1      1    1 179.07  64.63686  26    4.11    3.50     NA      NA
92     1      1    1 187.96  90.71840  28    4.07    3.70   0.70    8.47
```

Be careful with NA values

Validation within an subject
========================================================

Require last testScore >= mean testScore for each subject


```r
lowscore <- function(x) {
  x[length(x)] < mean(x)
}
sld <- split(ld[,'testScore'], ld[,'id'])
badid <- names(which(sapply(sld, lowscore)))
finalld <- ld[!(ld[,'id'] %in% badid),]
dim(finalld)
```

```
[1] 73  4
```

Contact Me
========================================================

## R clinic

1st Friday of each month

1:30pm in the Biostatistics Conference Room

11105, 2525 West End Avenue

Contact Cole at [cole.beck@vumc.org](mailto:cole.beck@vumc.org?subject=R Clinic)
