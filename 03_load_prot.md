<link href="http://kevinburke.bitbucket.org/markdowncss/markdown.css" rel="stylesheet"></link>



# Step 03 - Loading and Processing Protein Data

## Motivation

Now that I've decided to use the reference-aligned data, all I need to do before I can merge the data is to extract the unique protein reads from `data/203.prot.fa` that match perfectly to the DNA sequences in the library. I have similar code written already that I used for the non-reference aligned code.

It works in three steps. We could also use `grep`, but this is faster computationally, and I already have the code written.

0. Since there are only 280 promo/cds combos, and I have a file containing
them, I will filter on those first. 
1. We use bowtie to quickly align the Protein to the DNA, similar to how we did the RNA. In this case, we will only keep protein data that matches exactly. 
2. Because bowtie finds partial matches, we use a perl regexp to parse out the reads that are complete (by looking for a full length promoter, RBS, and CDS.)

## Running Bowtie

>Bowtie Settings:
    `-k 280`: report up to 280 alignments per contig (280 promo/rbs combos)
    `-v 0`: 0 mismatches per contig (no indels)
    `-l 10`: seed length of 10
    `-p 16`: use all 16 processors
    `-m 1`: throw away reads that do not match to one sequence only

```bash
#perform bowtie for Protein. Afterwards, remove from that output any protein
#reads that are not complete matches
#build the bowtie index
mkdir data/03_load_prot
/opt/bowtie/bowtie-build data/203.norestrict.fa data/03_load_prot/203

/opt/bowtie/bowtie -v 0 -l 10 -k 280 -p 2 -m 1 \
    --norc --best --strata --suppress 2,6 \
    --un data/203.prot.unmapped.fa -f data/03_load_prot/203 \
    <(gzcat data/203.prot.fa.gz | grep -B1 -if data/all_promo_rbs.txt \
        | grep -v '\--') \
    | perl -ne '/([ATGC]{40})([ATGC]{18,21})(ATG[ATGC]{30})/ 
        && print;' > data/203.prot.bowtie
## reads processed: 60188
## reads with at least one reported alignment: 15879 (26.38%)
## reads that failed to align: 44111 (73.29%)
## reads with alignments suppressed due to -m: 198 (0.33%)
## Reported 15879 alignments to 1 output stream(s)

wc -l data/203.prot.bowtie
## 14140 (down from 15879 due to checking for partial aligns with perl pipe)
```

Out of 14,234 possible sequences, we got 14,140 with protein reads. 94 constructs have no Protein reads.  Let's read them into R and take a look. The `load_raw_reads` function comes from step 02.



```r
col.names = c("Read.num", "Count", paste("Bin", c(1:12), sep = "."), 
    "Name", "Offset.L", "Read.seq", "Alts", "Mismatches")

prot.raw <- load_raw_reads(paste(getwd(), "data/203.prot.bowtie", 
    sep = "/"), is.rna = F, col.names = col.names)
```




## Conclusions

### Summary of Missing Data

We now have all the raw data. Before we start merging and calculating, we now can see how many constructs are missing from each data set. 

* `94` constructs missing from Protein:

* `17` constructs missing 
    from RNA
* `15` constructs missing 
    from DNA
* `15` constructs missing from both DNA and Protein 
* There is no overlap between missing RNA and DNA constructs. 
* **`111`** costructs in total that are missing RNA, DNA, or Protein. 

This was pretty straightforward. Onto the merging and score calculation.




