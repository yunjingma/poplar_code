# run genomic data on hpc ugent

## transfer data from psb server to ugent hpc DATA folder

1.login vampire and cd to folder of that file
' scp -r pod5_pass vsc46851@login.hpc.ugent.be:/data/gent/vo/001/gvo00194/vsc46851
vsc46851@login.hpc.ugent.be:/scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/barcode1_phasing '

## Transfer dorado models from vampire to Ugent hpc

yunma@vampire:/scratch/tmp$ scp rerio.tar.gz vsc46851@login.hpc.ugent.be:/data/gent/vo/001/gvo00194/vsc46851/
rerio.tar.gz                                                                                                    100%  222MB  80.5MB/s   00:02
yunma@vampire:/scratch/tmp$ scp dorado_plant.sh vsc46851@login.hpc.ugent.be:/data/gent/vo/001/gvo00194/vsc46851/
dorado_plant.sh   

# step 1:base calling of POD5 data with dorado that downloaded in my DATA folder, model with hac
#!/bin/bash
#Basic parameters
#PBS -N jobname           ## Job name
#PBS -l nodes=1:ppn=2     ## 1 node, 2 processors per node (ppn=all to get a full node)
#PBS -l walltime=72:00:00 ## Max time your job will run (no more than 72:00:00)

#Situational parameters: remove one '#' at the front to use
#PBS -l gpus=1            ## GPU amount (only on accelgor or joltik)
#PBS -l mem=32gb          ## If not used, memory will be available proportional to the max amount
#PBS -m abe               ## Email notifications (abe=aborted, begin and end)

cd /scratch/gent/vo/001/gvo00194/vsc46851
DIR=/data/gent/vo/001/gvo00194/vsc46851
$VSC_DATA_VO_USER/dorado-0.5.0-linux-x64/bin/dorado basecaller -r --device cuda:all $VSC_DATA_VO_USER/dna_r10.4.1_e8.2_400bps_hac@v4.3.0 ${DIR}/pod5_pass/barcode01/ > /scratch/gent/vo/001/gvo00194/vsc46851/dorado/barcode1_calls.bam

***Notes***
I runned barcode by barcode, it took 2h to finish one barcode


# step 2:transform .bam file to .fasta with samtools
#version of SAMtools/1.18-GCC-12.3.0 on cluster doduo
#memory used 13.38M
#!/bin/bash
#Basic parameters
#PBS -N bam to fasta           ## Job name
#PBS -l nodes=1:ppn=2     ## 1 node, 2 processors per node (ppn=all to get a full node)
#PBS -l walltime=72:00:00 ## Max time your job will run (no more than 72:00:00)
#PBS -l mem=4gb          ## If not used, memory will be available proportional to the max amount
#PBS -m abe               ## Email notifications (abe=aborted, begin and end)

cd /scratch/gent/vo/001/gvo00194/vsc46851
module load SAMtools
for i in {1..10}; do
    samtools fasta dorado/barcode${i}_calls.bam > dorado/barcode${i}_calls.fasta
done

# step 3:use Porechop to trim adapter
#!/bin/bash
#Basic parameters
#PBS -N trim adapter           ## Job name
#PBS -l nodes=1:ppn=12     ## 1 node, 2 processors per node (ppn=all to get a full node)
#PBS -l walltime=72:00:00 ## Max time your job will run (no more than 72:00:00)
#PBS -l mem=64gb          ## If not used, memory will be available proportional to the max amount
#PBS -m abe               ## Email notifications (abe=aborted, begin and end)
#Porechop/0.2.4-intel-2019b-Python-3.7.4

cd /scratch/gent/vo/001/gvo00194/vsc46851
module load Porechop
for i in {6..10}; do
    porechop -i dorado/barcode${i}_calls.fasta -o dorado/barcode${i}_porechop.fasta --threads 12
done
 ***NOTES***
 I trimmed 10 together, while after 5, the job was cancelled because of hitting the walltime, need to make sure that the fifth 5 is finished properly
 #rerunned the first 5 and with 12 cores, finished in 11 hours
 
 # step 4:run  flye
