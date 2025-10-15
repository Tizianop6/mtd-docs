In case the RelVal samples are not enough to carry out an in-depth study, this page shows how to generate samples from scratch.

# Generating samples at PU=0 and with vanilla CMSSW

A good place to start is to take RelVals as an example and then produce them privately.

For instance, one can search for existing SingleMu samples, e.g. `/RelValSingleMuPt10/*CMSSW_15_1_0_pre1*mcRun4*/GEN-SIM-RECO` on DAS. 

One of the results is the following: `/RelValSingleMuPt10/CMSSW_15_1_0_pre1-141X_mcRun4_realistic_v3_STD_RegeneratedGS_Run4D110_noPU-v1/GEN-SIM-RECO`, namely a Single Muon sample at pT = 10 GeV generated at PU = 0 and using CMSSW_15_1_0_pre1.

After finding the dataset name, you can look it up on the [RelVal website](https://cms-pdmv-prod.web.cern.ch/relval/): searching for `/RelValSingleMuPt10/CMSSW_15_1_0_pre1-141X_mcRun4_realistic_v3_STD_RegeneratedGS_Run4D110_noPU-v1/GEN-SIM-RECO` one gets to the [following](https://cms-pdmv-prod.web.cern.ch/relval/relvals?output_dataset=/RelValSingleMuPt10/CMSSW_15_1_0_pre1-141X_mcRun4_realistic_v3_STD_RegeneratedGS_Run4D110_noPU-v1/GEN-SIM-RECO&shown=2047&page=0&limit=50) page and can access the cmsDriver commands by clicking on the appropriate link under the "Actions" section.

The cmsDriver commands are the ones used to build the config files for the RelVals, and are a good starting point to start private generation.

Now, moving to an lxplus machine, you can start the generation by creating your working directory and downloading a local version of CMSSW. Highly suggested to use the same version of the RelVal reference sample, in our case CMSSW_15_1_0_pre1. 

You can have a local version using the following command:

    cmsrel CMSSW_15_1_0_pre1 
then you can enter the source directory and set the environment variables

    cd CMSSW_15_1_0_pre1
    cd src
    cmsenv
now, in a directory of your choice, you can run the cmsDriver commands in order to produce the config files that are able to carry out the sample generation. 

    cd ..
    mkdir yoursampledir
    cd yoursampledir

And then you can run the first cmsDriver command:

    cmsDriver.py SingleMuPt10_Eta2p85_cfi --beamspot DBrealisticHLLHC --conditions auto:phase2_realistic_T33_13TeV --customise SLHCUpgradeSimulations/Configuration/aging.customise_aging_1000 --datatier GEN-SIM --era Phase2C17I13M9 --eventcontent FEVTDEBUG --fileout "file:step1.root" --geometry ExtendedRun4D110 --nStreams 1 --nThreads 8 --no_exec --number 10 --python_filename step_1_cfg.py --relval 9000,100 --step GEN,SIM || exit $?

here we will not discuss all the parameters given to cmsDriver, the reference is [here](https://twiki.cern.ch/twiki/bin/view/CMSPublic/SWGuideCmsDriver).

The produced file contains the following part:


	process.generator = cms.EDFilter("Pythia8PtGun",
	    PGunParameters = cms.PSet(
		AddAntiParticle = cms.bool(True),
		MaxEta = cms.double(2.85),
		MaxPhi = cms.double(3.14159265359),
		MaxPt = cms.double(10.01),
		MinEta = cms.double(-2.85),
		MinPhi = cms.double(-3.14159265359),
		MinPt = cms.double(9.99),
		ParticleID = cms.vint32(-13)
	    ),
	    PythiaParameters = cms.PSet(
		parameterSets = cms.vstring()
	    ),
	    Verbosity = cms.untracked.int32(0),
	    firstRun = cms.untracked.uint32(1),
	    psethack = cms.string('single mu pt 10')
	)

above you can set the kind of particle that you are producing, the kinematic range, and the presence of an additional antiparticle shot back-to-back with respect to the first one.


Take a look also at this part, which is a the beginning of the config file, in order to set the number of events that are generated:

	process.maxEvents = cms.untracked.PSet(
	    input = cms.untracked.int32(10),
	    output = cms.optional.untracked.allowed(cms.int32,cms.PSet)
	)

Afterwards, you can carry out the generation by sending the command:

	cmsRun step_1_cfg.py 
	
The same can be done for the next steps, you can carry out the generation of the config files in the following  way:

	# Command for step 2:
	cmsDriver.py step2 --conditions auto:phase2_realistic_T33 --customise SLHCUpgradeSimulations/Configuration/aging.customise_aging_1000 --datatier GEN-SIM-DIGI-RAW --era Phase2C17I13M9 --eventcontent FEVTDEBUGHLT --filein "file:step1.root" --fileout "file:step2.root" --geometry ExtendedRun4D110 --nStreams 1 --nThreads 8 --no_exec --number 10 --python_filename step_2_cfg.py --step DIGI:pdigi_valid,L1TrackTrigger,L1,L1P2GT,DIGI2RAW,HLT:@relvalRun4 || exit $?
	
	# Command for step 3:
	cmsDriver.py step3 --conditions auto:phase2_realistic_T33 --customise SLHCUpgradeSimulations/Configuration/aging.customise_aging_1000 --datatier GEN-SIM-RECO,MINIAODSIM,DQMIO --era Phase2C17I13M9 --eventcontent FEVTDEBUGHLT,MINIAODSIM,DQM --filein "file:step2.root" --fileout "file:step3.root" --geometry ExtendedRun4D110 --nStreams 1 --nThreads 8 --no_exec --number 10 --python_filename step_3_cfg.py --step RAW2DIGI,RECO,RECOSIM,PAT,VALIDATION:@phase2Validation+@miniAODValidation,DQM:@phase2+@miniAODDQM || exit $?

	# Command for step 4:
	cmsDriver.py step4 --conditions auto:phase2_realistic_T33 --customise SLHCUpgradeSimulations/Configuration/aging.customise_aging_1000 --era Phase2C17I13M9 --filein "file:step3_inDQM.root" --fileout "file:step4.root" --filetype DQM --geometry ExtendedRun4D110 --mc --nStreams 1 --no_exec --number 10 --python_filename step_4_cfg.py --scenario pp --step HARVESTING:@phase2Validation+@phase2+@miniAODValidation+@miniAODDQM || exit $?

	# Command for step 5:
	cmsDriver.py step5 --conditions auto:phase2_realistic_T33 --customise SLHCUpgradeSimulations/Configuration/aging.customise_aging_1000 --datatier ALCARECO --era Phase2C17I13M9 --eventcontent ALCARECO --filein "file:step3.root" --fileout "file:step5.root" --geometry ExtendedRun4D110 --nStreams 1 --nThreads 8 --no_exec --number 10 --python_filename step_5_cfg.py --step ALCA:SiPixelCalSingleMuonLoose+SiPixelCalSingleMuonTight+TkAlMuonIsolated+TkAlMinBias+MuAlOverlaps+EcalESAlign+TkAlZMuMu+TkAlDiMuonAndVertex+HcalCalHBHEMuonProducerFilter+TkAlUpsilonMuMu+TkAlJpsiMuMu || exit $?
	
and then you can run the n-th step in this way:

	cmsRun step_n_cfg.py
	
	
# Generating samples with an edited version of CMSSW


# Adding PU

