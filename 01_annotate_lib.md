<link href="http://kevinburke.bitbucket.org/markdowncss/markdown.css" rel="stylesheet"></link>



# Step 01 - Annotating Library Member Data Frame

## Motivation

Here we are loading all the library member sequences into a DataFrame called `lib_seqs` and annotating them based on their promoter, RBS, CDS, gene, etc. At the end we check to make sure that the library has every item, the sequences all check out, and that the sequences are all the same length.

## Splitting 

We want to take `203.norestrict.fa` and split it on the upper/lowercase  boundaries. First, lets look at the UC/LC situation for each of the 
sequence types in 203 and also the 202 library.

### Comparison of Library Sequences between 203 and 202

The 203 library consists of a promoter, a RBS, an ATG start codon, and then  exactly 10 more codons (30 bp) of the sequence in question. So I want to cut at the lc/uc boundary, then cut again at the ATG.

Things are different for the natural UTR, like this:

```
>BBaJ23100-ribF-1
ttgacggctagctcagtcctaggtacagtgctagcTTAATTTCACTGTTTTGAGCCAGACATGAAGCTGATACG...
```

Compared to the designed UTR, like this:

```
>BBaJ23100-ribF-2
ttgacggctagctcagtcctaggtacagtgctagcTTAATATTAAAGAGGAGAAAtactagATGAAGCTGATAC...
```

Here is an example of that same promo/RBS pair in 202:

```console
$ grep -B1 -i TTAATATTAAAGAGGAGAAA 202.norestrict.fa
>BBa_J23100--B0030_RBS
TTGACGGCTAGCTCAGTCCTAGGTACAGTGCTAGCTTAATattaaagaggagaaatta
```

### Aligning them by hand:

```
wt  ttgacggctagctcagtcctaggtacagtgctagcTTAAT TTCACTGTTTTGAGCCAGACATGAAGCTGA...
                                   bbbbb.rrrrrrrrrrrrrrrrrrrr    MET
rbs ttgacggctagctcagtcctaggtacagtgctagcTTAATATTAAAGAGGAGAAAtactagATGAAGCTGA...
                                   bbbbbrrrrrrrrrrrrrrrrrrrXXXXXXMET
202 TTGACGGCTAGCTCAGTCCTAGGTACAGTGCTAGCTTAATattaaagaggagaaattaCATATG <GFP>
                                   bbbbbrrrrrrrrrrrrrrrrrrrXXX   MET
```

`b` for barcode, `r` for RBS, and `X` for variable rbs/cds spacer, and    `MET`for start codon.

So, take home messages:

* WT promo/rbs combos in 203 have no spacer between the WT RBS and the MET codon. 
* `BBaJ23100/BBaJ23108` RBSes have `TACATG` separating the RBS sequence and the MET. This is different than the same RBSs for 202, which necessarily end in `CATATG`. So they are not equivalent, strictly speaking. Which might complicate comparison of 202 and 203 libraries. 
* The equivalent 202 RBSes (`BBa_J23100/BBa_J23108`) have the barcode on the promoter, not the RBS (which is actually more correct.)

So first we need to annotate the library sequences and split them up using this information. 

For the 203 sequences, we split into 
* *Promoter* -  all LC seq + 5 bases (for barcode)
* *RBS* - middle of this sequence
* *CDS* - last 33 bases (ATG + 10 aa)

### Regex Prep for reaidng FASTA library sequences into R

I reformatted `203.norestrict.fa` into a tab-delimited file with perl regexp. I also added column headers. 

```bash
echo -en "Name\tPromoter\tGene\tCDS.num\tPromoter.seq\tRBS.seq\tCDS.seq" \
    > 203.norestrict.txt

perl -ne 'chomp; s/^>((\w+)-(\w+)-(\d+))/\n$1\t$2\t$3\t$4/; 
    s/^([atgc]+[ATGC]{5})([ATGCatgc]*?)(ATG[ATGCatgc]{30})$/\t$1\t$2\t$3/;
    print $_;' 203.norestrict.fa >> 203.norestrict.txt
```

## Reading Into R

Now we have to read `203.norestrict.txt` into R as a tab-delimited text file. 



```r
lib_seqs <- read.table(file = paste(getwd(), "/data/203.norestrict.txt", 
    sep = ""), sep = "\t", header = T)
```





### Annotating `lib_seqs` DataFrame

Now we want to add the library info to each sequence.

Using the leader peptide number (`CDS.num`) we can assign the predetermined attributes to each sequence, like RBS identity, codon usage, and secondary structure. 