#!/bin/bash
#Basic parameters
#PBS -N flye_barcode1           ## Job name
#PBS -l nodes=4:ppn=2     ## 1 node, 2 processors per node (ppn=all to get a full node)
#PBS -l walltime=72:00:00 ## Max time your job will run (no more than 72:00:00)
#PBS -l mem=64gb          ## If not used, memory will be available proportional to the max amount
#PBS -m abe               ## Email notifications (abe=aborted, begin and end)

cd /scratch/gent/vo/001/gvo00194/vsc46851

module load Flye/2.9.2-GCC-11.3.0
#flye --asm-coverage 40 --genome-size 450M --debug --threads 8 --iterations 4 --nano-hq /scratch/gent/vo/001/gvo00194/vsc46851/dorado/barcode1_porechop.fasta --out-dir /scratch/gent/vo/001/gvo00194/vsc46851/assembly/
flye --genome-size 450M --debug --threads 8 --iterations 4 \
--nano-hq /scratch/poplargwas/barcode1/hp_i_1.fasta hp_nonseperated.fasta\
--out-dir /scratch/poplargwas/barcode1/barcode1_haplotype1.fasta

# step5:call 5mc and 6ma 
#11 hours per barcode
#!/bin/bash
#Basic parameters
#PBS -N barcode3me          ## Job name
#PBS -l nodes=1:ppn=2     ## 1 node, 2 processors per node (ppn=all to get a full node)
#PBS -l walltime=72:00:00 ## Max time your job will run (no more than 72:00:00)
#PBS -l gpus=1            ## GPU amount (only on accelgor or joltik)
#PBS -l mem=32gb          ## If not used, memory will be available proportional to the max amount
#PBS -m abe               ## Email notifications (abe=aborted, begin and end)

cd /scratch/gent/vo/001/gvo00194/vsc46851
DIR=/data/gent/vo/001/gvo00194/vsc46851
$VSC_DATA_VO_USER/dorado-0.5.0-linux-x64/bin/dorado basecaller -r --device cuda:all \
--modified-bases-models /data/gent/vo/001/gvo00194/vsc46851/rerio/dorado_models/res_dna_r10.4.1_e8.2_400bps_sup@v4.0.1_5mC@v2,/data/gent/vo/001/gvo00194/vsc46851/rerio/dorado_models/res_dna_r10.4.1_e8.2_400bps_sup@v4.0.1_6mA@v2 \
$VSC_DATA_VO_USER/dna_r10.4.1_e8.2_400bps_hac@v4.3.0 ${DIR}/pod5_pass/barcode04/ > /scratch/gent/vo/001/gvo00194/vsc46851/dorado/barcode4me_calls.bam

# step 6:run minimap2 to allign with ref genome
#!/bin/bash
#Basic parameters
#PBS -N minimap2           ## Job name
#PBS -l nodes=1:ppn=12     ## 1 node, 2 processors per node (ppn=all to get a full node)
#PBS -l walltime=72:00:00 ## Max time your job will run (no more than 72:00:00)
#PBS -l mem=32gb          ## If not used, memory will be available proportional to the max amount
#PBS -m abe               ## Email notifications (abe=aborted, begin and end)
#minimap2/2.26-GCCcore-12.2.0

module load minimap2
minimap2 -ax map-ont /data/gent/vo/001/gvo00194/vsc46851/GCA_015852605.2_ASM1585260v2_genomic.fna \
/scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/barcode1_10_porechop.fasta/barcode1_porechop.fasta \
/scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_aln.sam

# index ref genome with samtools
samtools faidx /data/gent/vo/001/gvo00194/vsc46851/GCA_015852605.2_ASM1585260v2_genomic.fna

# index aln.bam file and sort with samtools
#!/bin/bash
#Basic parameters
#PBS -N longshot          ## Job name
#PBS -l nodes=1:ppn=12     ## 1 node, 2 processors per node (ppn=all to get a full node)
#PBS -l walltime=72:00:00 ## Max time your job will run (no more than 72:00:00)
#PBS -l mem=32gb          ## If not used, memory will be available proportional to the max amount
#PBS -m abe               ## Email notifications (abe=aborted, begin and end)
#Longshot/0.4.5-GCCcore-11.3.0
module load SAMtools
samtools view -Sb -o /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_aln.bam /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_aln.sam
samtools sort -O bam -o /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_sorted_aln.bam /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_aln.bam
samtools index /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_sorted_aln.bam

