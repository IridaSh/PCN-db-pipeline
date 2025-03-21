# PCN-db Pipeline

A pipeline for analyzing plasmid copy numbers (PCN) in bacterial genomes.

## Table of Contents
1. [Overview](#overview)
2. [Requirements](#requirements)
3. [Setup](#setup)
4. [Running the Pipeline](#running-the-pipeline)
5. [Expected Output](#expected-output)
6. [Performance](#performance)

## Overview

This pipeline analyzes plasmid copy numbers (PCN) in bacterial genomes using:
- NCBI genome data
- SRA sequencing data
- Kallisto for transcript quantification
- Themisto for pseudoalignment
- Breseq for mutation analysis

## Requirements

### Software
- Python 3.x
- Biopython
- Kallisto
- SRA-Toolkit
- pysradb
- ncbi-datasets-cli
- Breseq
- Themisto

### Hardware
- Recommended: Duke Compute Cluster (DCC)
- Storage: ~15TB for raw sequencing data
- Memory: 16GB minimum
- Time: ~2 weeks for full pipeline

## Setup

1. Create project structure:
   ```bash
   mkdir -p {data,results,src}
   ```

2. Download required data:
   ```bash
   wget -O data/prokaryotes.txt https://ftp.ncbi.nlm.nih.gov/genomes/GENOME_REPORTS/prokaryotes.txt
   ```

3. Filter for complete genome and plasmids:
   ```bash
   (head -n 1 data/prokaryotes.txt && grep "plasmid" data/prokaryotes.txt | grep "chromosome") > results/prokaryotes-with-chromosomes-and-plasmids.txt
   ```

4. Set up conda environment:
   ```bash
   conda create --name PCNdb_env --clone base
   conda activate PCNdb_env
   pip install pysradb biopython
   conda install -c bioconda kallisto breseq
   conda install -c conda-forge ncbi-datasets-cli
   ```

5. Using Docker (Recommended for Testing):
   ```bash
   # Build the Docker image
   docker build -t pcn-pipeline:latest .

   # Run the container with mounted volumes for data persistence
   docker run -v $(pwd)/data:/app/data \
             -v $(pwd)/results:/app/results \
             pcn-pipeline:latest

   # To run in test mode (default)
   # This will process a smaller subset of genomes
   docker run -v $(pwd)/data:/app/data \
             -v $(pwd)/results:/app/results \
             -e TEST_MODE=True \
             -e TEST_GENOME_COUNT=1000 \
             -e TEST_DOWNLOAD_LIMIT=50 \
             pcn-pipeline:latest

   # To run in production mode
   docker run -v $(pwd)/data:/app/data \
             -v $(pwd)/results:/app/results \
             -e TEST_MODE=False \
             pcn-pipeline:latest

   # To disable FASTQ compression
   docker run -v $(pwd)/data:/app/data \
             -v $(pwd)/results:/app/results \
             -e COMPRESS_FASTQ=False \
             pcn-pipeline:latest
   ```

   Notes:
   - The `-v` flags create persistent volumes, so your data and results are saved even after the container stops
   - Environment variables can be combined as needed
   - Data will be stored in ./data and results in ./results on your host machine
   - First run may take longer as it downloads and processes reference data

6. Install SRA-Toolkit:
   - On DCC: `module load SRA-Toolkit`
   - Locally: Download from [SRA-Tools GitHub](https://github.com/ncbi/sra-tools)

## Running the Pipeline

1. Copy source code to `src/` directory
2. Run the pipeline:
   ```bash
   sbatch --mem=16G -t 430:00:00 -p youlab --wrap="python src/PCN_pipeline.py"
   ```

## Expected Output

The pipeline generates:
1. Reference genomes in `data/NCBI-reference-genomes/`
2. SRA data in `data/SRA/`
3. Analysis results:
   - `results/PIRA-PCN-estimates.csv`
   - `results/NCBI-replicon_lengths.csv`
   - `results/kallisto-replicon_copy_numbers.csv`
   - `results/themisto-replicon-read-counts.csv`

## Performance

| Stage | Time Estimate | Data Size |
|-------|---------------|-----------|
| SRA Data Download | ~12 days | ~15TB |
| Reference Genome Download | Varies | ~50GB |
| Full Pipeline | ~2 weeks | ~15TB |

Note: Run in `/work` directory on DCC due to large storage requirements.