```r
lib_seqs$CDS.num <- as.integer(lib_seqs$CDS.num)

# split Leader into RBS identities
lib_seqs$RBS = NA
lib_seqs$RBS[which(lib_seqs$CDS.num %in% c(1, 5, 9, 13:22))] <- "WT"
lib_seqs$RBS[which(lib_seqs$CDS.num %in% c(2, 6, 10, 23:32))] <- "BB0030"
lib_seqs$RBS[which(lib_seqs$CDS.num %in% c(3, 7, 11, 33:42))] <- "BB0032"
lib_seqs$RBS[which(lib_seqs$CDS.num %in% c(4, 8, 12, 43:52))] <- "BB0034"
lib_seqs$RBS <- factor(lib_seqs$RBS, rev(c("BB0030", "BB0034", "BB0032", 
    "WT")))

# split leader into CDS types (WT, min/max rare codons, secondary
# structure)
lib_seqs$CDS.type <- (as.integer(lib_seqs$CDS.num) - 3)%%10
lib_seqs$CDS.type[which(lib_seqs$CDS.num %in% c(1:4))] <- "WT"
lib_seqs$CDS.type[which(lib_seqs$CDS.num %in% c(5:8))] <- "Min Rare"
lib_seqs$CDS.type[which(lib_seqs$CDS.num %in% c(9:12))] <- "Max Rare"
lib_seqs$CDS.type <- factor(lib_seqs$CDS.type, rev(c("WT", "Min Rare", 
    "Max Rare", 0:9)), rev(c("WT", "Min Rare", "Max Rare", paste("∆G", c(1:10), 
    sep = " "))))

# function to get the length of any factored string field and add RBS
# length to each sequence, since this varies, unlike promoter and CDS
get_len <- function(df, field) {
    seq_field <- paste(field, "seq", sep = ".")
    len_field <- paste(field, "len", sep = ".")
    df[, len_field] <- nchar(as.character(df[1, seq_field]))
    return(df)
}
require(plyr)
lib_seqs <- ddply(lib_seqs, .(RBS.seq), get_len, "RBS")
```




## Checking `lib_seqs` DataFrame for completeness

First we should check that all the identities are equally distributed:



```r
# Promoter
table(lib_seqs$Promoter)
```



```
## 
## BBaJ23100 BBaJ23108 
##      7124      7124 
```



```r
# RBS Type
table(lib_seqs$RBS)
```



```
## 
##     WT BB0032 BB0034 BB0030 
##   3562   3562   3562   3562 
```



```r
# CDS Type
table(lib_seqs$CDS.type)
```



```
## 
##    ∆G 10     ∆G 9     ∆G 8     ∆G 7     ∆G 6     ∆G 5     ∆G 4     ∆G 3 
##     1096     1096     1096     1096     1096     1096     1096     1096 
##     ∆G 2     ∆G 1 Max Rare Min Rare       WT 
##     1096     1096     1096     1096     1096 
```



```r
# Gene
table(lib_seqs$Gene)
```



```
## 
## accA accD acpP acpS alaS asnS aspS bamA bamD  can coaE csrA cysS dapA dapB 
##  104  104  104  104  104  104  104  104  104  104  104  104  104  104  104 
## dapD dapE  der dnaE dnaX  dxr  dxs  eno  era erpA fabA fabB fabD fabG fabI 
##  104  104  104  104  104  104  104  104  104  104  104  104  104  104  104 
## fbaA  ffh fldA folA folC folD folE folK ftsA ftsB ftsI ftsL ftsQ ftsW ftsZ 
##  104  104  104  104  104  104  104  104  104  104  104  104  104  104  104 
## gapA gltX grpE gyrA hemA hemB hemH hemL holA holB ileS infA ispA ispD ispE 
##  104  104  104  104  104  104  104  104  104  104  104  104  104  104  104 
## ispF ispG ispH kdsA kdsB lepB leuS  lgt ligA  lnt lolA lolB lolC lolD lolE 
##  104  104  104  104  104  104  104  104  104  104  104  104  104  104  104 
## lptD lpxB lpxC lpxD lpxK  map metG metK mnmA mraY mrdA mrdB msbA mukB mukE 
##  104  104  104  104  104  104  104  104  104  104  104  104  104  104  104 
## mukF murC murD murF murG murJ nadD nadE nadK nrdA nrdB  pgk pgsA pheS pheT 
##  104  104  104  104  104  104  104  104  104  104  104  104  104  104  104 
## plsC prfA prfB prmC proS pssA pyrG pyrH ribA ribC ribE ribF  rne rplS rpsA 
##  104  104  104  104  104  104  104  104  104  104  104  104  104  104  104 
## rpsB rseP secA serS suhB thiL thrS tilS  tmk topA trmD  tsf tyrS yeaZ yejM 
##  104  104  104  104  104  104  104  104  104  104  104  104  104  104  104 
## yqgF zipA 
##  104  104 
```




Now we can see why there are differences in the RBS lengths:



```r
ggplot(lib_seqs, aes(x = RBS, y = RBS.len)) + geom_point()
```

![plot of chunk 1.04-plot_rbs_len](figure/1.04-plot_rbs_len.png) 


WT sequences are all 20 bp, and the three designed RBSs are 18, 19, and 21 bp each. Good to know.









