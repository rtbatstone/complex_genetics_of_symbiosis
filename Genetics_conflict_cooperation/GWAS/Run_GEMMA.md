Running GEMMA, real and permutated data
================
Rebecca Batstone
2021-10-21

Setup
-----

Create the phenotype files (.fam) for GEMMA
-------------------------------------------

``` r
#################### for 89 strains #################### 

## load relevant df
ms <- read.delim("../Phenotypic_analyses/Data_input/multistrain_data.txt")

## add MAG to strain_ID
ms$add <- "MAG"
ms$strains <- str_c(ms$add, ms$Strain, sep = "", collapse = NULL)
ms$strains <- as.factor(ms$strains)

## select cols
ms89 <- ms %>%
  select(strains, contains("Fit"))

## look at distributions
### convert from wide to long format
ms89_long <- gather(ms89, key = "traits", value = "means",
         DZA_L_Fit_med, DZA_Lold_Fit_med, A17_L_Fit_med, A17_Lold_Fit_med)

## Density plots with semi-transparent fill
(density_emmeans <- ggplot(ms89_long, aes(x=means)) + 
  geom_density(alpha=.3) +
  facet_wrap(~ traits, ncol = 4, scales = "free") + 
  geom_vline(aes(xintercept=0), linetype="dashed", size=1) +
  theme_bw())
```

![](Run_GEMMA_files/figure-markdown_github/pheno_files-1.png)

``` r
## create .fam file:
ms89$V1 <- 0
ms89$V2 <- 0
ms89$V3 <- 0
ms89$strains2 <- ms89$strains

## select cols to include:
pheno89 <- ms89[,c("strains","strains2","V1","V2","V3",
                                   "DZA_L_Fit_med", "DZA_Lold_Fit_med", "A17_L_Fit_med", "A17_Lold_Fit_med")]

## save as .fam file
write.table(pheno89, file="./Data_output/phenos89.fam", 
            row.names = FALSE, col.names = FALSE, sep = "\t", quote = FALSE)
```

Filter vcf for GEMMA analyses
-----------------------------

Completed on remote server, rather than local machine:

``` bash
# first need to filter vcf (filter_vcf.sh)
### SETTINGS
MIN_MAF=0.05
MAX_MISSING=0.8 # Really means 20% max missingness
MIN_DP=20
MAX_DP=230
MAX_NON_REF=190

MEL_191_LIST="/path/to/strains"
GENOTYPES="/path/to/vcf"
VCFTOOLS="path/to/vcftools"

$VCFTOOLS --gzvcf $GENOTYPES --stdout --recode --keep $MEL_191_LIST | \
        $VCFTOOLS --vcf - --stdout --recode --max-missing $MAX_MISSING  \
        --maf $MIN_MAF --min-meanDP $MIN_DP --max-meanDP $MAX_DP --max-non-ref-ac-any $MAX_NON_REF > \
        "filtered.vcf" \
    || { echo "filtering vcf failed"; exit 1; }
```

Analyses on each dataset, "Complex\_genetics" (N = 191 strains) or "Genetics\_conflict\_cooperation" (N = 89 strains) were run within their own separate directories, to ensure output files were kept separate and were not overwritten. The code for both datasets is the exact same, except for the input files. I included the set of code relevant to the 89 strains (i.e., fitness data) for sake of clarity.

Completed on remote server, rather than local machine:

``` bash

#################### for 89 strains #################### 

# creates list of strain names to subset
cut -f 1 phenos89.fam > strains_89.txt 

$vcftools --gzvcf snps_all.vcf.bak --keep strains_89.txt --recode --out subset89
## 89 Individuals
## 816,444 sites

# filter sites based on LD groups:
cut -f 4 one_variant.tsv > keep_sites.txt

$vcftools --vcf subset89.recode.vcf --snps keep_sites.txt --recode --out subset89_LD
## 89 Individuals
## 8646 sites

# chrom

$vcftools --vcf subset89_LD.recode.vcf --chr chr --recode --out chr_subset89_LD
## 89 Individuals
## 757 sites

# psyma

$vcftools --vcf subset89_LD.recode.vcf --chr psyma --recode --out psyma_subset89_LD
## 89 Individuals
## 3277 sites

# psymb

$vcftools --vcf subset89_LD.recode.vcf --chr psymb --recode --out psymb_subset89_LD
## 89 Individuals
## 4612 sites
```

Convert vcf to bed for GEMMA
----------------------------

Completed on remote server, rather than local machine:

``` bash

#################### for 89 strains #################### 

# chrom

## all in one step (vcf to bed)
$PLINK --vcf chr_subset89_LD.recode.vcf --recode --make-bed --out "chr_subset89_LD_plink" \
--allow-extra-chr --allow-no-sex --maf 0.05 --geno --pheno "phenos89.fam"
## 650 vars
cp phenos89.fam chr_subset89_LD_plink.fam

# psyma

## all in one step (vcf to bed)
$PLINK --vcf psyma_subset89_LD.recode.vcf --recode --make-bed --out "psyma_subset89_LD_plink" \
--allow-extra-chr --allow-no-sex --maf 0.05 --geno --pheno "phenos89.fam"
## 2903 vars
cp phenos89.fam psyma_subset89_LD_plink.fam

# psymb

## all in one step (vcf to bed)
$PLINK --vcf psymb_subset89_LD.recode.vcf --recode --make-bed --out "psymb_subset89_LD_plink" \
--allow-extra-chr --allow-no-sex --maf 0.05 --geno --pheno "phenos89.fam"
## 4263 vars
cp phenos89.fam psymb_subset89_LD_plink.fam
```

Relatedness matrix
------------------

"-gk 2" calculates the standardized relatedness matrix

Note: need to have the .fam file for it to run: e.g., cp subset89\_LD\_plink.fam chr\_subset89\_LD\_plink.fam

Completed on remote server, rather than local machine:

``` bash

#################### for 89 strains ####################

$GEMMA -bfile chr_subset89_LD_plink -gk 2 -o chr_subset89_LD_ksmat ## 636 SNPs resulting
$GEMMA -bfile psyma_subset89_LD_plink -gk 2 -o psyma_subset89_LD_ksmat ## 2822 SNPs resulting
$GEMMA -bfile psymb_subset89_LD_plink -gk 2 -o psymb_subset89_LD_ksmat ## 4157 SNPs resulting
```

LMM option in GEMMA
-------------------

If you include multiple phenotypes in the same .fam file, you can use this loop -n <number> specifies column corresponding to phenotype in .fam file. -lmm option 4 includes all three tests (see GEMMA manual for more details)

Completed on remote server, rather than local machine:

``` bash

#################### for 89 strains ####################

# nano run_gemma.sh
for i in {1..4} # or however many numbers of columns (i.e., traits)
do 
  $GEMMA -bfile chr_subset89_LD_plink -k ./output/chr_subset89_LD_ksmat.sXX.txt \
  -lmm 4 -n $i -o 
chr_subset89_LD_ulmm_trait_$i
  echo "For chr, ran trait_$i through GEMMA"
  
  $GEMMA -bfile psyma_subset89_LD_plink -k ./output/psyma_subset89_LD_ksmat.sXX.txt \
  -lmm 4 -n $i -o psyma_subset89_LD_ulmm_trait_$i
  echo "For psyma, ran trait_$i through GEMMA"
  
   $GEMMA -bfile psymb_subset89_LD_plink -k ./output/psymb_subset89_LD_ksmat.sXX.txt \
   -lmm 4 -n $i -o psymb_subset89_LD_ulmm_trait_$i
  echo "For psymb, ran trait_$i through GEMMA"
done

# extract pve and se estimates
grep "pve estimate" run_gemma.out > pve_ulmm.txt
grep "se(pve)" run_gemma.out > se_ulmm.txt
paste pve_ulmm.txt se_ulmm.txt > pve_se_ulmm.txt
```

Combine GEMMA outputs (real runs)
---------------------------------

Completed on remote server, rather than local machine:

``` r
#################### for 89 strains ####################

## nano real_betas.R

require("dplyr")
require("purrr")

extract_fun <- function(files){
  traits <- read.table(files,header=T,sep="\t") # read in the file
    return(traits)
}  

files <- list.files(path ="./output/real", pattern=".assoc", full.names = T)
## file order needs to be checked, although should be correct if padding is used (01, 02)

traits <- lapply(files, extract_fun)

## extract positions and beta scores
### traits[c(1:4)] = chrom
### traits[c(5:8)] = psyma
### traits[c(9:12)] = psymb

# chrom
chr_realres <- traits[c(1:4)] %>% 
  reduce(full_join, by = "rs") %>%
  select("rs", starts_with("beta"))

names(chr_realres) <- c("rs","DZA_L_Fit_med", "DZA_Lold_Fit_med", "A17_L_Fit_med", "A17_Lold_Fit_med")

save(chr_realres, file="./chr_real_betas.Rdata")
                    
# psyma
psyma_realres <- traits[c(5:8)] %>% 
  reduce(full_join, by = "rs") %>%
  select("rs", starts_with("beta"))


names(psyma_realres) <- c("rs","DZA_L_Fit_med", "DZA_Lold_Fit_med", "A17_L_Fit_med", "A17_Lold_Fit_med")

save(psyma_realres, file="./psyma_real_betas.Rdata")

#psymb
psymb_realres <- traits[c(9:12)] %>% 
  reduce(full_join, by = "rs") %>%
  select("rs", starts_with("beta"))
 
names(psymb_realres) <- c("rs","DZA_L_Fit_med", "DZA_Lold_Fit_med", "A17_L_Fit_med", "A17_Lold_Fit_med")
 
save(psymb_realres, file="./psymb_real_betas.Rdata") 
```

Permutation tests
-----------------

All steps completed on remote server, rather than local machine.

### Randomize phenotypes

