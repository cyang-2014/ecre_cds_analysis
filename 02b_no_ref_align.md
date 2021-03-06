<link href="http://kevinburke.bitbucket.org/markdowncss/markdown.css" rel="stylesheet"></link>



# Step 02B - Aligning RNA Directly to DNA without a Reference Library

>I ended up scrapping this style of analysis in favor of doing the alignments against the reference library. After looking at the intersection of DNA, RNA and Protein that matched all criteria, we ended up gaining too few reads this way, and I was concerned that some of the sequences we gained would be error-prone or poor-quality data. 

## Motivation

In [Step 02](02_load_dna_rna.html), we aligned all the DNA and RNA reads directly to a reference library. One of the take home messages from that effort was that there are a lot of mismatching reads, and a lot of them are likely due to synthesis, and not sequencing. 

We are going to compute properties of each construct based on its nucleotide and DNA sequence, and so we are more intrested here in library diversity and not the identities of the sequences themselves. For instance, we don't care so much that the CDS sequence consists precisely the same amino acids as the front of the BamA gene, so much as that it has X secondary structure, Y codon adaptation index, etc. 

I'm going to (quickly) check to see if it is feasible to see if we can proceed without aligning both the RNA and DNA directly to the reference library. Perhaps instead we can first pull out all DNA with sufficient reads, and then subsequently align the RNA to those new 'reference' reads.

### Caveats to this approach

We can't use all of these sequences for a few reasons:

1. We must limited ourselves to sequences with perfect promoters and RBS. This hasn't removed too many reads, because the mismatch distribution histograms from Step 02 showed that not too many of the mismatches occur before base 40. 
2. The CDS need to be in frame, from the first ATG onward. This is easy enough to check for after splitting at the ATG. I'm not sure whether I should throw away or keep shorter CDSs; it will probably depend on how many there are. 
3. Dealing with multiple potential start sites might be problematic. For instance, if a mutation creates an extra ATG somewhere else close to the Shine-Dalrarno, it would complicate the amino acid analysis. To ensure that the start site does not change, I should remove sequences that create new ATGs close to the RBS in different frames, and make sure that none of the codons in the CDS were mutated to ATG. 
4. I also want to throw away any sequences with stop codons in them. 
5. As Sri and I discussed earlier, we are hiding any errors that are not part of the variable region, like the fluorescent coding sequences on the backbone.


### How do we choose a DNA read count cutoff, if we don't align to references?

In order to observe low RNA:DNA ratios we need to limit ourselves to DNA constructs with many DNA reads. To be sure we are looking at a synthesis and not a sequencing error in the RNA, we should see at least 5 reads in both replicates. 

Looking at the `plot_rna_dna.rna_ratio.tech_replicate.pdf` from the 202 analysis, it looks like a good ratio cutoff where most reads fall is about 2 `log10(-1.3)`, or if we have 5 read, then approximately 100 DNA reads per construct. If we don't see at least 5, then we can say the ratio is below 1:20. 

This is pretty much what we did on the 202 analysis, except in that case, almost all of the reads with low DNA occurred because they were high expressing constructs, causing impaired cell growth. This meant that for most of those sequences, the RNA would also be high, so we would not run into this problem often. 

### How many reference constructs have at least 100 reads per replicate?

We can of course see how many constructs that are in the `dna.subset` have over 100 reads:



```r
# Number of contigs matching perfectly with >=100 reads in both
# replicates:
sum(with(dna.subset, Mismatches.len == 0 & Count.A >= 100 & Count.B >= 
    100))
```



```
## [1] 13922
```



```r

# Number of constructs in library:
dim(lib_seqs)[1]
```



```
## [1] 14234
```



```r

# Designed sequences lost by this approach:
dim(lib_seqs)[1] - sum(with(dna.subset, Mismatches.len == 0 & Count.A >= 
    100 & Count.B >= 100)) - length(dna.missing)
```



```
## [1] 297
```




So we'd lose almost 300 of the 14,234 sequences due to low DNA counts. 

