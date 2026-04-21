---
title: "Palidis: A C++ Algorithm for Discovering Insertion Sequences in Metagenomic Data"
date: 2023-10-10
tags: ["c++", "algorithms", "bioinformatics", "nextflow", "open-source"]
summary: "How I built a maximal exact matching algorithm in C++ with two-bit encoding to discover novel mobile genetic elements from metagenomic sequencing data and how it found applications from antimicrobial resistance surveillance to gene therapy manufacturing."
---

{{< katex >}}

Insertion sequences are among the simplest and most abundant mobile genetic elements in bacterial genomes. They're short, typically 700 to 2,500 base pairs, and contain little more than the genes needed for their own transposition. But their simplicity belies their importance: insertion sequences are a primary vehicle for the horizontal transfer of antimicrobial resistance genes between bacteria, and they're dramatically underrepresented in existing reference databases.

During my PhD I built [Palidis](https://github.com/blue-moon22/palidis4) (Palindromic Detection of Insertion Sequences) — a bioinformatics pipeline that discovers novel insertion sequences directly from metagenomic sequencing data without relying on reference databases. At its core is a C++ algorithm called [pal-MEM](https://github.com/blue-moon22/pal-MEM) that I designed for efficiently finding inverted terminal repeats in large genomic datasets. The work was published in [Microbial Genomics](https://doi.org/10.1099/mgen.0.000917) and later found an unexpected second life in gene therapy manufacturing.

## The Problem

Insertion sequences are flanked by inverted terminal repeats (ITRs), short DNA sequences of 10–50 base pairs at each end that are reverse complements of each other. These ITRs are the binding sites for transposases, the enzymes that mediate the element's movement between genomic locations. Detecting ITRs is therefore the key to finding insertion sequences.

But there are several reasons why existing approaches struggle:

**Reference databases are incomplete.** The main database, ISfinder, catalogues known insertion sequences, but it's small and biased toward well-studied organisms. You can't find what isn't in the database.

**Assembly algorithms break on repeats.** Short-read assemblers struggle to resolve repeated elements. They may collapse them, include only one copy, or omit them entirely. An insertion sequence that's misassembled or incomplete won't be found by tools that work on assembled genomes.

**Exact palindrome detection is too strict.** ITR pairs are not always perfect reverse complements. Tools like EMBOSS that search for exact palindromes miss many real insertion sequences that have accumulated mutations since their insertion.

**Brute-force search is too slow.** Metagenomic datasets contain millions of reads, each ~100 base pairs. Checking every pair of reads for inverted repeat relationships is computationally prohibitive.

## The Algorithm: pal-MEM

The core challenge is finding pairs of reads that contain reverse-complementary subsequences, potential ITRs, across a dataset of millions of sequences. I needed an algorithm that was both fast enough for metagenomic-scale data and sensitive enough to find the biologically relevant matches.

pal-MEM (palindromic Maximal Exact Matching) is based on the E-MEM algorithm for computing maximal exact matches, but substantially modified for the specific problem of finding inverted repeats in short-read metagenomic data.

### Two-Bit Encoding

The first optimisation is at the representation level. DNA has a four-letter alphabet (A, C, G, T), which maps naturally to two bits per nucleotide:

| Nucleotide | Encoding |
|---|---|
| A | `00` |
| C | `01` |
| G | `10` |
| T | `11` |

This means a 15-mer (the default k-mer length) occupies just 30 bits, half a 64-bit integer. An entire 100 bp read fits in four 64-bit integers. The encoding reduces memory by 4x compared to character-based storage. More importantly, it makes k-mer comparison a single bitwise operation rather than a character-by-character string comparison.

The algorithm stores all reads as a continuous array of unsigned 64-bit integers, with each integer holding 32 nucleotides. Random 20-bit separator sequences mark read boundaries. A secondary data structure tracks the start and end positions of each read within this continuous array.

### Hash Table for k-mer Lookup

The algorithm builds a hash table from the reference sequences (which, for inverted repeat detection, are the reverse complements of the input reads). k-mers are the keys; their positions in the reference are the values.

A key insight from E-MEM is that not every k-mer needs to be stored. For a minimum match length *L* and k-mer size *k*, a k-mer only needs to be indexed if its position *p* satisfies:

$$p \equiv 0 \pmod{(L - k) + 1}$$

This guarantees that any MEM of length *L* will contain at least one indexed k-mer, while reducing the number of stored k-mers, and therefore memory usage, substantially.

The hash table uses open addressing with double hashing for collision resolution. Hash table sizes are pre-computed prime numbers chosen to maintain load factors that keep collision chains short.

### Extension with Interval Halving

When a k-mer match is found, the algorithm extends it in both directions to find the maximal exact match. Rather than extending one nucleotide at a time, pal-MEM uses an interval halving approach:

1. Extend by the maximum possible distance (to the boundary of the shortest sequence)
2. Compare the two extended regions using bitwise operations on the 64-bit integer representation
3. If they don't match, halve the extension distance and try again
4. Once a match is found, extend one nucleotide at a time until a mismatch is reached

This approach converges in *O(log n)* comparison steps rather than *O(n)*, with each comparison itself being a constant-time bitwise operation on 64-bit integers.

### Optimisations for Metagenomic Data

I made three modifications specific to the metagenomic use case:

**Reverse complement by default.** E-MEM searches for direct matches between sequences, with reverse complement matching as an option. Since we're specifically looking for inverted repeats (which are reverse complements), pal-MEM transforms the reference into reverse complements at the outset, converting the problem to direct matching.

**Early termination per read.** A short read (~100 bp) from Illumina sequencing will contain at most one ITR. Once a MEM is found within a read, pal-MEM skips to the next read rather than continuing to scan the remainder. This dramatically reduces the search space for large metagenomic libraries.

**Technical repeat filtering.** Sequencing libraries are dominated by technical reverse complements — overlapping fragments from both strands of a double-stranded DNA molecule. These produce MEMs at the prefix of one read and the suffix of another, mimicking biological inverted repeats. pal-MEM excludes MEMs whose start or end positions are within a buffer distance of the read boundaries, filtering out the most common class of false positives without an expensive separate deduplication step.

## The Palidis Pipeline

pal-MEM finds reads containing potential inverted repeats. Palidis wraps it in a Nextflow pipeline that goes from raw sequencing data to validated insertion sequences in five steps:

1. **Pre-process and find repeat sequences**: Convert FASTQ to FASTA, run pal-MEM to identify reads containing inverted terminal repeats.

2. **Map and filter by proximity**: Map the repeat-containing reads against assembled contigs using Bowtie2. A Python script identifies candidate ITRs where the mapped positions of the repeats fall within the expected distance range of an insertion sequence (500–3,000 bp by default).

3. **Cluster and validate**: Cluster candidate ITRs using CD-HIT-EST (sequence identity threshold, alignment coverage). Putative insertion sequences must have ITRs from the same cluster that are reverse complements of each other — confirmed by BLASTn alignment showing `Strand=Plus/Minus` with identity above the minimum ITR length.

4. **Annotate transposases**: Run InterProScan on predicted protein sequences from Prodigal to confirm the presence of transposase, integrase-like, or RNase H domains — the enzymatic machinery required for transposition.

5. **Generate outputs**: Produce a FASTA file of insertion sequences and a tab-delimited information file with coordinates, sample IDs, contig mappings, and protein family annotations.

The entire pipeline runs in a single Docker/Singularity container, with Nextflow handling parallelisation across samples and HPC job scheduling via nf-core institutional configs.

## Results

Applied to 264 human oral and gut metagenomes from the Human Microbiome Project, Palidis identified **2,517 insertion sequences** from 1,837 contigs across 218 samples. After clustering to remove redundancy, this produced an **Insertion Sequence Catalogue (ISC) of 879 unique insertion sequences**.

Of these:
- **519 (59%) were novel** — not found in ISfinder, the main reference database
- **360 (41%) matched** ISfinder entries, with 60 having strong homology (e-value < 1e-50) and 300 having loose homology
- The catalogue contained **87 unique transposases** across elements ranging from 524 to 2,999 bp

Querying the ISC against a database of 661,405 bacterial genomes revealed evidence of **horizontal gene transfer across bacterial classes**: the same insertion sequences appearing in genomes from different genera, sometimes spanning 21 genera and 46 species. Several genera (Bacteroides, Corynebacterium, Prevotella) had significant numbers of insertion sequences not represented in ISfinder at all, highlighting the gap in existing databases.

## Impact Beyond AMR

The original motivation was antimicrobial resistance surveillance: understanding how resistance genes spread between bacteria via mobile genetic elements. My PhD work showed that AMR genes are commonly linked to insertion sequences, and that country-specific resistance profiles can be traced partly through the mobile element landscape.

But the tool found an unexpected second application.

**Gene therapy manufacturing** relies on adeno-associated virus (AAV) vectors, which use ITRs, the same structures Palidis detects, for viral DNA replication and packaging. These 145 bp ITR sequences form T-shaped hairpin structures that are essential for the vector to function. During plasmid production in bacteria, these complex ITR structures are unstable and prone to spontaneous deletions and mutations. Damaged ITRs severely reduce or eliminate vector effectiveness.

Palidis can verify ITR integrity in both plasmid DNA and finished AAV vectors by checking for a single dominant cluster of ITRs, indicating uniformity. This quality control application is documented in a [WIPO patent application](https://patentscope.wipo.int/), a use case I never anticipated when building a tool to find bacterial transposable elements.