Use same phenotypes file as before, but shuffle values within each phenotype col for each phenotype, and produce 1000 shuffled sets.

Completed on remote server, rather than local machine:

``` r
# randomize .fam file
phenotypes <- read.delim("./phenos89.fam", header=FALSE)

# create a function to randomize phenotype file
random_phenos <- function(df, x){
  
    # randomize each col, excluding first (sample_IDs):
    rnd <- lapply(phenotypes[,6:10], sample)  ## or 6:9 for 89 strains
    # bind randomize cols
    rnd_df <- as.data.frame(rnd) 
    # format for GEMMA (.fam file), and save:
    phenotypes_rnd <- cbind(phenotypes[,c(1:5)],rnd_df) 
    
    # write-out random .fam files:
    write.table(phenotypes_rnd,file=paste0('./output/perm/','chr_phenotype_rnd',x,'.fam'), 
                 quote=FALSE, row.names=FALSE,col.names=FALSE,sep="\t")
    write.table(phenotypes_rnd,file=paste0('./output/perm/','psyma_phenotype_rnd',x,'.fam'), 
                 quote=FALSE, row.names=FALSE,col.names=FALSE,sep="\t")
    write.table(phenotypes_rnd,file=paste0('./output/perm/','psymb_phenotype_rnd',x,'.fam'), 
                 quote=FALSE, row.names=FALSE,col.names=FALSE,sep="\t")
}
# create 1000 randomized phenotype files:
random <- lapply(1:1000, random_phenos, df = phenotypes) 

# run in bg (takes a few mins)
nohup Rscript pheno_rand.R > pheno_rand.out &
```

### Create .bed and .bim files for permutations

Need names of all file inputs to match.

Completed on remote server, rather than local machine:

``` bash

#################### for 89 strains ####################

# nano make_beds.sh

for i in {1..1000} 
do 
   # for chrom
   cp ./chr_subset89_LD_plink.bed ./output/perm/gemma_input_files/chr_phenotype_rnd$i.bed 
   cp ./chr_subset89_LD_plink.bim ./output/perm/gemma_input_files/chr_phenotype_rnd$i.bim
   # for psyma
   cp ./psyma_subset89_LD_plink.bed ./output/perm/gemma_input_files/psyma_phenotype_rnd$i.bed 
   cp ./psyma_subset89_LD_plink.bim ./output/perm/gemma_input_files/psyma_phenotype_rnd$i.bim 
   # for psymb
   cp ./psymb_subset89_LD_plink.bed ./output/perm/gemma_input_files/psymb_phenotype_rnd$i.bed 
   cp ./psymb_subset89_LD_plink.bim ./output/perm/gemma_input_files/psymb_phenotype_rnd$i.bim 
done
```

### Run GEMMA on permutated datasets

Completed on remote server, rather than local machine:

``` bash

#################### for 89 strains ####################

# nano run_gemma_random.sh
for (( i = 1; i <= 1000; i++ ));      ### Outer for loop, rnd phenotypes ###
do

    for j in {1..4}; ### Inner for loop, specify number of traits ###
    do
       $GEMMA -bfile ./output/perm/gemma_input_files/chr_phenotype_rnd$i -k ./output/chr_subset89_LD_ksmat.sXX.txt \
       -lmm 4 -n $j -o ./perm/chr_phenotype_rnd${i}_lmm_trait${j}
    ### Make sure this code is the exact same used for the actual data, except for the -bfile and output names ###

      $GEMMA -bfile ./output/perm/gemma_input_files/psyma_phenotype_rnd$i -k ./output/psyma_subset89_LD_ksmat.sXX.txt \
      -lmm 4 -n $j -o ./perm/psyma_phenotype_rnd${i}_lmm_trait${j}
    ### Make sure this code is the exact same used for the actual data, except for the -bfile and output names ###  

      $GEMMA -bfile ./output/perm/gemma_input_files/psymb_phenotype_rnd$i -k ./output/psymb_subset89_LD_ksmat.sXX.txt \
      -lmm 4 -n $j -o ./perm/psymb_phenotype_rnd${i}_lmm_trait${j}
    ### Make sure this code is the exact same used for the actual data, except for the -bfile and output names ###  
    done
done

## 4 traits X 3 genomic elements X 1000 iterations = 12000 association files

# Move perms into trait directories: nano trait_directories.sh
for i in {1..4};
do
  mkdir trait_${i}/
  mv *trait${i}.* ./trait_${i}/
done

# Once it's fully run, extract pve and se estimates
grep "pve estimate" run_gemma_random.out > pve_ulmm_random.txt
grep "se(pve)" run_gemma_random.out > se_ulmm_random.txt
paste pve_ulmm_random.txt se_ulmm_random.txt > pve_se_ulmm_random.txt
```

### Merge outputs from perm test

Completed on remote server, rather than local machine:

``` bash
# nano merge_betas.sh
while read region; do
  i=0
    for x in {1..4}; 
    do
       cd trait_${x}/
       cut -f 2 ${region}_phenotype_rnd1_lmm_trait${x}.assoc.txt > ${region}_delim_us ## extract rs from file
       (head -n 1 ${region}_delim_us && tail -n +2 ${region}_delim_us | sort) > ${region}_delim_s 
       ## sort by position, excluding header
            for file in ${region}_*.assoc.txt;
            do
               i=$(($i+1))
               (head -n 1 ${file} && tail -n +2 ${file} | sort -k 2) | cut -f 8 > ${file}_${i}.temp 
               ## sort betas by ps, excluding header, cut betas
            done
    paste -d\\t ${region}_delim_s *.temp > ${region}_perm_comb_trait${x}.tsv ## add back in rs (sorted the same way)
    rm *.temp
    cd ../ ## get out of trait directory
    mv ./trait_${x}/${region}_perm_comb_trait${x}.tsv ./ ## move combined beta file to perm directory
    echo "merged perms for trait ${x} for ${region}"
    done

done < regions.list ## chr, psyma, psymb
```

Determine significance
----------------------

findinterval method for checking whether betas fall outside the distribution. All steps completed on remote server, rather than local machine.

Completed on remote server, rather than local machine:

``` r
#################### for 89 strains ####################

# nano chr_sig.R

# load the realbetas for the chromosome
load(file="./chr_real_betas.Rdata") ## loads chr_realres

# make sure realres object is sorted the same way as sims
sims.rs <- read.csv("./output/perm/chr_perm_comb_trait1.tsv",sep="\t",header=T)$rs
sortindex <- sapply(sims.rs,function(z) which(chr_realres$rs == z))
realres.rs <- chr_realres[sortindex,] 

# Import GEMMA association files from permutations:
files <- list.files(path ="./output/perm", pattern="chr_perm_comb_", 
                    full.names = T) ## file order needs to be specified unless leading zero added

betacols.realres <- c(2:5) ## specify traits
colnames(realres.rs[betacols.realres]) ## double check this
realsnpsBETAS <- realres.rs[,betacols.realres]
realintFull <- matrix(NA,ncol=4,nrow=636) ## need to look up number of vars (str(chr_realres))
for(i in 1:4){
    permut <- read.table(files[i],header=T,sep="\t") #read in the file
    realintFull[,i] <- sapply(1:nrow(realres.rs), function(z) 
            findInterval(realsnpsBETAS[ z,i ],sort(permut[z,-1]))/1000 
            )
}   

# Does real data exceed 5% cuttoff?
realsigFull <- (realintFull > 0.975 | realintFull < 0.025)
colSums(realsigFull)
data.frame(trait =colnames(realres.rs[betacols.realres]) , snps = colSums(realintFull > 0.975 | realintFull < 0.025))

dfFIsig <- data.frame(realsigFull)
colnames(dfFIsig) <- paste(colnames(realres.rs[betacols.realres]),".FIsig",sep="")
realres.psFIsig <- cbind(realres.rs,dfFIsig)
save(realres.psFIsig, file = "./chr_betasigs.Rdata")

# nano psyma_sig.R

# load the realbetas for the psymA
load(file="./psyma_real_betas.Rdata") ## loads psyma_realres

# make sure realres object is sorted the same way as sims
sims.rs <- read.csv("./output/perm/psyma_perm_comb_trait1.tsv",sep="\t",header=T)$rs
sortindex <- sapply(sims.rs,function(z) which(psyma_realres$rs == z))
realres.rs <- psyma_realres[sortindex,] 

# Import GEMMA association files from permutations:
files <- list.files(path ="./output/perm", pattern="psyma_perm_comb_", 
                    full.names = T) ## file order needs to be specified

betacols.realres <- c(2:5) ## specify trait cols
colnames(realres.rs[betacols.realres]) ## double check this
realsnpsBETAS <- realres.rs[,betacols.realres]
realintFull <- matrix(NA,ncol=4,nrow=2822) ## need to look up number of vars
for(i in 1:4){
    permut <- read.table(files[i],header=T,sep="\t") #read in the file
    realintFull[,i] <- sapply(1:nrow(realres.rs), function(z) 
            findInterval(realsnpsBETAS[ z,i ],sort(permut[z,-1]))/1000 
            )
}   

# Does real data exceed 5% cuttoff?
realsigFull <- (realintFull > 0.975 | realintFull < 0.025)
colSums(realsigFull)
data.frame(trait =colnames(realres.rs[betacols.realres]) , snps = colSums(realintFull > 0.975 | realintFull < 0.025))

dfFIsig <- data.frame(realsigFull)
colnames(dfFIsig) <- paste(colnames(realres.rs[betacols.realres]),".FIsig",sep="")
realres.psFIsig <- cbind(realres.rs,dfFIsig)
save(realres.psFIsig, file = "./psyma_betasigs.Rdata")

# nano psymb_sig.R

# load the realbetas for the psymB
load(file="./psymb_real_betas.Rdata") ## loads psymb_realres

# make sure realres object is sorted the same way as sims
sims.rs <- read.csv("./output/perm/psymb_perm_comb_trait1.tsv",sep="\t",header=T)$rs
sortindex <- sapply(sims.rs,function(z) which(psymb_realres$rs == z))
realres.rs <- psymb_realres[sortindex,] 

# Import GEMMA association files from permutations:
files <- list.files(path ="./output/perm", pattern="psymb_perm_comb_", 
                    full.names = T) ## file order needs to be specified

betacols.realres <- c(2:5) ## specify trait cols
colnames(realres.rs[betacols.realres]) ## double check this
realsnpsBETAS <- realres.rs[,betacols.realres]
realintFull <- matrix(NA,ncol=4,nrow=4157) ## need to look up number of vars
for(i in 1:4){
    permut <- read.table(files[i],header=T,sep="\t") #read in the file
    realintFull[,i] <- sapply(1:nrow(realres.rs), function(z) 
            findInterval(realsnpsBETAS[ z,i ],sort(permut[z,-1]))/1000 
            )
}   

# Does real data exceed 5% cuttoff?
realsigFull <- (realintFull > 0.975 | realintFull < 0.025)
colSums(realsigFull)
data.frame(trait =colnames(realres.rs[betacols.realres]) , snps = colSums(realintFull > 0.975 | realintFull < 0.025))

dfFIsig <- data.frame(realsigFull)
colnames(dfFIsig) <- paste(colnames(realres.rs[betacols.realres]),".FIsig",sep="")
realres.psFIsig <- cbind(realres.rs,dfFIsig)
save(realres.psFIsig, file = "./psymb_betasigs.Rdata")

## scp'ed .Rdata files to ./Data_input/betasigs_14Apr2021/
```