### How many new constructs might we gain?

How many would we gain? Let's take a look at the raw counts file. Under this scheme, we want to discard any reads with Ns, since we won't be able to assign them easily. We could default them to the designed reference construct, but that would be a pain. We also are setting size restrictions between 70 and 100 bases. 

```bash
lib_prefix=/scratch/dbg/ecre/fa/203.norestrict
out_prefix=/scratch/dbg/ecre/203_hs
dna_prefix=/scratch/dbg/ecre/ct/203_hsdna/203_hsdna.counts
rna_prefix=/scratch/dbg/ecre/ct/203_hsrna/203_hsrna.counts
    
#Count unique contigs with at least 100 reads in both replicates
less -S $dna_prefix.fa | perl -pe 'chomp; s/>/\n>/; 
        s/^([ATGC])/\t$1/;' \
    | perl -ne '@l = split; !(/N/) && $l[2] >= 100 && $l[3] >= 100 
        && length($l[4]) < 100 && length($l[4]) > 70 && print $_;' \
    | sort -nrk2 | cut -f2 | uniq -c \
    | cut -f1 | perl -nle '$sum += $_ } END { print $sum'

#Output: 16972

#Count unique contigs with at least 50 reads in both replicates
less -S $dna_prefix.fa | perl -pe 'chomp; s/>/\n>/; 
        s/^([ATGC])/\t$1/;' \
    | perl -ne '@l = split; !(/N/) && $l[2] >= 50 && $l[3] >= 50 
        && length($l[4]) < 100 && length($l[4]) > 70 && print $_;' \
    | sort -nrk2 | cut -f2 | uniq -c \
    | cut -f1 | perl -nle '$sum += $_ } END { print $sum'

#Output: 27154
```

So we end up with 16972 sequences with 100+ DNA reads. That's an extra 2,738 reads. If we lower our cutoff to 50 (which are still likely synthesis errors, but we'd have a smaller RNA ratio dynamic range), then that number goes up to 27154, or double our library size. 

### Checking the Protein Sequences

As Sri pointed out, the bottleneck here will really be Protein contigs, since there are fewer cells sorted than anything else. I should really see how many unique protein contigs there are. I can check that similarly:

```bash

prot_prefix=/scratch/dbg/ecre/ct/203_hiseq/203_hiseq.counts

#Count unique contigs with at least 200 reads
less -S $prot_prefix.fa | perl -pe 'chomp; s/>/\n>/; 
        s/^([ATGC])/\t$1/;' \
    | perl -ne '@l = split; !(/N/) && $l[1] >= 200
        && length($l[14]) < 100 && length($l[14]) > 70 && print $_;' \
    | sort -nrk2 | cut -f2 | uniq -c \
    | cut -f1 | perl -nle '$sum += $_ } END { print $sum'

#Output: 165600
```

This number is about 10x what we expect. That seems like a little much. I guess we'll see what happens when we intersect the DNA, RNA, and protein data. 

### Performing final filtering for DNA & Protein using grep

I'm going to save the raw counts files locally for RNA, DNA, and Protein, after filtering them based on length, read count, etc. 

After further discussion with Sri, we don't expect to see the same extreme dynamic range with this set of RNA data because we are only using two promoters. Because of this, we can lower the amount of DNA that we require to 25 reads per replicate.

Contig Type | Min Len | Max Len | Min Read Count (Per replicate for DNA/RNA)
---|---|---|---
DNA|70|100|25
RNA|52|90|4
Protein|70|100|100

