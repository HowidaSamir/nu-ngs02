Crash variant calling
=====================

Download Fastq files
```
mkdir -p ~/workdir/fqData && cd ~/workdir/fqData
wget https://de.cyverse.org/dl/d/3CE425D7-ECDE-46B8-AB7F-FAF07048AD42/samples.tar.gz
tar xvzf samples.tar.gz
```


Download reference file
```
cd ~/workdir/sample_data
wget https://de.cyverse.org/dl/d/A9330898-FC54-42A5-B205-B1B2DC0E91AE/dog_chr5.fa.gz
gunzip dog_chr5.fa.gz
```



BWA Alignment
=============

## install [bwa](http://bio-bwa.sourceforge.net/bwa.shtml)
```
source activate ngs1
conda install -c bioconda bwa 
```

## index your genome

```
mkdir -p ~/workdir/bwa_align/bwaIndex && cd ~/workdir/bwa_align/bwaIndex
ln -s ~/workdir/sample_data/dog_chr5.fa .
bwa index -a bwtsw dog_chr5.fa
```

## sequence alignment

```
cd ~/workdir/bwa_align
R1="$HOME/workdir/fqData/BD143_TGACCA_L005_R1_001.pe.fq.gz"
R2="$HOME/workdir/fqData/BD143_TGACCA_L005_R2_001.pe.fq.gz"
/usr/bin/time -v bwa mem bwaIndex/dog_chr5.fa $R1 $R2 > BD143_TGACCA_L005.sam
```


Visualize mapping
=================

## install [samtools](http://www.htslib.org/doc/samtools.html)
```
source activate ngs1
conda install samtools
```

## Index your alignment file
```
# Convert the SAM file into a BAM file that can be sorted and indexed:
samtools view -hbo BD143_TGACCA_L005.bam BD143_TGACCA_L005.sam

# Sort the BAM file by position in genome:
samtools sort BD143_TGACCA_L005.bam -o BD143_TGACCA_L005.sorted.bam

# Index the BAM file so that we can randomly access it quickly:
samtools index BD143_TGACCA_L005.sorted.bam
```

## Visualize mapping using [tview](http://samtools.sourceforge.net/tview.shtml) in samtools
```
samtools tview -p chr5:62155107 BD143_TGACCA_L005.sorted.bam bwaIndex/dog_chr5.fa
```
- Understanding [Pileup format](https://en.wikipedia.org/wiki/Pileup_format) should help explaining the symbols in tview
- In the viewer: 
    * press `?` for help
    * `q` to quit
    * CTRL-h and CTRL-l do “big” scrolls
    * g chr5:62155107 will take you to a specific location.


## Visualize mapping using [IGV](https://bioinformatics-ca.github.io/resources/IGV_Tutorial.pdf)

```
cd ~
wget http://data.broadinstitute.org/igv/projects/downloads/2.5/IGV_Linux_2.5.0.zip
unzip IGV_Linux_2.5.0.zip
sudo echo 'export IGV=$HOME/IGV_Linux_2.5.0/igv.sh' >> ~/.bashrc
source ~/.bashrc
source activate ngs1
bash $IGV -g bwaIndex/dog_chr5.fa BD143_TGACCA_L005.sorted.bam
```

Call variants
=============

mpileup: 
- Generate genotype likelihoods for one or multiple alignment files. 
- Individuals are identified from the SM tags in the @RG header lines. Multiple individuals can be pooled in one alignment file, also one individual can be separated into multiple files. If sample identifiers are absent, each input file is regarded as one sample.
- Base alignment quality (BAQ). BAQ is the Phred-scaled probability of a read base being misaligned. Applying this option greatly helps to reduce false SNPs caused by misalignments.
- `-Ou` option for piping between bcftools subcommands to speed up performance by removing unnecessary compression/decompression and VCF←→BCF conversion.

Call: 
- SNP/indel calling 
- `-m` multiallelic-caller
- `-v` output variant sites only

## Install [BCFTools](http://www.htslib.org/doc/bcftools.html)
```
conda install bcftools
bcftools mpileup -Ou -f bwaIndex/dog_chr5.fa BD143_TGACCA_L005.sorted.bam |\
bcftools call -Ov -mv > BD143_TGACCA_L005.vcf
```