Merge results (191 and 89 strains)
----------------------------------

``` r
## chromosome first
load(file = "../../Complex_genetics/GWAS/Data_output/betasigs_29Mar2021/chr_betasigs.Rdata") ## realres.psFIsig
chr_sigs <- realres.psFIsig

load("./Data_input/betasigs_14Apr2021/chr_betasigs_89strains.Rdata") ## realres.psFIsig
chr_sigs_fit <- realres.psFIsig

chr_sigs_all <- full_join(chr_sigs, chr_sigs_fit, by ="rs", all = TRUE)
chr_sigs_all$region <- "Chromosome"

## psyma
load("../../Complex_genetics/GWAS/Data_output/betasigs_29Mar2021/psyma_betasigs.Rdata") ## realres.psFIsig
psyma_sigs <- realres.psFIsig

load("./Data_input/betasigs_14Apr2021/psyma_betasigs_89strains.Rdata") ## realres.psFIsig
psyma_sigs_fit <- realres.psFIsig

psyma_sigs_all <- full_join(psyma_sigs, psyma_sigs_fit, by ="rs", all = TRUE)
psyma_sigs_all$region <- "pSymA"

## psymb
load("../../Complex_genetics/GWAS/Data_output/betasigs_29Mar2021/psymb_betasigs.Rdata") ## realres.psFIsig
psymb_sigs <- realres.psFIsig

load("./Data_input/betasigs_14Apr2021/psymb_betasigs_89strains.Rdata") ## realres.psFIsig
psymb_sigs_fit <- realres.psFIsig

psymb_sigs_all <- full_join(psymb_sigs, psymb_sigs_fit, by ="rs", all = TRUE)
psymb_sigs_all$region <- "pSymB"

## combine altogether:
SNPs_comb <- rbind(chr_sigs_all, psyma_sigs_all, psymb_sigs_all)
SNPs_comb$ps <- sapply(strsplit(as.character(SNPs_comb$rs), split="-"), "[", 4)
## 8063 SNPs

## add in snpEff info:
VCF_ann <- read.delim("../../Complex_genetics/GWAS/Data_output/vcf_annotations_29Mar2021/subset191_LD.ann.TABLE")
## extract effect and impact from snpEff annotation field
VCF_ann$effect <- sapply(strsplit(as.character(VCF_ann$ANN), split="\\|"), "[", 2)
VCF_ann$impact <- sapply(strsplit(as.character(VCF_ann$ANN), split="\\|"), "[", 3)
VCF_ann$gene_interval <- sapply(strsplit(as.character(VCF_ann$ANN), 
                                         split="\\|"), "[", 5)
## create rs
VCF_ann$region <- ifelse(VCF_ann$CHROM == "NZ_CP021797.1", "chr", 
                         ifelse(VCF_ann$CHROM == "NZ_CP021798.1", "psyma",
                                "psymb"))
VCF_ann$add1 <- "freebayes"
VCF_ann$add2 <- "snp"
VCF_ann$rs <- str_c(VCF_ann$add1, VCF_ann$add2, VCF_ann$region, VCF_ann$POS,VCF_ann$ALT,
                    sep = "-", collapse = NULL)
## combine
SNPs_vcf_ann <- left_join(SNPs_comb, 
                          VCF_ann[,c("rs","effect","impact",
                                     "gene_interval","ALT","REF")], by = "rs")
## 8063 SNPs

## get MAF, minor allele, et c., from GEMMA outputs
chr_GEMMA_info1 <- 
  read.delim("./Data_input/betasigs_14Apr2021/chr_subset89_LD_ulmm_trait_1.assoc.txt")
psyma_GEMMA_info1 <- 
  read.delim("./Data_input/betasigs_14Apr2021/psyma_subset89_LD_ulmm_trait_1.assoc.txt")
psymb_GEMMA_info1 <- 
  read.delim("./Data_input/betasigs_14Apr2021/psymb_subset89_LD_ulmm_trait_1.assoc.txt")
GEMMA_info1 <- rbind(chr_GEMMA_info1, psyma_GEMMA_info1, psymb_GEMMA_info1)
### merge with 191 strains
chr_GEMMA_info2 <- 
  read.delim("../../Complex_genetics/GWAS/Data_output/betasigs_29Mar2021/chr_subset191_LD_ulmm_trait_01.assoc.txt")
psyma_GEMMA_info2 <- 
  read.delim("../../Complex_genetics/GWAS/Data_output/betasigs_29Mar2021/psyma_subset191_LD_ulmm_trait_01.assoc.txt")
psymb_GEMMA_info2 <- 
  read.delim("../../Complex_genetics/GWAS/Data_output/betasigs_29Mar2021/psymb_subset191_LD_ulmm_trait_01.assoc.txt")
GEMMA_info2 <- rbind(chr_GEMMA_info2, psyma_GEMMA_info2, psymb_GEMMA_info2)

GEMMA_info <- full_join(x= GEMMA_info1, y= GEMMA_info2, by = "rs", 
                        suffix = c("_89","_191"))
## 8063 SNPs

SNPs_all <- full_join(SNPs_vcf_ann, GEMMA_info[,c("rs","allele1_89", "allele0_89",
                                                  "af_89",
                                                  "allele1_191", "allele0_191",
                                                  "af_191")], by = "rs")
## 8063 SNPs

## include minor allele state: whether it's ALT or REF
SNPs_all$ma_state_89 <- ifelse(SNPs_all$ALT == SNPs_all$allele1_89, "ALT", "REF")
SNPs_all$ma_state_89 <- as.factor(SNPs_all$ma_state_89)
SNPs_all$ma_state_191 <- ifelse(SNPs_all$ALT == SNPs_all$allele1_191, "ALT", "REF")
SNPs_all$ma_state_191 <- as.factor(SNPs_all$ma_state_191)

### add in gene products
load(file = "../../Complex_genetics/GWAS/Data_output/gene_functions.Rdata") # loads funcs

## match info
SNPs_all$start_pos <- funcs$start_pos[match(SNPs_all$rs, funcs$rs)]
SNPs_all$end_pos <- funcs$end_pos[match(SNPs_all$rs, funcs$rs)]
SNPs_all$ncbi_func <- funcs$ncbi_func[match(SNPs_all$rs, funcs$rs)]
SNPs_all$RefSeq_ID <- funcs$RefSeq_ID[match(SNPs_all$rs, funcs$rs)]
SNPs_all$protein_ID <- funcs$protein_ID[match(SNPs_all$rs, funcs$rs)]
SNPs_all$gene_ID <- funcs$gene_ID[match(SNPs_all$rs, funcs$rs)]
SNPs_all$gene_ann <- funcs$gene_name[match(SNPs_all$rs, funcs$rs)]

## replace na in NCBI fun with non-coding
SNPs_all$ncbi_func[is.na(SNPs_all$ncbi_func)] <- "non-coding"

## code region as factor
SNPs_all$region <- factor(SNPs_all$region, levels = c("Chromosome","pSymA","pSymB"))

## how many SNPs overlap?
sum(is.na(SNPs_all$af_89)) ## 448 unique to 191 set
```

    ## [1] 448