```bash
lib_prefix=/scratch/dbg/ecre/fa/203.norestrict
out_prefix=/scratch/dbg/ecre/203_hs
dna_prefix=/scratch/dbg/ecre/ct/203_hsdna/203_hsdna.counts
rna_prefix=/scratch/dbg/ecre/ct/203_hsrna/203_hsrna.counts
prot_prefix=/scratch/dbg/ecre/ct/203_hiseq/203_hiseq.counts

#Filter RNA/Prot/DNA on read length and read counts
less -S $dna_prefix.fa | perl -pe 'chomp; s/>/\n>/; 
        s/^([ATGC])/\t$1/;' \
    | perl -ne '@l = split; !(/N/) && $l[2] >= 25 && $l[3] >= 25
        && length($l[4]) < 100 && length($l[4]) > 70
        && s/\t([ATGC])/\n$1/ && print $_;' \
    > /scratch/dbg/ecre/203.dna.fa &

less -S $rna_prefix.fa | perl -pe 'chomp; s/>/\n>/; 
        s/^([ATGC])/\t$1/;' \
    | perl -ne '@l = split; !(/N/) && $l[2] >= 4 && $l[3] >= 4
        && length($l[4]) < 90 && length($l[4]) > 52
        && s/\t([ATGC])/\n$1/ && print $_;' \
    > /scratch/dbg/ecre/203.rna.fa &

less -S $prot_prefix.fa | perl -pe 'chomp; s/>/\n>/; 
        s/^([ATGC])/\t$1/;' \
    | perl -ne '@l = split; !(/N/) && $l[1] >= 100
        && length($l[14]) < 100 && length($l[14]) > 70
        && s/\t([ATGC])/\n$1/ && print $_;' \
    > /scratch/dbg/ecre/203.prot.fa &
```

I also want to filter these reads on perfect promoters. There are only two, so I can just grep for them quickly. 

```console
$ grep -Pic '(ctgacagctagctcagtcctaggtataatgctagcCACCG|ttgacggctagctcagtcctaggtacagtgctagcTTAAT)' \
    /scratch/dbg/ecre/203.dna.fa
44315

$ grep -Pic '(ctgacagctagctcagtcctaggtataatgctagcCACCG|ttgacggctagctcagtcctaggtacagtgctagcTTAAT)' \
    /scratch/dbg/ecre/203.prot.fa
91545
```

Let's filter on RBS also. I need to make a simple text file with all promoter and RBS combinations. I can do that with the `lib_seqs` object. 



```r
all_promo_rbs <- with(lib_seqs, unique(paste("^", Promoter.seq, RBS.seq, 
    sep = "")))
write(all_promo_rbs, file = paste(getwd(), "/data/all_promo_rbs.txt", 
    sep = ""))
length(all_promo_rbs)
```



```
## [1] 280
```




Let's look for DNA based on those promoters, and then filter the protein on that DNA that matches the promoters. Let's also throw away DNA that has fewer than 30 bases of AA sequence and 

Finally, we'll filter the DNA again for only keep DNA that shows up in the protein reads.

```bash 
#filter DNA on known promo/rbs combos
grep -B1 -if data/all_promo_rbs.txt data/203.dna.fa | grep -v '\--' \
    > data/203.dna.filtered.fa &
    
#split the read into 3 parts, filter for ATG start codon and 30 bases of CDS
perl -pe 's/([^ATGC])\n/$1\t/' data/203.dna.filtered.fa \
    | perl -ne 's/([ATGC]{40})([ATGC]{18,}?)(ATG[ATGC]{30})/$1\t$2\t$3/ 
        && length($3) % 3 == 0 && s/(\d)\t([ATGC])/$1\n$2/ 
        && print;' > data/203.dna.filtered2.fa

```

## Align Protein with Bowtie

Now we align the Protein to DNA using Bowtie. (grep would be slower). First we build the Bowtie index:

```bash
#build the bowtie index
/opt/bowtie/bowtie-build data/203.dna.filtered2.fa data/203.filtered2
```


The we run bowtie, searching for reads that match the DNA perfectly. This will also grab partial matches, but we can throw that out the same way we removed invalid DNA reads, with a perl regex. 

>Bowtie Settings:
    `-k 280`: report up to 280 alignments per contig (280 promo/rbs combos)
    `-v 0`: 0 mismatches per contig (no indels)
    `-l 10`: seed length of 10
    `-p 16`: use all 16 processors
    `-m 1`: throw away reads that do not match to one sequence only

