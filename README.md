# MEeR-seq
Pipeline for mapping the 5' ends of rRNA intermediates as a proxy for assessing ribosome biogenesis.

## Removing Adapters
Adapters were removed, and reads were filtered via cutadapt/4.2.

```
cutadapt --match-read-wildcards --times 3 -e 0.1 -O 1 \
--nextseq-trim=10 -m 36 -a AGATCGGAAGAGCACACGTC -a AGATCGGAAGAGC -a G{10} \
-A AGATCGGAAGAGCGTCGTGT -A CCGCCTACACGAC -A CTACACGACGCTC -A ACGACGCTCTTCC -A G{10} -o $OUTPUT_R1 \
-p $OUTPUT_R2 $R1 $R2 > $METRICS_FILE
```

## Extracting the Unique Molecular Identifier (UMI)
UMI-tools/1.1.5 was used to extract the UMI from the sequence and place it in the header for downstream deduplication steps.

```
umi_tools extract --extract-method=string --bc-pattern=NNNNNNNNNNNNNN --stdin="$R1"  \
--read2-in="$R2" --stdout="$OUTPUT_R1" --read2-out="$OUTPUT_R2"
```

## Aligning Reads to a Reference Genome using Bowtie2
We first generated the index using a custom genome containing all rRNA fragments. For "$REFERENCE_GENOME" see the file: "rRNA_processing.fa"

```
bowtie2-build --threads 8 "$REFERENCE_GENOME" "$OUTPUT_DIR"
```

Then, we aligned to this reference genome using bowtie2/2.5.4

```
bowtie2 -p 8 -q --local -x "$REFERENCE_GENOME" -1 "$R1" -2 "$R2" -S "$OUTPUT_PREFIX" --met-file "$METRICS_FILE" --un-conc "${OUTPUT_PREFIX}_unmap.fastq"
```
followed by sorting and indexing using samtools/1.6

```
samtools view -bS "$FILE" | samtools sort -o "$OUTPUT_DIR/${SAMPLE_NAME}_sorted.bam" -
samtools index -@ 8 "$OUTPUT_DIR/${SAMPLE_NAME}_sorted.bam"
```

## Deduplicate Reads
We used UMI-tools to deduplicate samples after alignment. UMI-tools does this by removing reads aligned to the same genomic position with the same UMI in the read name.
```
umi_tools dedup -I "$BAM" --output-stats="$STATS_OUTPUT" --paired -S "$OUTPUT"
```
## Extract R1
Extracted R1 to map the true 5'end using samtools/1.6

```
samtools view -hb -f 66 "$FILE" -o "$OUTPUT_DIR/${SAMPLE_NAME}_R1.bam"
samtools index "$OUTPUT_DIR/${SAMPLE_NAME}_R1.bam"
```

## Convert to bigwig using deeptools/3.5.1

```
bamCoverage -b "$FILE" -o "$OUTPUT_DIR/${SAMPLE_NAME}.bw" --outFileFormat bigwig --normalizeUsing RPKM --binSize 1
```

## Extract RPKM values and plot
The final analysis was performed using a custom R script. See the file bigwig_analysis.Rmd for an example

Sample labeling: P_5_1: NAT10WT_replicate1_5'end, P_5_1: NAT10WT_replicate2_5'end, H_5_1: E389L_replicate1_5'end, H_5_2: E389L_replicate2_5'end