``` r
sum(is.na(SNPs_all$af_191)) ## 98 unique to 89 set
```

    ## [1] 98

``` r
# subset to relevant traits
## select relevant cols
SNPs_sel <- SNPs_all %>%
  select(-ends_with("_1"), -ends_with("_2"), -ends_with("_1.FIsig"), -ends_with("_2.FIsig"),
         -contains("height"), -contains("leaf"), -contains("_L_"), -contains("plast"))

## only include SNPs overlapping in both sets
SNPs <- SNPs_sel %>%
  filter(!is.na(af_89) & !is.na(af_191)) %>%
  droplevels(.)
## 7517 overlapping variants

## noticed lack of interval for locus on pSymB, looked up on ncbi:
SNPs$gene_interval[!nzchar(SNPs$gene_interval)] <- "CDO30_27950-CDO30_27955"

## add in how many variants within LD group
one_variant <- 
  read.delim("../../Complex_genetics/GWAS/Data_output/one_variant.tsv", header=FALSE)
colnames(one_variant) <- c("region","start_ps","end_ps","rs","group_no","vars_in_group")
### count number of commas to get total variant count within group
one_variant$no_vars_group <- str_count(one_variant$vars_in_group, ',') + 1
### add to SNPs
SNPs$no_vars_group <- one_variant$no_vars_group[match(SNPs$rs, one_variant$rs)]

save(SNPs, file = "./Data_output/SNPs.Rdata")
write.csv(SNPs, "./Data_output/SNPs_wide.csv", row.names = FALSE)
```

