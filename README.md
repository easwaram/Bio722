# Bio722
## RNA-Seq DEG analysis Project

### 1. FastQC to check read quality
### 2. Trimmomatic or Cutadapt or FastP for trimming
### 3. FastQC to check quality post-trimming
### 4. Salmon or HISAT2 for pseudo-mapping/splice-aware mapping
### 5. QoRTs for checking quality of mapping
### 6. HTSeq for counting (salmon already counts)
### 7. Use Tximport/matrix to create matrices from count data (input for DEG analysis)
   - Check mapping quality, stats, etc... using PCA and clustering plots
### 8. DESeq2.
- Output of DEG as tables, charts, diagrams, etc.


## Raw Data:
### Input: fastq files from NCBI SRA Database
### Obtain these fastq files from SRA database using accession list:

    for sra_id in $(cat SRR_Acc_List_gut.txt); do
    
   ### To download a specific SRA run
    /usr/local/sratoolkit/bin/prefetch.3.0.10 $sra_id
    
  ### Convert SRA to FASTQ
    /usr/local/sratoolkit/bin/fasterq-dump.3.0.10 --split-files $sra_id --outdir SRA_raw_fastq_gut
    
    done

### Repeat the above for gill accession list:
      for sra_id in $(cat SRR_Acc_List_gill.txt); do          	/usr/local/sratoolkit/bin/prefetch.3.0.10 $sra_id;          
	    /usr/local/sratoolkit/bin/fasterq-dump.3.0.10 --split-files $sra_id --outdir SRA_raw_fastq_gill;      

    done


### Rename these fastq files to something easier (according to R1 and R2 of each sample)
### ***they have now been renamed and can be found in directory "SRA_all_raw_fastq" such that:
### {sra_id}_{tissue}_{control#}_{read#}.fastq


## Step 1: fastqc will unzip the file for you (if in .fastq.gz but we didn't have to since already in .fastq)

### If running fastqc on multiple fastq files in a directory
    fastqc *.fastq -o YourOutputDirect/ 
