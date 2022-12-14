# Positive selection analysis using Hyphy

### Install the tools needed
```
conda install -c bioconda minimap2
conda install -c bioconda samtools
conda install -c bioconda bcftools
```

### 1. Extraction of sequences of a specific gene
- Downloading the genome sequence `fasta` and annotation `gff3` of the reference `NC_045512.2`

  ```
  wget -O - "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=nuccore&id=NC_045512.2&rettype=fasta" > NC_045512.2.fasta

  wget -O - "https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/009/858/895/GCF_009858895.2_ASM985889v3/GCF_009858895.2_ASM985889v3_genomic.gff.gz" | gunzip > NC_045512.2.gff
  ```

- Extract the `Spike gene` sequence from the reference genome
  ```
  samtools faidx NC_045512.2.fasta NC_045512.2:21563-25384 > NC_045512.2.spike.fasta
  ```

- Map the sequences to the Spike gene sequence then sort and index
  ```
  minimap2 -ax asm5 NC_045512.2.spike.fasta combined.fasta | samtools sort -o combined.spike.sorted.bam
  
  samtools index combined.spike.sorted.bam
  ```

- Convert the `bam` file which contains the sequences aligned to the `Spike gene` into a `fasta` format
  ```
  samtools fasta combined.spike.sorted.bam > combined.spike.fasta
  ```

### 2. Codon-aware Multiple Sequence Alignment
- Correct for frame-shift mutations
```
hyphy pre-msa.bf --reference NC_045512.2.spike.fasta --input combined.spike.fasta CPU=10
```

- Generate multiple sequence alignment of the spike protein sequence
```
muscle -align combined.spike.fasta_protein.fas -output combined.spike.fasta_protein.msa -threads 10
```

- Generate corrected aligned nucleotide sequence of the spike gene 
```
hyphy post-msa.bf --protein-msa combined.spike.fasta_protein.msa --nucleotide-sequences combined.spike.fasta_nuc.fas --output combined.fin.msa CPU=10
```

### 3. Phylogenetic tree construction: Maximum Likelihood method
- Construct tree using maximum likelihood with `GTR` substitution model, `+I+G` invariable site plus discrete Gamma model, and `1000 bootstrapping`
```
iqtree2 -s combined.fin.msa -m GTR+I+G -T AUTO -B 1000
```

### 4. Selection analysis
- Test using `FUBAR` (Fast, Unconstrained Bayesian AppRoximation)
```
hyphy fubar --alignment combined.fin.msa --tree combined.fin.msa.treefile CPU=10
```

### 5. Viewing of results using [Hyphy vision](http://vision.hyphy.org/FUBAR)
- Upload/load the `combined.fin.msa.FUBAR.json` file
