# LDSC_mod
AMP _ CMD ：
# make gwas summary file
How to add a new GWAS?
1[data download] All raw GWAS data are located at:
/sc/arion/projects/roussp01b/resources/databases/gwas
2[make a record in master spreadsheet] Relevant information for our (MAGMA, LDsc, scRDS) pipelines are kept in separate TSV file located at:
/sc/arion/projects/roussp01b/resources/databases/ldsc/metadata.tsv
3[GWAS preprocessing] Check that GWAS fits the following criteria. If they don’t, please adjust/filter raw GWAS accordingly:
[MAGMA preprocessing] Edit/Run the following scripts:

# STEP1: magma_A.sh
# STEP2: magma_B.sh
# STEP3: ./run_cadmagma.sh; note: not sh x (change basedir in downstream_magma.R)




# Run LDSC
## for run selected traits or seft-generated gwas summary:
grabSumstats function in down_stream_ldsc.R

## for R version or R packages,
1. comment for xx
#MY_RLIB = "/sc/arion/projects/roussp01a/jaro/programs/R_libs_4_0_3/"
#.libPaths(c(MY_RLIB, .libPaths()))

2. MODIFY step4jobGen function in STEP4.R; because of R library
step4jobGen=function(...,STEP4_IS_DRY_RUN=F,ACCOUNT_FOR_JOBS="acc_roussp01a",runID=42){
  myArgs=list(...)
  step4Dir=myArgs$outDir
  step4cores=if(is.null(myArgs$step4cores)) 1 else myArgs$step4cores+1

  #Test for existing output dir
  print(outDir)
  if(!is.null(myArgs$checkIfOutDirExists)) if(w$checkIfOutDirExists & file.exists(outDir)) myStop("output dir mustn't already exist") 
  myArgs$checkIfOutDirExists=F

  #create output structure
  lapply(paste0(step4Dir, "/", c("files", "log", "meta-files", "results", "/scripts")),dir.create,showWarnings=F,recursive=T) 

  #copy scripts #NOTE: rather volatile. 
  filesToSource=c("downstream_greatR.R","downstream_ldsc.R","STEP4.R")
  mapply(file.copy,paste0(primarySourceDir,"/",filesToSource), paste0(step4Dir,"/scripts/",filesToSource), overwrite =T)

  #save arguments
  save(myArgs, file=paste0(step4Dir,"/files/step4inputData.Rdata"))

  #write shell script wrapper for abovementioned R script:
  scriptname = paste0(step4Dir, "/scripts/step4wrap.sh")
  write(file=scriptname, paste0("#!/bin/bash
    #BSUB -J step4wrap",runID,"
    #BSUB -q premium
    #BSUB -P ", ACCOUNT_FOR_JOBS, " 
    #BSUB -n ", step4cores, " 
    #BSUB -R 'span[hosts=1]'
    #BSUB -R 'rusage[mem=", round(120000/step4cores), "]'
    #BSUB -W 140:00 
    #BSUB -L /bin/bash
    #BSUB -oo ", step4Dir, "/log/step4wrap.out
    #BSUB -eo ", step4Dir, "/log/step4wrap.err

    module purge
    module load openssl
    module load R/4.0.3
    source /hpc/packages/minerva-centos7/anaconda3/2021.5/etc/profile.d/conda.sh

    conda run -n Py39_R43_Ju10 Rscript ", step4Dir, "/scripts/step4wrap.R
  "))
    
      scriptname_R = paste0(step4Dir, "/scripts/step4wrap.R")
  write(file=scriptname_R, paste0("

    ",paste0("source(\"",step4Dir,"/scripts/",filesToSource,"\")\n",collapse=""), "
    load(\"", step4Dir,"/files/step4inputData.Rdata\")
    do.call(step4,myArgs)
  "))

  
  if(!STEP4_IS_DRY_RUN){ 
    system(paste0("cat ", scriptname, " | bsub"), intern=T)
  }else{
    message(paste0("step4 bash script written to ",scriptname, ". Just bsub it to run"))
  }
}