### can specify the number of threads for large processes using '-t n'where n is the number of processesors


    fastqc SRA_all_raw_fastq/*.fastq -o fastqc_all_raw_reads

## Step 2: trimmomatic of raw reads from Input: "SRA_all_raw_reads"
### Adaptors are from TruSeq3-PE-2.fa and NEB kit (used in this experiment for library prep); this file is called "TruSeq3-PE-2_NEB.fa"

### This was the simplified bash script that worked!- this can be found in "trimmer.sh"
    #!/bin/bash

    for infile in /home/grdstd6/SRA_all_raw_fastq/*_1.fastq
    do
      base=$(basename ${infile} _1.fastq)
      java -jar /opt/local/trimmomatic/Trimmomatic-0.39/trimmomatic-0.39.jar PE -threads 16 \
    /home/grdstd6/SRA_all_raw_fastq/${base}_1.fastq /home/grdstd6/SRA_all_raw_fastq/${base}_2.fastq \
    /home/grdstd6/trimmed_reads/${base}_R1_PE.fastq /home/grdstd6/trimmed_reads/${base}_R1_SE.fastq  /home/grdstd6/trimmed_reads/${base}_R2_PE.fastq /home/grdstd6/trimmed_reads/${base}_R2_SE.fastq   ILLUMINACLIP:/home/grdstd6/TruSeq3-PE-2_NEB.fa:2:30:10 LEADING:3 TRAILING:3 MAXINFO:40:0.4 MINLEN:50

     done
### set up a loop to run the paired input for you

    for file in ${files[@]}
    do
    raw_dir=/home/grdstd6/SRA_all_raw_fastq
    trim_dir=/home/grdstd6/trimmed_reads
    files=(${raw_dir}/*_1.fastq)
      name=${file}
      base=$(basename ${name} _1.fastq)
      java -jar /opt/local/trimmomatic/Trimmomatic-0.39/trimmomatic-0.39.jar PE -threads 16    
    ${raw_dir}/${base}_1.fastq ${raw_dir}/${base}_2.fastq  ${trim_dir}/${base}_R1_PE.fastq ${trim_dir}/${base}_R1_SE.fastq  ${trim_dir}/${base}_R2_PE.fastq ${trim_dir}/${base}_R2_SE.fastq   ILLUMINACLIP:/home/grdstd6/TruSeq3-PE-2_NEB.fa:2:30:10 LEADING:3 TRAILING:3 MAXINFO:40:0.4 MINLEN:50
     done




## Step 3: Fastqc of trimmed paired end fastq
	/opt/local/fastqc/fastqc_v0.11.3/fastqc trimmed_reads/*_PE.fastq -o fastqc_trimmed_paired_reads

### fastqc post-trim looks good- eg: gill C3_1 had an overrepresented sequence of Truseq adaptor but this is now gone.


## Step 4: HISAT2 for mapping uses paired end reads from trimmomatic
### Download reference genome for Danio rerio (GRCz11) from NCBI using command line tool "datasets" and "dataformat"
	datasets download genome accession GCF_000002035.6

### first use hisat2-build to index the genome
	/usr/local/hisat/hisat2-build -p 6 -f ncbi_dataset/data/GCF_000002035.6/GCF_000002035.6_GRCz11_genomic.fna hisat2_index/drerio_index_GRCz11

### then use hisat2 to map the reads using the index; bash script is called "mapping.sh"
	 drerio_index_GRCz11

#!/bin/bash

    for infile in {sample_dir}/*_R1_PE.fastq
    do
    index_dir=/home/grdstd6/hisat2_index/drerio_index_GRCz11
    sample_dir=/home/grdstd6/trimmed_reads
    out_dir=/home/grdstd6/hisat_mapped_reads
      base=$(basename ${infile} _R1_PE.fastq)

    /usr/local/hisat/hisat2 -f -x ${index_dir} \
      -1 ${sample_dir}/${base}_R1_PE.fastq \
      -2 ${sample_dir}/${base}_R2_PE.fastq \
      -p 8 -S ${out_dir}/${base}_mapped

    done

### Output: "hisat_summary_alignment" contains the summary of the hisat2 alignment


### convert ouput SAM files to BAM files using SAMtools
### Directly sorting the SAM file- this bash script is stored in "samtobam.sh"
    #!/bin/bash

    for infile in /home/grdstd6/hisat_mapped_reads/*_mapped.sam
    do
      base=$(basename ${infile} _mapped.sam)
    /usr/local/samtools/current/samtools view -@8 -Sb /home/grdstd6/hisat_mapped_reads/${base}_mapped.sam | /usr/local/samtools/current/samtools sort -@8 -O bam -o /home/grdstd6/bam_hisat_mapped/${base}_sorted.bam

    done

###########################################################################################################
## Step 5: QoRTS

### The below is stored in the "qorts.sh" script

    #!/bin/bash

    for infile in /home/grdstd6/SRA_all_raw_fastq/*_1.fastq
    do
    qc_dir=QoRTs_QC_mapped_reads
    bam_file=/home/grdstd6/bam_hisat_mapped/*_sorted.bam
    raw_dir=/home/grdstd6/SRA_all_raw_fastq
    mapped_dir=/home/grdstd6/bam_hisat_mapped
    base=$(basename ${infile} _1.fastq)
    mkdir ${qc_dir}/${base}
    java -jar -Xmx8G /usr/local/qorts/QoRTs.jar QC \
                       --generatePlots \
                       --genomeFA ncbi_dataset/data/GCF_000002035.6/GCF_000002035.6_GRCz11_genomic.fna \
                       --rawfastq ${raw_dir}/${base}_1.fastq,${raw_dir}/${base}_2.fastq \
                       ${mapped_dir}/${base}_sorted.bam \
                       /home/grdstd6/danio_rerio_GRCz11_genomic.gtf \
                       ${qc_dir}/${base}/

    done


### The above keeps stating an error in the gtf file format. Not sure why this is and I have tried changing the format of the tf file- still no luck

### Will try RSeQC instead:


### First need to convert sorted bam files to indexed files using samtools for use of tin.py in RSeQC

    for infile in /home/grdstd6/bam_hisat_mapped/*_sorted.bam
    do
      base=$(basename ${infile} _sorted.bam)
    /usr/local/samtools/current/samtools index -@10 /home/grdstd6/bam_hisat_mapped/*_sorted.sam --output /home/grdstd6/bam_hisat_mapped/bam_sorted_indexed/${base}_sorted_indexed.bam.bai

    done

### Nevermind, couldn't specify output when input is more than 1 file so did 6 files individually- stored in bam_sorted_indexed

    samtools index SRR12614073_gut_C2_sorted.bam  SRR12614073_gut_C2_sorted_indexed.bam.bai

### Used this to get the bed file off of UCSC- directed from NCBI to here
     wget --timestamping 'ftp://hgdownload.soe.ucsc.edu/goldenPath/danRer11/bigZips/danRer11.trf.bed.gz' -O danRer11.bed.gz

### Now I can use RSeQC

### danio_rerio_GRCz11.bed is the BED12 file from UCSC using Tools -> Table Browser

### getting the TIN number
    /usr/local/python2.7/Python-2.7.12/python /usr/local/RSeQC/RSeQC-2.6.4/scripts/tin.py -i /home/grdstd6/bam_hisat_mapped/bam_sorted_indexed/ -r /home/grdstd6/danio_rerio_GRCz11.bed

### Some simple statistics (comparable to what is available in samtools)
    /usr/local/python2.7/Python-2.7.12/python /usr/local/RSeQC/scripts/bam_stat.py -i Pairend_nonStrandSpecific_36mer_Human_hg19.bam > BAM_stats.txt

### Gene body coverage
    /usr/local/python2.7/Python-2.7.12/python /usr/local/RSeQC/scripts/geneBody_coverage.py -i Pairend_nonStrandSpecific_36mer_Human_hg19.bam -r hg19_RefSeq.bed -o output

### Read distribution
    /usr/local/python2.7/Python-2.7.12/python /usr/local/RSeQC/scripts/read_distribution.py  -i Pairend_nonStrandSpecific_36mer_Human_hg19.bam -r hg19_RefSeq.bed > read_distribution_out

### splice junction annotation
    /usr/local/python2.7/Python-2.7.12/python /usr/local/RSeQC/scripts/junction_annotation.py  -i Pairend_nonStrandSpecific_36mer_Human_hg19.bam -r hg19_RefSeq.bed -o junction_out


    /usr/local-centos6/python2.7/Python-2.7.12/python /usr/local-centos6/RSeQC/RSeQC-2.6.4/scripts/geneBody_coverage.py -i /home/grdstd6/bam_hisat_mapped/bam_sorted_indexed/ -r /home/grdstd6/danio_rerio_GRCz11.bed -o output

### Still getting a permission denied error even though using the same script in Ian's repo.- still not sure why.


## Step 6: HTSeq counting of reads

    #!/bin/bash

    python -m HTSeq.scripts.count -f bam -r pos /home/grdstd6/bam_hisat_mapped/*_sorted.bam /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/genomic.gtf


### first sort bam file by name and then rerun below code
    htseq-count -f bam -r pos /home/grdstd6/bam_hisat_mapped/*_sorted.bam /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/genomic.gtf

### By adding "-n" this sorts by name instead of chromsomal position: The below can be found in htseq.sh

    #!/bin/bash

    for infile in /home/grdstd6/hisat_mapped_reads/*_mapped.sam
    do
      base=$(basename ${infile} _mapped.sam)
    /usr/local/samtools/current/samtools view -@8 -Sb /home/grdstd6/hisat_mapped_reads/${base}_mapped.sam | /usr/local/samtools/current/samtools sort -@8 -O bam -n -o /home/grdstd6/bam_hisat_name_sorted/${base}_name_sorted.bam

    done

### Now lets do htseq-count again but using a for loop since the output does not get output into a file unless you specify! The -n flag to specify number of processors didn't work so had to remove it.
### The below can be found in htseq.sh

    #!/bin/bash

    for infile in /home/grdstd6/bam_hisat_name_sorted/*_name_sorted.bam
    do
      base=$(basename ${infile} _name_sorted.bam)
    htseq-count -f bam -r name ${infile} /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/genomic.gtf -o /home/grdstd6/htseq_counts/${base}_htseq_counts.txt

    done

### Will retry this since the output files look weird (should have 1 output file per hisat aligment file with counts in 2 column table)

    htseq-count -f bam -r name SRR12614064_gut_C3_name_sorted.bam /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/genomic.gtf > SRR12614064_gut_C3_htseq_counts.txt

### The above looks good! Now time to do it individually for each bam file:

    htseq-count -f bam -r name SRR12614073_gut_C2_name_sorted.bam /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/genomic.gtf > SRR12614073_gut_C2_htseq_counts.txt

    htseq-count -f bam -r name SRR12614074_gut_C1_name_sorted.bam /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/genomic.gtf > SRR12614074_gut_C1_htseq_counts.txt

### Round 2

    screen htseq1
    htseq-count -f bam -r name SRR12876838_gill_C3_name_sorted.bam /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/genomic.gtf > SRR12876838_gill_C3_htseq_counts.txt

    screen htseq2
    htseq-count -f bam -r name SRR12876847_gill_C2_name_sorted.bam /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/genomic.gtf > SRR12876847_gill_C2_htseq_counts.txt

    screen htseq3
    htseq-count -f bam -r name SRR12876848_gill_C1_name_sorted.bam /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/genomic.gtf > SRR12876848_gill_C1_htseq_counts.txt



## Step 4: Redo alignment/mapping using salmon- better than STAR for well annotated genomes
### First need to make salmon decoy.txt file (containing the sequence ids) and use this to run salmon indexing.
 
    grep "^>" <(gunzip -c GCF_000002035.6_GRCz11_genomic.fna.gz) | cut -d " " -f 1 > decoys.txt
    sed -i.bak -e 's/>//g' decoys.txt

### Now we make the reference file containing both transciptome and genome sequences
    cat /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/rna.fna /home/grdstd6/ncbi_dataset/data/GCF_000002035.6/GCF_000002035.6_GRCz11_genomic.fna > GRCz11_salmon_reference.fa

### Use transcript file and decoys.txt file for creating salmon index
    salmon index -t GRCz11_salmon_reference.fa -d decoys.txt -p 12 -i salmon_index_GRCz11


    ### Now can generate counts usig the trimmed paired end files. The below is stored in the script "salmon_quant.sh"

    #!/bin/bash

    for infile in /home/grdstd6/trimmed_reads/*_R1_PE.fastq
    do
    index_dir=/home/grdstd6/salmon_index_GRCz11
    sample_dir=/home/grdstd6/trimmed_reads
    out_dir=/home/grdstd6/salmon_counts
    base=$(basename ${infile} _R1_PE.fastq)

    salmon quant -i ${index_dir} -l A \
      -1 ${sample_dir}/${base}_R1_PE.fastq \
      -2 ${sample_dir}/${base}_R2_PE.fastq \
      -p 16 --validateMappings --rangeFactorizationBins 4 \
      --seqBias --gcBias \
      -o ${out_dir}/${base}_quant

    done

### Most likely library type (-l A) was determined as IU
### can use --writeMappings argument for getting mapping information in SAM like file output - I have % mapping rate information from the log

###########################################################################################################
## Step 7: Use tximport for salmon

    if (!requireNamespace("BiocManager", quietly = TRUE))
      install.packages("BiocManager")

    BiocManager::install("DESeq2")
    BiocManager::install("edgeR")
    BiocManager::install("limma")
    BiocManager::install("tximport")
    BiocManager::install("tximportData") # this is relatively big
    BiocManager::install("tximeta")
    BiocManager::install("GenomeInfoDb")
    BiocManager::install("org.Dr.eg.db")

    avail <- BiocManager::available() # available packages
    avail[grep("Drerio", avail)]

### we want the TxDb package downloaded- not the BSgenome
    BiocManager::install("TxDb.Drerio.UCSC.danRer11.refGene") # danio rerio

### Few other libraries we need
    library(readr)
    library("RColorBrewer")
    library(gplots)

    setwd("/Users/melli/OneDrive/Desktop/GRCz11_salmon_quant")
    getwd()
    dir()

### Names of samples.
    samples <- c("SRR12614064_gut_C3",
                 "SRR12614073_gut_C2",
                 "SRR12614074_gut_C1",
                 "SRR12876838_gill_C3",
                 "SRR12876847_gill_C2",
                 "SRR12876848_gill_C1")

    print(samples)

    quant_files <- file.path( dir(), "quant.sf")
    quant_files


    file.exists(quant_files)

    library("TxDb.Drerio.UCSC.danRer11.refGene")
    txdb <- TxDb.Drerio.UCSC.danRer11.refGene # easier to write
    txdb # Some basic information about the transcriptome
    transcripts(txdb) # a list of all of the transcripts along with genome ranges.
    transcriptsBy(txdb, "gene") # grouped by gene

### Now we can extract just the transcript and gene identifiers and create the file we need
    k <- keys(txdb, keytype = "TXNAME")

    tx2gene <- select(x = txdb, keys = k, "GENEID", "TXNAME")

    head(tx2gene)

    library(tximport)
    txi <- tximport(quant_files, 
                    type = "salmon", tx2gene = tx2gene, ignoreTxVersion = TRUE)

### Had to add ignoreTxVersion to get rid of Error in .local(object, ...) : 
### None of the transcripts in the quantification files are present in the first column of tx2gene. Check to see that you are using the same annotation for both.
### Example IDs (file): [NM_001001398.2, NM_001001399.2, NM_001001400.1, ...]
### Example IDs (tx2gene): [NM_198371, NM_001017766, NM_001017577, ...]

### It now works (tXDB version after the period was removed so these now match)
    summary(txi)
    head(txi$counts)
    str(txi)

    cor_vals <- cor(txi$counts[,1:6]) 
    hist(cor_vals)

    pairs(log2(txi$counts[,1:6]), 
          pch = 20, lower.panel = NULL, col = rgb(0,0,1, 0.5))

    pairs(log2(txi$counts[,1:2]), 
          pch = 20, lower.panel = NULL, col = rgb(0,0,1, 0.25))


    library(DESeq2)

    tissue.design <- data.frame(
      row.names=samples,
      background=c(rep("gut", 3), rep("gill", 3))
    )
    print(tissue.design)


    ddsTxi <- DESeqDataSetFromTximport(txi, colData = tissue.design, design = ~background)

### This makes PCA for conditions (gut vs gill)
    for_pca <- rlog(ddsTxi, 
                    blind = TRUE)

    dim(for_pca)
    plotPCA(for_pca, 
            intgroup=c("background"),
            ntop = 2000)


## Step 8: DESEq2 on tximport salmon counts
### Let's test out DESeq2 on the entire imported dataset first
    ddsTxi2 <- DESeq(ddsTxi)

    plotDispEsts(ddsTxi2,
                 legend = F)

    txi.res <- results(ddsTxi2)
    txi.res
    resultsNames(ddsTxi2)
    summary(txi)


    plotMA(filtered.txi.deseq, ylim =c(-5, 5))

### Shrinking the estimates
    library(apeglm)

    txi.res <- lfcShrink(ddsTxi2, coef="background_gut_vs_gill", type="apeglm")
    txi.res

    summary(txi.res)
### This is the older version "normal" that Ian used
    txi.res.shrunk <- lfcShrink(ddsTxi2, 
                                coef = 2,
                                type = "normal",
                                lfcThreshold = 0)

    plotMA(txi.res, ylim =c(-5, 5))


### Now lets filter for our chemical defensome genes only before we run a GO analysis
    defense.genes.list.salmon <- readLines("C:/Users/melli/OneDrive/Desktop/ZFIN_genes/Complete lists/complete_ncbi_gene_ids")
    print(defense.genes.list.salmon)


    filtered.txi.deseq <- txi.res[defense.genes.list.salmon,]

    head(filtered.txi.deseq)

    filtered.txi.deseq <- na.omit(filtered.txi.deseq)

### Now readjust the adjust pvalues using the raw pvalues of the subset of genes you're analyzing downstream
    filtered.txi.deseq$padj <- p.adjust(filtered.txi.deseq$pvalue, method="BH")

    summary(filtered.txi.deseq)
### Now can make figures using the "filtered.txi.deseq" file.


    str(ddsTxi2)

### default alpha is p<0.1 but can change alpha as below
    test_results <- results(ddsTxi2, 
                            alpha = 0.05)
    summary(test_results)
    head(test_results)


### top DEG genes for comparison
    top.genes <- head(filtered.txi.deseq[order(filtered.txi.deseq$padj),], 10)
    top.genes
    write.csv(top.genes, file = "txi_top_genes.csv")


    plotCounts(ddsTxi2, 
               gene = which.min(condition_results$padj),
               intgroup="condition")

### plot counts for ONLY abcg2c gene
    plotCounts(ddsTxi2, 
               gene = "569858",
               intgroup="background",
               pch = 6, col = "red")

### plot counts for ONLY cyp3c1 gene
    plotCounts(ddsTxi2, 
               gene = "324340",
               intgroup="condition",
               pch = 6, col = "red")

### For making a heatmap
    library("pheatmap")
    select <- order(rowMeans(counts(ddsTxi2,normalized=TRUE)),
                    decreasing=TRUE)[1:6000]
    df <- as.data.frame(colData(ddsTxi2)[("background")])
    ntd <- normTransform(ddsTxi2)

    pheatmap(assay(ntd)[select,], cluster_rows=FALSE, show_rownames=FALSE,
             cluster_cols=FALSE, annotation_col=df)

### GO analysis

    BiocManager::install("clusterProfiler")
    BiocManager::install("AnnotationDbi")
    BiocManager::install("org.Dr.eg.db")
    install.packages("ggupset")

    library(tidyverse)
    library(clusterProfiler)
    library(org.Dr.eg.db)
    library(AnnotationDbi)
    library(DOSE)
    library(enrichplot)
    library(ggupset)

    txi.sigs <- filtered.txi.deseq[filtered.txi.deseq$padj < 0.05 & filtered.txi.deseq$baseMean > 50,]

    genes_to_test <- rownames(txi.sigs[txi.sigs$log2FoldChange > 0.5,])
    GO_results <- enrichGO(gene = genes_to_test, OrgDb = "org.Dr.eg.db", keyType = "ENTREZID", ont = "BP")
    as.data.frame(GO_results)

    fit <- plot(barplot(GO_results, showCategory = 20))
    png("out.png", res = 250, width = 1000, height = 1200)
    print(fit)
    dev.off()

# list of only up-regulated/down-regulated genes for venn diagram
    txi.up <- filtered.txi.deseq[which(filtered.txi.deseq$log2FoldChange > 0 & filtered.txi.deseq$padj < 0.1),]
    txi.down <-filtered.txi.deseq[which(filtered.txi.deseq$log2FoldChange < 0 & filtered.txi.deseq$padj < 0.1),]
    rownames(txi.up)
    write.csv(rownames(txi.down), file = "txi_down_genes.csv")

###########################################################################################################
## Step 7: Import HTSeq- count data

    setwd("/Users/melli/OneDrive/Desktop/htseq_counts")

    samples <- c("SRR12614064_gut_C3",
                 "SRR12614073_gut_C2",
                 "SRR12614074_gut_C1",
                 "SRR12876838_gill_C3",
                 "SRR12876847_gill_C2",
                 "SRR12876848_gill_C1")

### A function to read one of the count files produced by HTSeq
    read.sample <- function(sample.name){
      file.name <- paste(sample.name, "_htseq_counts.txt", sep="")
      result <- read.delim(file.name, col.names=c("gene", "count"), sep="\t", colClasses=c("character", "numeric"))
    }

### Read the first sample
    sample.1 <- read.sample(samples[1])

    head(sample.1)
    nrow(sample.1)

### Read the second sample
    sample.2 <- read.sample(samples[2])

### Let's make sure the first and second samples have the same number of rows and the same genes in each row
    nrow(sample.1) == nrow(sample.2)
    all(sample.1$gene == sample.2$gene)

### Now let's combine them all into one dataset
    all.data <- sample.1
    all.data <- cbind(sample.1, sample.2$count)
    for (c in 3:length(samples)) {
      temp.data <- read.sample(samples[c])
      all.data <- cbind(all.data, temp.data$count)
    }

### We now have a data frame with all the data in it:
    head(all.data)

    colnames(all.data)[2:ncol(all.data)] <- samples

### Now look:
    head(all.data)

### Let's look at the bottom of the data table
    tail(all.data)

### Let's remove the last 5 rows since they are summaries
    all.data <- all.data[1:(nrow(all.data)-5),]

    tail(all.data)

    library("DESeq2")

### Remove the first column since all data needs to be counts in the table
    raw.deseq.data <- all.data[,2:ncol(all.data)]

### Set row names to the gene names
    rownames(raw.deseq.data) <- all.data$gene

    head(raw.deseq.data)
    tail(raw.deseq.data)

### Create metadata for the sample information
    tissue.design <- data.frame(
      row.names=samples,
      background=c(rep("gut", 3), rep("gill", 3))
    )
### Double check it...
    tissue.design

### Check to make sure these are the same:
    all(rownames(tissue.design) == colnames(raw.deseq.data))

### Now lets make a DeSeq2 object
    ddsMat <- DESeqDataSetFromMatrix(countData = raw.deseq.data,
                                     colData = tissue.design,
                                     design= ~background)
    ddsMat


### This makes PCA for conditions (gut vs gill)
    for_pca <- rlog(ddsMat, 
                blind = TRUE)

    dim(for_pca)
    plotPCA(for_pca, 
            intgroup=c("background"),
            ntop = 2000)


## Step 8: DESeq2 on HTSeq count matrix

### Let's test out DESeq2 on the entire imported dataset first
    ddsMat2 <- DESeq(ddsMat)

      plotDispEsts(ddsMat2,
                 legend = F)

    mat.res <- results(ddsMat2)
    mat.res
    resultsNames(ddsMat2)

    plotMA(mat.res, ylim =c(-5, 5))

### Shrinking the estimates
    library(apeglm)

    mat.res <- lfcShrink(ddsMat2, coef="background_gut_vs_gill", type="apeglm")
    mat.res
    summary(mat.res)
    plotMA(mat.res, ylim =c(-5, 5))


### Now lets filter for our chemical defensome genes only before we run a GO analysis
    defense.genes.list.htseq <- readLines("C:/Users/melli/OneDrive/Desktop/Gene names only- Chemical defensome.txt")

    filtered.mat.deseq <- mat.res[defense.genes.list.htseq,]
    head(filtered.mat.deseq)

### Need to remove the NA rows from the raw data file:
    filtered.mat.deseq <- na.omit(filtered.mat.deseq)

    head(filtered.mat.deseq)


### Now readjust the adjust pvalues using the raw pvalues of the subset of genes you're analyzing downstream
    filtered.mat.deseq$padj <- p.adjust(filtered.mat.deseq$pvalue, method="BH")

    summary(filtered.mat.deseq)
### Now can make figures using the "filtered.txi.deseq" file.

### top DEG genes for comparison
    top.genes.mat <- head(filtered.mat.deseq[order(filtered.mat.deseq$padj),], 10)
    top.genes.mat
    write.csv(top.genes.mat, file = "mat_top_genes.csv")


    install.packages("pheatmap")
    library("pheatmap")
    select <- order(rowMeans(counts(filtered.dds,normalized=TRUE)),
                decreasing=TRUE)[1:300]
    df <- as.data.frame(colData(filtered.dds)[("background")])
    ntd <- normTransform(filtered.dds)

    pheatmap(assay(ntd)[select,], cluster_rows=FALSE, show_rownames=FALSE,
         cluster_cols=FALSE, annotation_col=df)

    plotCounts(filtered.dds, 
               gene = "slc22a2",
               intgroup="background",
               pch = 6, col = "red")

### GO analysis

    BiocManager::install("clusterProfiler")
    BiocManager::install("AnnotationDbi")
    BiocManager::install("org.Dr.eg.db")
    install.packages("ggupset")

    library(tidyverse)
    library(clusterProfiler)
    library(org.Dr.eg.db)
    library(AnnotationDbi)
    library(DOSE)
    library(enrichplot)
    library(ggupset)

    mat.sigs <- filtered.mat.deseq[filtered.mat.deseq$padj < 0.05 & filtered.mat.deseq$baseMean > 50,]

    genes_to_test <- rownames(mat.sigs[mat.sigs$log2FoldChange > 0.5,])
    GO_results <- enrichGO(gene = genes_to_test, OrgDb = "org.Dr.eg.db", keyType = "SYMBOL", ont = "BP")
    as.data.frame(GO_results)

    fit <- plot(barplot(GO_results, showCategory = 20))
    png("out.png", res = 250, width = 1000, height = 1000)
    print(fit)
    dev.off()


### list of only up-regulated/down-regulated genes for venn diagram
    mat.up <- filtered.mat.deseq[which(filtered.mat.deseq$log2FoldChange > 0 & filtered.mat.deseq$padj < 0.1),]
    mat.down <-filtered.mat.deseq[which(filtered.mat.deseq$log2FoldChange < 0 & filtered.mat.deseq$padj < 0.1),]
    rownames(mat.up)
    write.csv(rownames(mat.up), file = "mat_up_genes.csv")

