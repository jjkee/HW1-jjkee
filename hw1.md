BIOST 578 HW 1  
========================================================


The goal of this homework is to find all HCV gene expression data using the Illumina platform submitted by an investigator at Yale.

First, install from Bioconductor the `GEOmetadb` package, which contains all metadata associated with the GEO database.  


```r
source("http://bioconductor.org/biocLite.R")
biocLite("GEOmetadb")
```


Then load the package. 


```r
library("GEOmetadb")
```


We also need to initiate a local SQLite database to store the metadata associated with the GEO database.


```r
getSQLiteFile()  # only need to do this once 
```


Then fill in the SQLite database with the GEO database metadata. 


```r
geo_con <- dbConnect(SQLite(), "GEOmetadb.sqlite")
```


# Using the `dbGetQuery` function

Now we can start quering the database to get information on the dataset that we want. We will use the `dbGetQuery` function to do the query. Specifically we are looking for entrees that have "HCV" in the title cell, "Illumina" in the manufacturer cell and "Yale" in the contact cell. The output is the title, GSE accession number, GPL accession number and manufacturer and description of platform for entrees that match. 


```r
dbGetQuery(geo_con, "SELECT gse.title, gse.gse, gpl.gpl, gpl.manufacturer, gpl.description FROM (gse JOIN gse_gpl ON gse.gse=gse_gpl.gse) j JOIN gpl ON j.gpl=gpl.gpl WHERE gse.title LIKE '%HCV%' AND gpl.manufacturer LIKE '%Illumina%' AND gse.contact LIKE '%Yale%';")
```

```
##                                                            gse.title
## 1 The blood transcriptional signature of chronic HCV [Illumina data]
## 2                 The blood transcriptional signature of chronic HCV
##    gse.gse  gpl.gpl gpl.manufacturer
## 1 GSE40223 GPL10558    Illumina Inc.
## 2 GSE40224 GPL10558    Illumina Inc.
##                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       gpl.description
## 1 The HumanHT-12 v4 Expression BeadChip provides high throughput processing of 12 samples per BeadChip without the need for expensive, specialized automation. The BeadChip is designed to support flexible usage across a wide-spectrum of experiments.;\t;\tThe updated content on the HumanHT-12 v4 Expression BeadChips provides more biologically meaningful results through genome-wide transcriptional coverage of well-characterized genes, gene candidates, and splice variants.;\t;\tEach array on the HumanHT-12 v4 Expression BeadChip targets more than 31,000 annotated genes with more than 47,000 probes derived from the National Center for Biotechnology Information Reference Sequence (NCBI) RefSeq Release 38 (November 7, 2009) and other sources.;\t;\tPlease use the GEO Data Submission Report Plug-in v1.0 for Gene Expression which may be downloaded from https://icom.illumina.com/icom/software.ilmn?id=234 to format the normalized and raw data.  These should be submitted as part of a GEOarchive.  Instructions for assembling a GEOarchive may be found at http://www.ncbi.nlm.nih.gov/projects/geo/info/spreadsheet.html;\t;\tOctober 11, 2012: annotation table updated with HumanHT-12_V4_0_R2_15002873_B.txt
## 2 The HumanHT-12 v4 Expression BeadChip provides high throughput processing of 12 samples per BeadChip without the need for expensive, specialized automation. The BeadChip is designed to support flexible usage across a wide-spectrum of experiments.;\t;\tThe updated content on the HumanHT-12 v4 Expression BeadChips provides more biologically meaningful results through genome-wide transcriptional coverage of well-characterized genes, gene candidates, and splice variants.;\t;\tEach array on the HumanHT-12 v4 Expression BeadChip targets more than 31,000 annotated genes with more than 47,000 probes derived from the National Center for Biotechnology Information Reference Sequence (NCBI) RefSeq Release 38 (November 7, 2009) and other sources.;\t;\tPlease use the GEO Data Submission Report Plug-in v1.0 for Gene Expression which may be downloaded from https://icom.illumina.com/icom/software.ilmn?id=234 to format the normalized and raw data.  These should be submitted as part of a GEOarchive.  Instructions for assembling a GEOarchive may be found at http://www.ncbi.nlm.nih.gov/projects/geo/info/spreadsheet.html;\t;\tOctober 11, 2012: annotation table updated with HumanHT-12_V4_0_R2_15002873_B.txt
```


The first row contains information on the Illumina dataset that we will want to use. I think the second row is just a collection of all datasets used in the study. 

# Using `data.table`s

Alternatively, we can convert the relevant database tables into data tables and query the data tables instead for the data that we want.

First, load the `data.table` package. 


```r
library(data.table)
```


Convert the gse, gpl and gse_gpl database tables into data tables.


