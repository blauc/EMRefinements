MKFILE_PATH    := $(shell pwd)
GMX_SRC_PATH    = /home/cblau/Projects/gromacs/
GMX             = /home/cblau/Projects/external_potential/gromacs-install/bin/gmx

INPUT_DIR       = $(MKFILE_PATH)/input
SIGMA           = 0.5

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

SCRATCH_DIR     = $(MKFILE_PATH)/scratch
TARGET_DENSITY  = $(SCRATCH_DIR)/target.ccp4

MDRUN_DIR       = $(SCRATCH_DIR)/mdrun
XTC             = $(MDRUN_DIR)/traj.xtc
XTC_CENTERED    = $(MDRUN_DIR)/traj_centered.xtc
XTC_NOJUMP      = $(MDRUN_DIR)/traj_nojump.xtc
TRR             = $(MDRUN_DIR)/traj.trr
OUT_TRAJ_PDB    = $(MDRUN_DIR)/traj_comp.pdb
DT_OUT	        = 100
MDRUN_LOG       = $(MDRUN_DIR)/md.log
ENERGY_FILE     = $(MDRUN_DIR)/ener.edr

TPR_DIR         = $(SCRATCH_DIR)/tpr
TPR             = $(TPR_DIR)/emin.tpr
TOPOLOGY        = $(TPR_DIR)/topology.top
TOPOLOGY_ITP    = $(TPR_DIR)/posre.itp
BOXED_STRUCTURE = $(TPR_DIR)/boxed.gro
TOP_STRUCTURE   = $(TPR_DIR)/top.gro
SIM_INPUT       = $(TPR_DIR)/sim_input.gro
MDOUT_MDP       = $(TPR_DIR)/mdout.mdp

GROMPP          = $(GMX) grompp
GROMPP_ARGS     = --f $(MDP_FILE) --p $(TOPOLOGY) --o $(TPR) --c $(SIM_INPUT) -po $(MDOUT_MDP)

MDRUN           = $(GMX) mdrun
MD_ARGS         = --s $(TPR) --v -nt 8 -x $(XTC) -g $(MDRUN_LOG) -e $(ENERGY_FILE)

RUN             = cd $(MDRUN_DIR); $(MDRUN) $(MD_ARGS)

all: $(XTC)

$(TARGET_DENSITY) :
	$(GMX) map -s $(TARGET_STRUCTURE) -sigma $(SIGMA) -mo $(TARGET_DENSITY)

$(BOXED_STRUCTURE): $(INPUT_STRUCTURE) | $(TPR_DIR)
	$(GMX) editconf -f $(INPUT_STRUCTURE) -o $(BOXED_STRUCTURE) -bt triclinic -d 1.1

$(SCRATCH_DIR):
	mkdir $(SCRATCH_DIR)

$(TPR_DIR): | $(SCRATCH_DIR)
	mkdir $(TPR_DIR)

$(MDRUN_DIR): | $(SCRATCH_DIR)
	mkdir $(MDRUN_DIR)

$(XTC): $(TPR) $(EXP_INPUT) $(TARGET_DENSITY) | $(MDRUN_DIR)
	$(RUN)

$(TOPOLOGY): $(BOXED_STRUCTURE)
	printf "$(FORCE_FIELD)\n$(WATER_MODEL)" | $(GMX) pdb2gmx -f $(BOXED_STRUCTURE) -o $(TOP_STRUCTURE) -p $(TOPOLOGY) -i $(TOPOLOGY_ITP)

$(TOP_STRUCTURE) : $(TOPOLOGY)

$(TPR): $(TOPOLOGY) $(MDP_FILE) $(EXP_INPUT) $(SIM_INPUT)| $(TPR_DIR)
	$(GROMPP) $(GROMPP_ARGS)

$(SIM_INPUT): $(TOP_STRUCTURE) | $(TOPOLOGY)
	$(GMX) solvate -cp $(TOP_STRUCTURE) -p $(TOPOLOGY) -o $(SIM_INPUT)

ddd: $(TPR) $(EXP_INPUT) $(TARGET_DENSITY) | $(MDRUN_DIR)
	cd $(MDRUN_DIR); ddd --gdb --args $(MDRUN) $(MD_ARGS)

ddt:
	cd $(MDRUN_DIR); ddt $(MDRUN) $(MD_ARGS)

view: $(MDRUN_DIR)/out.xtc $(INPUT_STRUCTURE) $(TARGET_DENSITY)
	printf "1\n" | $(GMX) trjconv -f $(MDRUN_DIR)/out.xtc -s $(TPR) -o $(OUT_TRAJ_PDB) -skip $(SKIP) -pbc whole
	pymol -q -x -d "load $(OUT_TRAJ_PDB), traj; load $(TARGET_DENSITY)"
#	chimera $(OUT_TRAJ_PDB) $(TARGET_DENSITY)

$(OUT_TRAJ_PDB):
	printf "0" | gmx trjconv -f $(XTC) -s $(TPR) -o $(OUT_TRAJ_PDB) -pbc mol

clean:
	rm -rf $(SCRATCH_DIR)
	rm -f \#*
	rm -f mdout.mdp