```bash
#perform bowtie for Protein. Afterwards, remove from that output any protein
#reads that are not complete matches
perl -ne 's/([ATGC]{40})([ATGC]{18,}?)(ATG[ATGC]{30})/$1\t$2\t$3/ 

/opt/bowtie/bowtie -v 0 -l 10 -k 280 -p 2 -m 1 \
    --norc --best --strata --suppress 2,6 \
    --un data/203.prot.unmapped.fa -f data/203.filtered2 data/203.prot.fa \
    | perl -ne 's/([ATGC]{40})([ATGC]{18,}?)(ATG[ATGC]{30})/$1\t$2\t$3/ 
        && length($3) % 3 == 0 && print;' > data/203.prot.bowtie
## reads processed: 165600
## reads with at least one reported alignment: 18254 (11.02%)
## reads that failed to align: 145854 (88.08%)
## reads with alignments suppressed due to -m: 1492 (0.90%)
## Reported 18254 alignments to 1 output stream(s)

wc -l data/203.prot.bowtie
## 17083 (down from 18254 due to checking for partial aligns with perl pipe)

#now remove the reads from the DNA set that are not matched with protein
grep -A1 -Fw -f <(cut -f 15 data/203.prot.bowtie) data/203.dna.filtered2.fa \
    | grep -v '\--' > data/203.dna.filtered3.fa

wc -l data/203.dna.filtered3.fa 
##  32268 (16134 records)
```

## Align RNA with Bowtie

So now we have the protein and the DNA done; on to the RNA. We need to make a new bowtie index of the filtered DNA first. Again, we check the RNA output for unique alignments (with `-m 1`) and pipe the output to perl, but this time it is simply to separate the CDS from the RBS/Promoter region for easier  splitting. 

```bash
#build the bowtie index
/opt/bowtie/bowtie-build data/203.dna.filtered3.fa data/203.filtered3
        
#perform bowtie for RNA
/opt/bowtie/bowtie -v 0 -l 10 -k 280 -p 2 -m 1 \
    --norc --best --strata --suppress 2,6 \
    --un data/203.rna.unmapped.fa -f data/203.filtered3 \
    <(perl -pe 's/([^ATGC])\n/$1\t/' data/203.rna.fa \
    | perl -ne '@l = split; ($l[1] > 4
        && length($l[4]) < 90 && length($l[4]) > 52) 
        && (s/\t([ATGC])/\n$1/ && print);') \
    | perl -ne 's/([ATGC]{18,}?)(ATG[ATGC]{30})/$1\t$2/ 
        && print;' > data/203.rna.bowtie
## reads processed: 340721
## reads with at least one reported alignment: 61846 (18.15%)
## reads that failed to align: 275815 (80.95%)
## reads with alignments suppressed due to -m: 3060 (0.90%)
## Reported 61846 alignments to 1 output stream(s)

cut -f5 data/203.rna.bowtie | sort | uniq -c | wc -l
   14765
```

## Final Subset of RNA, DNA and Protein

We are left with **14,765** reads that pass all the criteria, which is close to the 14,234 that we started with. Finally, we remove the DNA and Protein reads that do not appear in the RNA to get a consistent set of 14,765 constructs.

```bash
#now remove the reads from the DNA set that are not matched with rna and 
#create a 'bowtie-like' output so the DNA/RNA/Protein records can all be
#processed similarly. 
grep -A1 -Fw -f <(cut -f 5 data/203.rna.bowtie) data/203.dna.filtered3.fa \
    | grep -v '\--' \
    | perl -pe 's/([^ATGC])\n/$1\t/ && s/>//' > data/203.dna.bowtie

wc -l data/203.dna.bowtie
##  14765
```

## Conclusions

After all of this, we only gained 822 new sequences, and we lost approximately 291 of the designed sequnces because they had too few reads.  I don't think there are enough new sequences to warrant the headache caused by the potential bad data introduced here. It is a pain to do all of this work and then go back to the previous style of analysis (with mismatches), but I think overall it will be easier to stick to the designed sequences only. 

