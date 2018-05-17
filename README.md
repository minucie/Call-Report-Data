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

setwd('..')
upath <- getwd()
setwd(path)
```

## Part-3A Need to do manual downloads of FFIEC bulk data zip file and put in manually in rawdata/cdr folder.
Please download the Call Reports-- balance Sheets, Income Statement and Past Due--Four Periods for years in Tab Deliminated format from the following link: https://cdr.ffiec.gov/public/PWS/DownloadBulkData.aspx . This process has to be done manually. Please save them in the `rawdata/cdr` folder.

The following code unzip the downloaded file. 
```
dir.create(file.path(upath, 'rawdata/cdr'))
dir.create(file.path(upath, 'rawdata/cdr/unzip'))
fileList <- list.files(path =  paste0(upath, '/rawdata/cdr'), pattern="FFIEC")
foldername <- gsub('.zip', '', fileList)
for(i in 1:length(foldername)){
  dir.create(file.path(upath, paste0('rawdata/cdr/unzip/', foldername[i])))
}

setwd(paste0(upath, '/rawdata/cdr'))
for(i in 1:length(fileList)){
  unzip(fileList[i], exdir = paste0(getwd(),'/unzip/', foldername[i]))
}
```

## Part-3B Import data, merge and rbind
The following code import all the unzipped FFIEC CDR data. For each year, it merges the dataset and creats a list of dataset: DF for main data frame and CB for codebook. The data for each year is row binded with year wise identifier. If new variables are introduced or old variable are dropped, the code will generate NA.
```
datapath <- paste0(upath, '/rawdata/cdr/unzip/')
setwd(datapath)
DF <- list()
CB <- list()
for(i in 1:length(foldername)){
  dir <- paste0(datapath, foldername[i])
  setwd(dir)
  if("Readme.txt" %in% list.files(path = dir)) unlink("README.txt")
  FNames <- list.files(path = dir)
  CR <- read.delim(FNames[1], stringsAsFactors = FALSE)
  for (j in 2:length(FNames)){
    temp <- read.delim(FNames[j], stringsAsFactors = FALSE)
    CR <- merge(CR, temp)
    rm(temp)
  }
    DF[[i]] <- CR[ -1,]
    CB[[i]] <- CR[1,]
}
rm(CR)

lapply(DF, dim)
names(DF) <- paste0('y', 2011:2017)
ipak('data.table')
df <- rbindlist(DF, use.names=TRUE, fill=TRUE, idcol="YID")
setwd(upath)
write.csv(df, 'main.csv') ## ~1GB file
```

## Part-3C Create modified code book
```
names(CB) <- paste0('y', 2011:2017)
cb <- rbindlist(CB, use.names=TRUE, fill=TRUE, idcol="YID")
setwd(upath)
write.csv(cb, 'codebook.csv')
```
