# Call-Report-Data
Cleaning/merging call report data from FFIEC

## Clean the global enviroment
```
rm(list = ls())
dev.off(dev.list()['RStudioGD'])
```

## Part-1 : Load all the required packages
```
ipak <- function(pkg){
  new.pkg <- pkg[!(pkg %in% installed.packages()[, 'Package'])]
  if (length(new.pkg)) 
    install.packages(new.pkg, dependencies = TRUE, repos = 'http://cran.us.r-project.org')
  sapply(pkg, require, character.only = TRUE)
}
ipak(c('rstudioapi', 'Hmisc'))
```

## Part-2 : Setup Working Directory
```
path <- dirname(rstudioapi::getActiveDocumentContext()$path)
setwd(path)
```

## Part-3 : Download, unzip and load the stuctural files of Chicago File in R environment
```
setwd('..')
upath <- getwd()
setwd(path)
dir.create(file.path(upath, 'rawdata'))
dir.create(file.path(upath, 'rawdata/ffiec'))
dir.create(file.path(upath, 'rawdata/ffiec/unzip'))

for(y in 11:17){
  for(q in c('03', '06', '09', '12')){
    dfname <- paste0('rawdata/ffiec/', 'call', y, q, '-zip', '.zip')
    myurl <- paste0('https://www.chicagofed.org/~/media/others/banking/financial-institution-reports/commercial-bank-data/','call', y, q, '-zip', '.zip')
    if (file.exists(dfname) == FALSE) { # get the zip file
      download.file(myurl, destfile = dfname, mode='wb')
    }
    unzip(dfname, exdir = 'rawdata/ffiec/unzip')
  }
}

fileList <- list.files(path=paste0(upath, '/rawdata/ffiec/unzip'), pattern=".xpt")
setwd(paste0(upath, '/rawdata/ffiec/unzip'))
df <- lapply(fileList, sasxport.get)
setwd(path)
```

## Part-4A Need to do manual downloads of FFIEC bulk data zip file and put in manually in rawdata/cdr folder.
Please download the Call Reports-- balance Sheets, Income Statement and Past Due--Four Periods for years in Tab Deliminated format from the following link: https://cdr.ffiec.gov/public/PWS/DownloadBulkData.aspx . This process has to be done manually. Please save them in the `rawdata/cdr` folder.
```
dir.create(file.path(upath, 'rawdata/cdr'))
dir.create(file.path(upath, 'rawdata/cdr/unzip'))
fileList <- list.files(path =  paste0(upath, '/rawdata/cdr'), pattern="FFIEC")
setwd(paste0(upath, '/rawdata/cdr/'))
lapply(fileList, function(k) unzip(k, exdir = 'unzip'))
setwd(path)
```

## Part-4B Import data
```
CRLoad <- function(dir){
  if("Readme.txt" %in% list.files(path = dir)) unlink("README.txt")
  FNames <- list.files(path = dir)
  CR <- read.delim(FNames[1], stringsAsFactors = FALSE)
  for (i in 2:length(FNames)){
    temp <- read.delim(FNames[i], stringsAsFactors = FALSE)
    CR <- merge(CR, temp)
  }
  CR
}

setwd(paste0(upath, '/rawdata/cdr/unzip/'))
Merged <- CRLoad(getwd())
colnames(Merged) <- tolower(colnames(Merged))
setwd(path)
```

## Part-4C Create modified code book
```
CRCodeBook <- function(x){
  symbol <- names(x)
  # First row has short descriptions #
  descript <- as.character(x[1, ])
  CB <- data.frame(symbol, descript, stringsAsFactors = FALSE)
  CB <- subset(CB, descript != "")
  CB <- subset(CB, descript != "NA")
  CB
}

CodeBook <- CRCodeBook(Merged)
```

## Part-5 : Extra comand to drop or retain variable based on their pattern names.
Given the names of variable the following codes will either retain them or drop them.
```
dropvar <- function(df, grepID){
  make_match <- paste(grepID, collapse='|')
  df <- df[ ,-grep(make_match, colnames(df))]
  return(df)
}
```

```
retainvar <- function(df, grepID){
  make_match <- paste(grepID,collapse='|')
  df2 <- df[ ,grep(make_match, colnames(df))]
  #df <- cbind(df1, df2)
  return(df2)
}
```
The `dft` will be a list of data for each year with the common variable names.
```
dft <- lapply(df, function(k) retainvar(k, grepID = colnames(Merged)))
```