### Convert from wide to long (191 and 89 strains)

``` r
# load dataset
load(file = "./Data_output/SNPs.Rdata") ## loads SNPs

# create long version of GEMMA_vars_sig
rownames(SNPs) <- SNPs[,1] ## name rows by position

# use gather to convert wide to long
SNPs_long.1 <- SNPs %>% 
  gather(betas, scores, ends_with("_3"), ends_with("_4"), ends_with("_med")) %>% ## gather betas
  mutate(rs_betas = paste(rs, betas, sep="_")) %>%
  select(-ends_with(".FIsig"))

SNPs_long.2 <- SNPs %>% 
  gather(betas, sig, ends_with(".FIsig")) %>% ## gather sigs
  mutate(rs_betas = paste(rs, betas, sep="_")) %>%
  select(rs_betas, sig)

SNPs_long.2$rs_betas <- str_replace(SNPs_long.2$rs_betas, ".FIsig", "")

# merge into one, getting rid of redundant cols
SNPs_long1 <- full_join(SNPs_long.1, SNPs_long.2, by = "rs_betas")

# Rename cols
colnames(SNPs_long1)[colnames(SNPs_long1)=="betas"] <- "assoc"

# rename factors
SNPs_long1$assoc <- recode_factor(SNPs_long1$assoc, 
                                       `A17_Lold_Fit_med` = "fit_A17_6O",
                                       `DZA_Lold_Fit_med` = "fit_DZA_5O")

# split into trait and env
SNPs_long1.s <- separate(data = SNPs_long1, 
                            col = assoc, into = c("trait","line","exp"), sep = "\\_", remove = FALSE)

# format cols, add line
SNPs_long1.s$ps <- as.numeric(SNPs_long1.s$ps)
SNPs_long1.s <- SNPs_long1.s %>% 
  unite(col = line_exp, c("line","exp"),sep = "-", remove = FALSE)
SNPs_long1.s$line_exp <- factor(SNPs_long1.s$line_exp, 
                                    levels = c("DZA-3","A17-4","DZA-5O","A17-6O"))

# condense maf and state
SNPs_long1.s$maf <- ifelse(SNPs_long1.s$trait == "fit", SNPs_long1.s$af_89, SNPs_long1.s$af_191)
SNPs_long1.s$ma_state <- ifelse(SNPs_long1.s$trait == "fit", paste(SNPs_long1.s$ma_state_89),
                                paste(SNPs_long1.s$ma_state_191))

## select cols
SNPs_long <- SNPs_long1.s %>%
  select(-ends_with("_89"), -ends_with("_191"), -rs_betas)

## add in direction:
SNPs_long$dir <- ifelse(SNPs_long$scores > 0, "-", "+")

# filter only sig SNPs
SNPs_sig_long <- SNPs_long %>%
  filter(sig == TRUE) %>%
  select(-sig)

SNPs_sig_long_sum <- SNPs_sig_long %>%
  group_by(trait, region, line) %>%
  summarize(count = n())

# save
write.csv(SNPs_sig_long, file = "./Data_output/SNPs_ann_effect.csv", 
          row.names = FALSE)
save(SNPs_sig_long, file = "./Data_output/SNPs_sig_long.Rdata")
```

### Summarize sig associations at the SNP and gene levels (191 and 89 strains)

``` r
load(file = "./Data_output/SNPs_sig_long.Rdata") # loads SNPs_sig_long

SNPs_ann_ps <- SNPs_sig_long %>%
    mutate(assoc = paste0(trait,dir,"_", line)) %>%
    group_by(rs) %>% ## summarized to rs-level
  summarize(
    no_effects = n(),
    ave_effect = mean(abs(scores)),
    lines = paste(unique(line), collapse = ", "),
    assocs = paste(unique(assoc), collapse = ", ")) %>%
  as.data.frame(.)

SNPs_ann_ps$host <- ifelse(SNPs_ann_ps$lines == "A17, DZA" | 
                             SNPs_ann_ps$lines == "DZA, A17", "Both",
                           ifelse(SNPs_ann_ps$lines == "A17", "A17_only", 
                                  "DZA_only"))

save(SNPs_ann_ps, file = "./Data_output/SNPs_ann_ps.Rdata")

# Summarize to the gene-level (all traits)
SNPs_ann_gene <- SNPs_sig_long %>%
  filter(ncbi_func != "non-coding") %>%
  mutate(assoc = paste(trait, line, sep = '-')) %>%
  group_by(region, RefSeq_ID, 
           protein_ID, ncbi_func, gene_interval, start_pos, end_pos) %>%
 summarize(
    no_vars = n_distinct(ps),
    min_ps = min(ps), 
    max_ps = max(ps), 
    no_effects = n(),
    ave_score = mean(scores),
    assocs = paste(unique(assoc), collapse = ", "),
    lines = paste(unique(line), collapse = ", "),
    exps = paste(unique(exp), collapse = ", ")) %>%
  as.data.frame(.)

SNPs_ann_gene$host <- ifelse(SNPs_ann_gene$lines == "A17, DZA" | 
                               SNPs_ann_gene$lines == "DZA, A17", "Both",
                           ifelse(SNPs_ann_gene$lines == "A17", "A17 only", 
                                  "DZA only"))

write.csv(SNPs_ann_gene, "./Data_output/SNPs_ann_gene.csv", row.names = FALSE)

SNPs_ann_gene.sum <- SNPs_ann_gene %>%
  group_by(region, host) %>%
  summarize(count = n())

kable(SNPs_ann_gene.sum)
```

