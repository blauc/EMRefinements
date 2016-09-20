MKFILE_PATH    := $(shell pwd)
GMX_SRC_PATH    = /home/cblau/Projects/gromacs/
GMX             = /home/cblau/local/gromacs-install/bin/gmx

INPUT_DIR       = $(MKFILE_PATH)/input
SIGMA           = 0.2
SPACING         = 0.1

INPUT_STRUCTURE = $(INPUT_DIR)/1ake.pdb
TARGET_STRUCTURE = $(INPUT_DIR)/4ake.pdb
EXP_INPUT       = $(INPUT_DIR)/input.json
MDP_FILE        = $(INPUT_DIR)/emin.mdp

SKIP            = 1

FORCE_FIELD     = 8
#  1: AMBER03 2: AMBER94 3: AMBER96 4: AMBER99 5: AMBER99SB 6: AMBER99SB-ILDN 7: AMBERGS
#  8: CHARMM27
#  9: GROMOS96 10: GROMOS96 11: GROMOS96 12: GROMOS96 13: GROMOS96 14: GROMOS96
#  15: OPLS-AA/L

WATER_MODEL     = 1
# 1: TIP3P 2: TIP4P 3: TIPS3P 4: TIP5P 5: SPC 6: SPC/E 7: None

RESULTS_DIR     = $(MKFILE_PATH)/results
RMSD_FROM       = $(RESULTS_DIR)/from_input.xvg
RMSD_TO         = $(RESULTS_DIR)/to_target.xvg


# Intermediate
SCRATCH_DIR     = $(MKFILE_PATH)/scratch
MDRUN_DIR       = $(SCRATCH_DIR)/mdrun

TARGET_DENSITY  = $(SCRATCH_DIR)/target.ccp4
TARGET_DENSITY_FROM_MD = $(MDRUN_DIR)/target.ccp4

XTC_MD          = $(MDRUN_DIR)/mdrun.xtc
MD_MDP_FILE     = $(INPUT_DIR)/mdrun.mdp
XTC             = $(MDRUN_DIR)/traj.xtc
OUT_TRAJ_PDB    = $(MDRUN_DIR)/traj_comp.pdb
DT_OUT	        = 100
MDRUN_LOG       = $(MDRUN_DIR)/md.log
ENERGY_FILE     = $(MDRUN_DIR)/ener.edr

TPR_DIR         = $(SCRATCH_DIR)/tpr
TPR_EMIN        = $(TPR_DIR)/emin.tpr
TPR_ION_EMIN    = $(TPR_DIR)/emin_ion.tpr
TOPOLOGY        = $(TPR_DIR)/topology.top
TOPOLOGY_TARGET = $(TPR_DIR)/topology_target.top
TOPOLOGY_ITP    = $(TPR_DIR)/posre.itp
TOPOLOGY_ITP_TARGET    = $(TPR_DIR)/posre_target.itp
BOXED_STRUCTURE = $(TPR_DIR)/boxed.gro
BOXED_STRUCTURE_TARGET = $(TPR_DIR)/boxed_target.gro
TOP_STRUCTURE   = $(TPR_DIR)/top.gro
TOP_STRUCTURE_TARGET   = $(TPR_DIR)/top_target.gro
MD_INPUT_STRUCTURE  = $(TPR_DIR)/sim_input.gro
MDOUT_MDP       = $(TPR_DIR)/mdout.mdp
FITXTC          = $(MDRUN_DIR)/out.xtc
GRO_IONS        = $(TPR_DIR)/ions.gro
TOPOLOGY_IONS   = $(TPR_DIR)/iontop.top
MINIMIZED_STRUCTURE = $(RESULTS_DIR)/minimized.gro

TPR_MD          = $(TPR_DIR)/mdrun.tpr

FINAL_STRUCTURE = $(RESULTS_DIR)/final.gro
GROMPP          = $(GMX) grompp
GROMPP_ARGS     =
BOXTYPE         = triclinic
D_BOX           = 1.1
NTHREADS        = 4 #$(shell lscpu | grep "^CPU(s):" | awk '{print $$2}')
MDRUN           = $(GMX) mdrun -ntmpi 1  

all: $(XTC_MD)

density: $(TARGET_DENSITY)

$(BOXED_STRUCTURE_TARGET): $(TARGET_STRUCTURE) | $(TPR_DIR)
	$(GMX) editconf -f $(TARGET_STRUCTURE) -o $(BOXED_STRUCTURE_TARGET) -bt $(BOXTYPE) -d $(D_BOX)

$(TOP_STRUCTURE_TARGET): $(BOXED_STRUCTURE_TARGET)
	printf "$(FORCE_FIELD)\n$(WATER_MODEL)" | $(GMX) pdb2gmx -vsite h -f $(BOXED_STRUCTURE_TARGET) -o $(TOP_STRUCTURE_TARGET) -p $(TOPOLOGY_TARGET) -i $(TOPOLOGY_ITP_TARGET)


rmsd: $(FITXTC) $(TOP_STRUCTURE_TARGET)
	printf "1\n1\n" | $(GMX) rms -f $(FITXTC) -s $(TOP_STRUCTURE) -o results/from_input.xvg -xvg none -skip $(SKIP)
	printf "1\n1\n" | $(GMX) rms -f $(FITXTC) -s $(TOP_STRUCTURE_TARGET) -o results/to_target.xvg -xvg none -skip $(SKIP)

