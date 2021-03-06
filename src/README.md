# Introduction

This folder contains the codes that convert the EndCap binary data files into root trees that store the relevant information on each neutron event.

The conversion from binary data to fully mapped and tof-corrected
data is performed by running three applications/executables, which should be used in the following order:

        read_cdt_irina
        CreateCDTTree
        analysis

## Short description of the detector geometry

The EndCap detector consists of 4 sub-detectors called SUMOs (SUper MOdules) stack on top of each other. The SUMOs are numbered from 3 to 6. Each SUMOs consists of a number of trapezoidal segments and each segment consists of two independent Multi Wire Proportional Counters sharing a common cathode, which is segmented into 32 strips of equal size. The signals generated in each counter are amplified and collected by an anode plane with 32 wires. The anode plane is located half-distance between the segmented cathode and the segment wall. The inner walls of each segment and both sides of the segmented cathode are coated with a think layer of the B4C neutron converter.

The following design information is important to remember because it will be used in the source codes to mark the geographical location on the detector map of the capture of the neutron by the Boron converter followed by the generation of a signal in the readout electronics:

    SUMO3:  4 segments => 4x2 independent counters => 4x2x32 anode wires => 4x32 cathode strips.
    SUMO4:  6 segments => 6x2 independent counters => 6x2x32 anode wires => 6x32 cathode strips.
    SUMO5:  8 segments => 8x2 independent counters => 8x2x32 anode wires => 8x32 cathode strips.
    SUMO6:  10 segments => 10x2 independent counters => 10x2x32 anode wires => 10x32 cathode strips.

The design of the 4 SUMOs was done such that the stack forms a 12-degree sector of a circular detector around the beamline.

## Format of the datafiles

(see also the information included in the /data/ folder)

The raw datafiles produced by the EndCap detector contain three event types that are relevant to us: neutron events, chopper events and the boardID. The last two event types are also called METADATA.

The neutron events contain 4 sub-events: anode ID, cathode ID, timestamp and subID. The subID is always 0 in all datafiles collected with Endcap in July 2019. The chopper events contain only the chopper timestamp. The boardID is not an event per se but indicates which of the four SUMOs recorded the neutron signal.


## Description of the source codes

### read_cdt_irina.cpp

The original version of this code was sent to me in September 2019 by Morten Jagd Christensen from DMSC-ECDC. I think that the code was written by the former ECDC-member Martin Shetty during the EndCap test in July 2019. The original code reads out the binary datafiles (*.bin files in the /data/ folder) and prints out the parsed data.

#### Usage
    ./read_cdt_irina /data/*.bin >> logfile.out  


I made minimal changes to the original code. I just added a few lines to save the event information in the ascii output file "data_for_root.txt".
(The "data_for_root.txt" ascii file included in this folder was created by running read_cdt_irina with the event file "ni_sample_20190710_2_20190710_.bin" included in the /data/ folder.)

The output file has the the following format:

    1     1416799    -1    -1    -1
    111   362001      7    31     0
    1     1416697    -1    -1    -1
    111   334762     51    36     0
    111   340573     74    16     0
    1     1416964    -1    -1    -1
    111   516611     17     7     0
    1     1416799    -1    -1    -1
    11    714833    -10   -10   -10
    1     1418045    -1    -1    -1
    11    714700    -10   -10   -10
    1     1416697    -1    -1    -1

This file format helps with organising the event information in the root tree in the next step of the analysis.

The first column of the ascii file contains the "markers" for the event type. "1" is the marker for a boardID event, "111" marks a neutron event and "11" marks a choper event. Depending on the event type, the second column contains either the boardID, the neutron event timestamp or the  chopper timestamp. The negative numbers in the last three columns have no physical meaning. BoardID and chopper events will always have negative numbers in the last three columns, while the neutron event must always have positive values corresponding to the anode ID, cathode ID and subID (which is always zero in this data set.) Please note that at this step of the analysis the anodeID and cathodeID correspond to the channel number in   in the readout electronics.

### CreateCDTTree.C

You need to have ROOT installed in order to run this code.
(download from this site https://root.cern.ch/)

#### Usage:
Type

    > ./CreateCDTRootFile()

The macro reads the ascii file "data_for_root.txt" and transfers its content to the event tree "cdt_ev". Each line in the ascii file will become a column of the event tree. In ROOT language the elements of a column are called 'branches' (of the tree). In the present analysis the branches (and leaves) of the 'cdt_ev' tree are:

	 neutronTime     
	 cathode         
	 anode           
	 subID           
	 boardID         
	 module          
	 sumo            
	 chopperTime     

One branch corresponds to one neutron event. All branches of a specific ROOT tree must have the same number of entries (leaves). The leaves of the branch contain all available information on the detected neutron.

The output of the CreateCDTTree.C macro is the ROOT file "cdt.root", which is created in the current working directory.

To inspect and plot the content of the ROOT file open a ROOT session and the ROOT file by typing:

    > root -l cdt.root

To see the content of the ROOT file type:

    > root[1].ls

(you will get

		root [1] .ls
		TFile**		cdt.root
		TFile*		cdt.root
		KEY: TTree	cdt_ev;1	cdt_ev
)

To show the content of event#10 in the event tree you need to type:

    > root [2] cdt_ev->Show(10)

and you will get

    ======> EVENT:10
    neutronTime     = 107517
    cathode         = 27
    anode           = 59
    subID           = 0
    boardID         = 0
    module          = 1
    sumo            = 0
    chopperTime     = 0

You can easily plot any variable (leaf) of the tree by typing:

    > root[3]cdt_ev->Draw("cathode")

or for a 2D-plot

    > root[4]cdt_ev->Draw("cathode:anode")

Have fun playing with ROOT ;-)


### analysis_cdt.C

This macro uses the information contained in the tree "cdt_ev" created at the previous step, loops through the raw neutron events and assigns a chopper timestamp and boardID to each neutron event, applies the necessary corrections to the neutron timestamps and maps out the readout channels to the user defined map with the detector voxels.

This step of the analysis is the most time consuming due to the large number of operations performed on the neutron event data (calibrations, transformations, channel mapping).

See presentation ChannelMappingEndCap.pdf in the /documentation/ folder for additional info on channel mapping.


#### Usage:

Type

    > ./analysis

The output of this code is the same ROOT file "cdt.root" that now contains the original event tree plus 2 new event trees. Although ROOT allows to append new branches to an existing tree, for debugging purposes I prefer to leave the tree with the raw data intact and create new trees, one for the "calibrated" data called "cdt_new" and one for the voxel mapping called "cdt_new_cal".

To inspect and plot the content of the ROOT file open a ROOT session and the ROOT file by typing:

    > root -l cdt.root

To see the content of the ROOT file type:

root[1].ls

(you will get

root [1] .ls
	TFile**		cdt.root
 	TFile*		cdt.root
  	KEY: TTree	cdt_ev;1	cdt_ev
  	KEY: TTree	cdt_new;1	cdt_new

)