# sniffles to call structual variants
#!/bin/bash
#Basic parameters
#PBS -N longshot          ## Job name
#PBS -l nodes=1:ppn=12     ## 1 node, 2 processors per node (ppn=all to get a full node)
#PBS -l walltime=72:00:00 ## Max time your job will run (no more than 72:00:00)
#PBS -l mem=32gb          ## If not used, memory will be available proportional to the max amount
#PBS -m abe               ## Email notifications (abe=aborted, begin and end) 
module load Sniffles/2.0.7-GCC-11.3.0
sniffles --input /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1/barcode1_aln_sorted.bam --vcf /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1/barcode1_aln_sorted.vcf 


ragtag
ragtag.py scaffold /data/gent/vo/001/gvo00194/vsc46851/GCA_015852605.2_ASM1585260v2_genomic.fna /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1/barcode1_assembly.fasta -o /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1/barcode1_ragtag.agp 


# longshot to call vcf and sperate haplotype
cd /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1
mkdir longshot_barcode1_10
module purge GCCcore/12.2.0
module load GCC/11.3.0
module load Longshot/0.4.5-GCCcore-11.3.0
longshot -O /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1/longshot_barcode1_10/barcode1phased.bam --bam /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_sorted_aln.bam --ref /data/gent/vo/001/gvo00194/vsc46851/GCA_015852605.2_ASM1585260v2_genomic.fna --out /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/longshot_barcode1_10/barcode1.vcf

***Notes***
#after longshot, the sequence is already tagged with HP:i:1 or HP:i:2, can be directly used to seperate
#this line of code could check the bam file 
samtools view /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1/out_barcode1phased.bam | grep "HP:i:"



module purge GCCcore/12.2.0
module load GCC/11.3.0
need to use the same version of GCCcore



module load SAMtools
samtools view -Sb -o /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_aln.bam /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_aln.sam
samtools sort -O bam -o /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_sorted_aln.bam /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_aln.bam
samtools index /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_sorted_aln.bam

cd /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1
mkdir longshot_barcode1_10
module purge GCCcore/12.2.0
module load GCC/11.3.0
module load Longshot/0.4.5-GCCcore-11.3.0
longshot -O /scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/longshot_barcode1_10/barcode1phased.bam --bam 
/scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/minimap_barcode1_10/barcode1_sorted_aln.bam --ref 
/data/gent/vo/001/gvo00194/vsc46851/GCA_015852605.2_ASM1585260v2_genomic.fna --out 
/scratch/gent/vo/001/gvo00194/vsc46851/barcode1_10_output/longshot_barcode1_10/barcode1phased.vcf 

# use perl from stephane to sperate haplotype
#!/bin/bash
#Basic parameters
#PBS -N phasing          ## Job name
#PBS -l nodes=1:ppn=12     ## 1 node, 2 processors per node (ppn=all to get a full node)
#PBS -l walltime=72:00:00 ## Max time your job will run (no more than 72:00:00)
#PBS -l mem=32gb          ## If not used, memory will be available proportional to the max amount
#PBS -m abe               ## Email notifications (abe=aborted, begin and end)
module load SAMtools
samtools fasta -T HP /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1/out_barcode1phased.bam > /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1/barcode1_phased_tagged.fasta


#!/bin/bash
# Basic parameters
#PBS -N phasing          ## Job name
#PBS -l nodes=1:ppn=12     ## 1 node, 2 processors per node (ppn=all to get a full node)
#PBS -l walltime=72:00:00 ## Max time your job will run (no more than 72:00:00)
#PBS -l mem=32gb          ## If not used, memory will be available proportional to the max amount
#PBS -m abe               ## Email notifications (abe=aborted, begin and end)

module load perl
perl /scratch/gent/vo/001/gvo00194/vsc46851/code_barcode1_10/Mfasta_tools2.pl /scratch/gent/vo/001/gvo00194/vsc46851/assembly/barcode1/barcode1_aln_sorted.fasta -SplitBC=HP

***notes***
could not work with hpc ugent, since it can not generate 3 files that belongs to hap 1, hap2 and non sperated hap
so I have to transfer data to vampire 