| region     | host     |  count|
|:-----------|:---------|------:|
| Chromosome | A17 only |     62|
| Chromosome | Both     |      3|
| Chromosome | DZA only |     13|
| pSymA      | A17 only |     80|
| pSymA      | Both     |    136|
| pSymA      | DZA only |    184|
| pSymB      | A17 only |    224|
| pSymB      | Both     |    240|
| pSymB      | DZA only |    190|

``` r
# summarize number of genes for each trait

# source the function
source("../Source_code/gene_sum_func.R")

trait.list <- c("chloro1","shoot","nod","nod.weight","fit")

gene_sum_out <- lapply(trait.list, gene_sum_func, df = SNPs_sig_long)
```

    ## [1] "chloro1"
    ## [1] "shoot"
    ## [1] "nod"
    ## [1] "nod.weight"
    ## [1] "fit"

``` r
Venn_diagram_nums <- gene_sum_out %>%
  reduce(full_join, by = "host_exps")

names(Venn_diagram_nums) <- c("host_exps","chloro1","shoot","nod","nod.weight","fit")

Venn_diagrams <- rbind(Venn_diagram_nums, c("Total_genes",colSums(Venn_diagram_nums[,2:6], na.rm=TRUE)))

kable(Venn_diagrams)
```

| host\_exps   | chloro1 | shoot | nod | nod.weight | fit |
|:-------------|:--------|:------|:----|:-----------|:----|
| A17 only\_4  | 100     | 124   | 203 | 167        | NA  |
| Both\_3, 4   | 9       | 34    | 40  | 34         | NA  |
| DZA only\_3  | 141     | 247   | 99  | 157        | NA  |
| A17 only\_6O | NA      | NA    | NA  | NA         | 210 |
| Both\_5O, 6O | NA      | NA    | NA  | NA         | 81  |
| DZA only\_5O | NA      | NA    | NA  | NA         | 235 |
| Total\_genes | 250     | 405   | 342 | 358        | 526 |

``` r
write.csv(Venn_diagrams, "./Data_output/Venn_numbers.csv", row.names = FALSE)
```

Select traits for analyses
--------------------------

``` r
load(file = "./Data_output/SNPs.Rdata") ## loads SNPs

## subset dataframe, rename cols

SNPs_sel <- SNPs %>%
  select(rs, 
         # betas DZA
         beta.chloro.DZA = chloro1_DZA_3, 
         beta.shoot.DZA = shoot_DZA_3, 
         beta.nod.DZA = nod_DZA_3, 
         beta.nod.weight.DZA = nod.weight_DZA_3, 
         beta.fit.DZA = DZA_Lold_Fit_med, 
         # sig DZA
         sig.chloro.DZA = chloro1_DZA_3.FIsig, 
         sig.shoot.DZA = shoot_DZA_3.FIsig, 
         sig.nod.DZA = nod_DZA_3.FIsig, 
         sig.nod.weight.DZA = nod.weight_DZA_3.FIsig, 
         sig.fit.DZA = DZA_Lold_Fit_med.FIsig, 
         # betas A17
         beta.chloro.A17 = chloro1_A17_4, 
         beta.shoot.A17 = shoot_A17_4, 
         beta.nod.A17 = nod_A17_4, 
         beta.nod.weight.A17 = nod.weight_A17_4, 
         beta.fit.A17 = A17_Lold_Fit_med, 
         # sig A17
         sig.chloro.A17 = chloro1_A17_4.FIsig, 
         sig.shoot.A17 = shoot_A17_4.FIsig, 
         sig.nod.A17 = nod_A17_4.FIsig, 
         sig.nod.weight.A17 = nod.weight_A17_4.FIsig, 
         sig.fit.A17 = A17_Lold_Fit_med.FIsig, 
         # other info:
         region, ps, start_pos, end_pos, af_191, af_89, impact, RefSeq_ID, ncbi_func,
         protein_ID, gene_ID, gene_interval, gene_ann, no_vars_group
         )

save(SNPs_sel, file = "./Data_output/SNPs_sel.Rdata")
```