$(TARGET_DENSITY) :
	$(GMX) map -s $(TARGET_STRUCTURE) -sigma $(SIGMA) -mo $(TARGET_DENSITY) -spacing $(SPACING)

$(BOXED_STRUCTURE): $(INPUT_STRUCTURE) | $(TPR_DIR)
	$(GMX) editconf -f $(INPUT_STRUCTURE) -o $(BOXED_STRUCTURE) -bt $(BOXTYPE) -d $(D_BOX)

$(SCRATCH_DIR):
	mkdir $(SCRATCH_DIR)

$(TPR_DIR): | $(SCRATCH_DIR)
	mkdir $(TPR_DIR)

$(MDRUN_DIR): | $(SCRATCH_DIR)
	mkdir $(MDRUN_DIR)

$(MINIMIZED_STRUCTURE): $(TPR_ION_EMIN) | $(MDRUN_DIR)
	cd $(MDRUN_DIR); $(MDRUN) --s $(TPR_ION_EMIN) --v --nt $(NTHREADS) --x $(XTC) --g $(MDRUN_LOG) --e $(ENERGY_FILE) --c $(MINIMIZED_STRUCTURE)

$(TOPOLOGY): $(BOXED_STRUCTURE)
	printf "$(FORCE_FIELD)\n$(WATER_MODEL)" | $(GMX) pdb2gmx -f $(BOXED_STRUCTURE) -o $(TOP_STRUCTURE) -p $(TOPOLOGY) -i $(TOPOLOGY_ITP)

$(TOP_STRUCTURE) : $(TOPOLOGY)

$(TPR_EMIN): $(TOPOLOGY) $(MDP_FILE) $(MD_INPUT_STRUCTURE)| $(TPR_DIR)
	$(GROMPP) --f $(MDP_FILE) --po $(MDOUT_MDP) --p $(TOPOLOGY) --o $(TPR_EMIN) --c $(MD_INPUT_STRUCTURE)

$(GRO_IONS) $(TOPOLOGY_IONS): $(TPR_EMIN)
	cp $(TOPOLOGY) $(TOPOLOGY_IONS)
	printf "SOL\n" | $(GMX) genion -conc 0.15 -neutral yes -s $(TPR_EMIN) -o $(GRO_IONS) -p $(TOPOLOGY_IONS)

$(TPR_ION_EMIN): $(GRO_IONS) $(TOPOLOGY_IONS)
	$(GROMPP) --f $(MDP_FILE) --po $(MDOUT_MDP) --p $(TOPOLOGY_IONS) --o $(TPR_ION_EMIN) --c $(GRO_IONS)

$(TPR_MD) : $(TOPOLOGY_IONS) $(MD_MDP_FILE) $(EXP_INPUT) $(TARGET_DENSITY) | $(MINIMIZED_STRUCTURE) 
	$(GROMPP) --f $(MD_MDP_FILE) --p $(TOPOLOGY_IONS) --o $(TPR_MD) --c $(MINIMIZED_STRUCTURE)

$(XTC_MD): $(TPR_MD) | $(MDRUN_DIR)
	cd $(MDRUN_DIR); $(MDRUN) --s $(TPR_MD) --v --nt $(NTHREADS) --x $(XTC_MD) --g $(MDRUN_LOG) --e $(ENERGY_FILE) --c $(FINAL_STRUCTURE)

$(MD_INPUT_STRUCTURE): $(TOP_STRUCTURE) | $(TOPOLOGY)
	$(GMX) solvate -cp $(TOP_STRUCTURE) -p $(TOPOLOGY) -o $(MD_INPUT_STRUCTURE)

ddd: $(TPR_MD) $(EXP_INPUT) $(TARGET_DENSITY) | $(MDRUN_DIR)
	cd $(MDRUN_DIR); ddd --gdb --args $(MDRUN) --s $(TPR_MD) --v -nt $(NTHREADS) -x $(XTC_MD) -e $(ENERGY_FILE) -g $(MDRUN_LOG) -c $(FINAL_STRUCTURE)

ddt: $(TPR_MD) | $(MDRUN_DIR)
	cd $(MDRUN_DIR); ddt $(MDRUN) --s $(TPR_MD) --v -nt $(NTHREADS) -x $(XTC_MD) -e $(ENERGY_FILE) -g $(MDRUN_LOG) -c $(FINAL_STRUCTURE)

allinea_map: $(TPR_MD) | $(MDRUN_DIR)
	cd $(MDRUN_DIR); map $(MDRUN) --s $(TPR_MD) --v -nt $(NTHREADS) -x $(XTC_MD) -e $(ENERGY_FILE) -g $(MDRUN_LOG) -c $(FINAL_STRUCTURE)

view: $(OUT_TRAJ_PDB) $(TARGET_DENSITY_FROM_MD)
	pymol -q -x -d "load $(OUT_TRAJ_PDB), traj; load $(TARGET_DENSITY_FROM_MD)"
#	chimera $(OUT_TRAJ_PDB) $(TARGET_DENSITY)

$(OUT_TRAJ_PDB): $(FITXTC)
	printf "1\n" | $(GMX) trjconv -f $(FITXTC) -s $(TPR_MD) -o $(OUT_TRAJ_PDB) -skip $(SKIP) 

redo_md: | clean_md all
clean_md:
	rm -rf $(MDRUN_DIR)

clean:
	rm -rf $(SCRATCH_DIR)
	rm -f \#*
	rm -f mdout.mdp
clean_backup:
	rm $(shell f \#)