```r
gse <- data.table(dbReadTable(con = geo_con, name = "gse", row.names = FALSE, 
    header = TRUE, sep = "\t"))
gpl <- data.table(dbReadTable(con = geo_con, name = "gpl", row.names = FALSE, 
    header = TRUE, sep = "\t"))
gse_gpl <- data.table(dbReadTable(con = geo_con, name = "gse_gpl", row.names = FALSE, 
    header = TRUE, sep = "\t"))
```


We need to merge the gse, gpl and gse_gpl data tables into one table before we can extract the information we want. Because some of the column names in the gse and gpl tables are the same, to avoid confusion after merging the tables, we will prefix the column names in each table by the name of the table (e.g. change the ID column name in the gse table into gse.ID).


```r
setnames(gse, apply(as.matrix(colnames(gse)), MARGIN = 1, FUN = function(x) {
    paste0("gse.", x)
}))
setnames(gpl, apply(as.matrix(colnames(gpl)), MARGIN = 1, FUN = function(x) {
    paste0("gpl.", x)
}))
```


The merging is a two-step process. First we merge the gse and gse_gpl tables using the gse column as a key. Then we merge the resulting table with the gpl table using the gpl column as a key.


```r
## Set keys to join gse and gse_gpl tables
setkey(gse, "gse.gse")
setkey(gse_gpl, "gse")

## Inner join gse and gse_gpl tables
j <- gse[gse_gpl, nomatch = 0]

## Set keys to join j and gpl tables
setkey(j, "gpl")
setkey(gpl, "gpl.gpl")

## Inner join j and gpl tables
k <- gpl[j, nomatch = 0]
```


Finally proceed with the query. This query is equivalent to the query we did using the `dbGetQuery` function.


```r
k[gse.title %like% "HCV" & gpl.manufacturer %like% "Illumina" & gse.contact %like% 
    "Yale", list(gse.title, gse.gse, gpl.gpl, gpl.manufacturer, gpl.description)]
```

```
##                                                             gse.title
## 1: The blood transcriptional signature of chronic HCV [Illumina data]
## 2:                 The blood transcriptional signature of chronic HCV
##     gse.gse  gpl.gpl gpl.manufacturer
## 1: GSE40223 GPL10558    Illumina Inc.
## 2: GSE40224 GPL10558    Illumina Inc.
##                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        gpl.description
## 1: The HumanHT-12 v4 Expression BeadChip provides high throughput processing of 12 samples per BeadChip without the need for expensive, specialized automation. The BeadChip is designed to support flexible usage across a wide-spectrum of experiments.;\t;\tThe updated content on the HumanHT-12 v4 Expression BeadChips provides more biologically meaningful results through genome-wide transcriptional coverage of well-characterized genes, gene candidates, and splice variants.;\t;\tEach array on the HumanHT-12 v4 Expression BeadChip targets more than 31,000 annotated genes with more than 47,000 probes derived from the National Center for Biotechnology Information Reference Sequence (NCBI) RefSeq Release 38 (November 7, 2009) and other sources.;\t;\tPlease use the GEO Data Submission Report Plug-in v1.0 for Gene Expression which may be downloaded from https://icom.illumina.com/icom/software.ilmn?id=234 to format the normalized and raw data.  These should be submitted as part of a GEOarchive.  Instructions for assembling a GEOarchive may be found at http://www.ncbi.nlm.nih.gov/projects/geo/info/spreadsheet.html;\t;\tOctober 11, 2012: annotation table updated with HumanHT-12_V4_0_R2_15002873_B.txt
## 2: The HumanHT-12 v4 Expression BeadChip provides high throughput processing of 12 samples per BeadChip without the need for expensive, specialized automation. The BeadChip is designed to support flexible usage across a wide-spectrum of experiments.;\t;\tThe updated content on the HumanHT-12 v4 Expression BeadChips provides more biologically meaningful results through genome-wide transcriptional coverage of well-characterized genes, gene candidates, and splice variants.;\t;\tEach array on the HumanHT-12 v4 Expression BeadChip targets more than 31,000 annotated genes with more than 47,000 probes derived from the National Center for Biotechnology Information Reference Sequence (NCBI) RefSeq Release 38 (November 7, 2009) and other sources.;\t;\tPlease use the GEO Data Submission Report Plug-in v1.0 for Gene Expression which may be downloaded from https://icom.illumina.com/icom/software.ilmn?id=234 to format the normalized and raw data.  These should be submitted as part of a GEOarchive.  Instructions for assembling a GEOarchive may be found at http://www.ncbi.nlm.nih.gov/projects/geo/info/spreadsheet.html;\t;\tOctober 11, 2012: annotation table updated with HumanHT-12_V4_0_R2_15002873_B.txt
```


Note that the query result is the same as `dbGetQuery`.
