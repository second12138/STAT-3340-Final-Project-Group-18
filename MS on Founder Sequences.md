# MS on Founder Sequences

## Download the reference file for GRCh38

```bash
mkdir ref
mkdir ref/GCGh38
wget -O ref/GRCh38/GRCh38_full_analysis_set_plus_decoy_hla.fa  ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/technical/reference/GRCh38_reference_genome/GRCh38_full_analysis_set_plus_decoy_hla.fa
```

Extract the chromosome19 reference

```bash
cd ref/GRCh38
awk '$0 ~ "^>" { match($1, /^>([^:]+)/, id); filename=id[1]} {print >> filename".fa"}' GRCh38_full_analysis_set_plus_decoy_hla.fa
cp chr19.fa ..
cd ../..
```

**Note**: the reference file has a problem, the titl of the fasta does not correspond to the on in the vcf file, i.e., it is chr19 instead of 19. It has to be changed manually.

Download the vcf file

```bash
wget -O ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.vcf.gz ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/data_collections/1000_genomes_project/release/20190312_biallelic_SNV_and_INDEL/ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.vcf.gz
```

Remove multiallelic variants

```bash
bcftools view -v snps,indels -m2 -M2 -Oz -o ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.no-multi.vcf.gz ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.vcf.gz 
```

Extract sample names for 1000 and 100 samples

```bash
bcftools query -l ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.no-multi.vcf.gz >all_names.txt
mkdir vcf_100
mkdir vcf_1000
head -n 1000 all_names.txt > vcf_1000/sample_ids_19.txt
head -n 1100 all_names.txt | tail -n 100 > vcf_100/sample_ids_19.txt
```

Extract VCF samples

```bash
bcftools view -c1 -S vcf_1000/sample_ids_19.txt -Oz --threads 8 -o vcf_1000/ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.no-multi.vcf.gz ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.no-multi.vcf.gz
bcftools view -c1 -S vcf_100/sample_ids_19.txt -Oz --threads 8 -o vcf_100/ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.no-multi.vcf.gz ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.no-multi.vcf.gz
```

Build the indexes

```bash
tabix -p vcf vcf_1000/ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.no-multi.vcf
tabix -p vcf vcf_100/ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.no-multi.vcf
```

Extract the sequences using `bcftools consensus` 

```bash
mkdir fasta_100
s_ids_100=(`cat vcf_100/sample_ids_19.txt`)
for i in ${$s_1ds_100[@]}; do
	bcftools consensus -f ref/chr19.fa -H 1 -s $i -o fasta_100/$i.1.19.fa vcf_100/ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.no-multi.vcf;
done
mkdir fasta_1000
s_ids_1000=(`cat vcf_1000/sample_ids_19.txt`)
for i in ${$s_ids_1000[@]}; do
	bcftools consensus -f ref/chr19.fa -H 1 -s $i -o fasta_1000/$i.1.19.fa vcf_1000/ALL.chr19.shapeit2_integrated_snvindels_v2a_27022019.GRCh38.phased.no-multi.vcf;
done
```

**Note**: This has to be tested. I used another python script.

Concatenate the sequences

```bash
cd fasta_100
cat $(ls -t) > ../chr19_query.fa
cp ref/chr19.fa fasta_1000/
cd ../fasta_1000
cat $(ls -t) > ../chr19_ref.fa
```

**Note**: this concatenation doesn't replace the fasta header with the sample name.

## Preprocessing the founder sequences

Generating a fasta file from the founder sequences, removing the gap symbols "-"

```bash
cd founder
mkdir original
mv * original
founders=(`ls -t original`)
for i in ${founders[@]}; do sed 's/-//g' original/$i > $i; done
touch founders.fa
for i in ${founders[@]}; do echo ">$i" >>founders.fa; cat $i >>founders.fa; echo "" >> founders.fa; done
```

## Build PHONI

from the build folder of PHONI

```bash
no_thresholds <path to chr19_ref.fa> -f -t <n threads> -m -s
./test/src/build_phoni <path to chr19_ref.fa>
no_thresholds <path to founders.fa> -f -t <n threads> -m -s
./test/src/build_phoni <path to founders.fa>
```

## Prepare the query

from the main folder of PHONI

```bash
./splitpattern.py <path to query.fa> <path to query.fa>.dir
```

## Run the queries

from the main folder of PHONI

```bash
./build/test/src/phoni <path to chr19_ref.fa> -p <path to query.fa>
./build/test/src/phoni <path to founders.fa> -p <path to query.fa>
```

**Note:** between the two runs, you need to make a copy op `<path to query.fa.length>` and  `<path to query.fa.pointers>` because they will bee overwritten.