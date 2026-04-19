#!/bin/bash

# !CALLING SEQUENCE:
#
#        ./runGSI.sh EXP LABELI num_dom
#
#           o EXP     :   Nome do experimento, ex. SA
#           o LABELI  :   Data da analise (YYYYMMDDHH), ex. 2021010100
#           o num_dom :   Quantidade de dominios, ex. 1
#           o ciclo   :   Numero de ciclo: 1, 2, 3...
#           o chem    :   Para uso com WRF-Chem - Sim: 1 ; Não: 0

function usage(){
   sed -n '/^# !CALLING SEQUENCE:/,/^# !/{p}' runGSI.sh | head -n 9
}

#
# Verificando argumentos de entrada
#

if [ $# -ne 5 ]; then
   usage
   exit 1
fi

EXP=${1}
LABELI=${2} ; HH=${LABELI:8:2}
NDOM=${3}
CICLO=${4}
chem=${5}

printf "\nLABELI: ${LABELI:0:8}00\n"

BASEDIR=${HOMEBE}/Opera
GSIROOT=${HOME}/Packages/gsi_new
RUNDIR=${BASEDIR}/run
EXEDIR=${GSIROOT}/build/src/gsi
SCRDIR=${BASEDIR}/scripts

EXPDIR=${RUNDIR}/${EXP}/${LABELI:0:10}/rungsi
LOGDIR=${EXPDIR}/logs

if [ $CICLO -eq 0 ]; then
  BK_ROOT=${RUNDIR}/${EXP}/${LABELI:0:10}/runwps
  BK_FILE=${BK_ROOT}/wrfinput_d01
 else
  wrf_gsi_date=${LABELI:0:4}-${LABELI:4:2}-${LABELI:6:2}_${LABELI:8:2}:00:00
  BK_ROOT=${RUNDIR}/${EXP}/${LABELI:0:10}/runwrf
  BK_FILE=${BK_ROOT}/wrf_gsi_input_d01_$wrf_gsi_date
fi

OBS_ROOT=/oper/dados/dboper/raw/arch/mod/ncep/gdas/${LABELI:0:4}/${LABELI:4:2}/${LABELI:6:2}
OBS_ROOTchem=${BASEDIR}/dadata/chem
#OBS_ROOT=/mnt/beegfs/eder.vendrasco/Opera/dadata/ob/preBUFR
PREPBUFR=${OBS_ROOT}/gdas.t${LABELI:8:2}z.prepbufr.nr
PREPBUFRchem=${OBS_ROOTchem}/modisbufr_template
PREPBUFRchemTXT=${OBS_ROOTchem}/MODIS_AOD_${LABELI:0:8}_${LABELI:8:2}00z.cols.bulletin.txt #MODIS_AOD_ocean_${LABELI:0:8}_${LABELI:8:2}00z.cols.bulletin.txt
PREPBUFRchemVIIRS=${OBS_ROOTchem}/modisbufr_template   # adicionado
PREPBUFRchemTXTVIIRS=${OBS_ROOTchem}/VIIRS_AOD_${LABELI:0:8}_${LABELI:8:2}00z.cols.bulletin.txt   # adicionado

GSI_NAMELIST=${SCRDIR}/comgsi_namelist.sh
if [ $chem -eq 1 ]; then GSI_NAMELIST=${SCRDIR}/comgsi_namelist_chem.sh ; fi
FIX_ROOT=${GSIROOT}/fix
GSI_EXE=${EXEDIR}/gsi.x
CRTM_ROOT=${GSIROOT}/CRTM_v2.3.0

#NETCDFC=${HOME}/BUILD_WRF/LIBRARIES/netcdf-c-4.9.2
#NETCDFF=${HOME}/BUILD_WRF/LIBRARIES/netcdf-fortran-4.6.0
#HDF5=${HOME}/BUILD_WRF/LIBRARIES/hdf5

#export LD_LIBRARY_PATH=$NETCDFC/lib:$NETCDFF/lib:$HDF5/lib:$LD_LIBRARY_PATH

if_hybrid=No     # Yes, or, No -- case sensitive !
if_4DEnVar=No    # Yes, or, No -- case sensitive ! (set if_hybrid=Yes first)
if_observer=No   # Yes, or, No -- case sensitive !
if_nemsio=No     # Yes, or, No -- case sensitive !
if_oneob=No      # Yes, or, No -- case sensitive !

bk_core=ARW
if [ $chem -eq 1 ]; then bk_core=WRFCHEM_GOCART ; obs_type=VIIRSAOD ; fi # comentado/modificado: antes MODISAOD
bkcv_option=NAM
if_clean=noclean

#
# setup whether to do single obs test
if [ ${if_oneob} = Yes ]; then
  if_oneobtest='.true.'
else
  if_oneobtest='.false.'
fi
#
# setup for GSI 3D/4D EnVar hybrid
if [ ${if_hybrid} = Yes ] ; then
  PDYa=`echo $LABELI | cut -c1-8`
  cyca=`echo $LABELI | cut -c9-10`
  gdate=`date -u -d "$PDYa $cyca -6 hour" +%Y%m%d%H` #guess date is 6hr ago
  gHH=`echo $gdate |cut -c9-10`
  datem1=`date -u -d "$PDYa $cyca -1 hour" +%Y-%m-%d_%H:%M:%S`  #1hr ago
  datep1=`date -u -d "$PDYa $cyca 1 hour"  +%Y-%m-%d_%H:%M:%S`  #1hr later
  if [ ${if_nemsio} = Yes ]; then
    if_gfs_nemsio='.true.'
    ENSEMBLE_FILE_mem=${ENS_ROOT}/gdas.t${gHH}z.atmf006s.mem
  else
    if_gfs_nemsio='.false.'
    ENSEMBLE_FILE_mem=${ENS_ROOT}/sfg_${gdate}_fhr06s_mem
  fi

  if [ ${if_4DEnVar} = Yes ] ; then
    BK_FILE_P1=${BK_ROOT}/wrfout_d01_${datep1}
    BK_FILE_M1=${BK_ROOT}/wrfout_d01_${datem1}

    if [ ${if_nemsio} = Yes ]; then
      ENSEMBLE_FILE_mem_p1=${ENS_ROOT}/gdas.t${gHH}z.atmf009s.mem
      ENSEMBLE_FILE_mem_m1=${ENS_ROOT}/gdas.t${gHH}z.atmf003s.mem
    else
      ENSEMBLE_FILE_mem_p1=${ENS_ROOT}/sfg_${gdate}_fhr09s_mem
      ENSEMBLE_FILE_mem_m1=${ENS_ROOT}/sfg_${gdate}_fhr03s_mem
    fi
  fi
fi

# The following two only apply when if_observer = Yes, i.e. run observation operator for EnKF
# no_member     number of ensemble members
# BK_FILE_mem   path and base for ensemble members
no_member=20
BK_FILE_mem=${BK_ROOT}/wrfarw.mem

#####################################################
# Users should NOT make changes after this point
#####################################################

BYTE_ORDER=Big_Endian

##################################################################################
# Check GSI needed environment variables are defined and exist

# Make sure LABELI is defined and in the correct format
if [ ! "${LABELI}" ]; then
  echo "ERROR: \$LABELI is not defined!"
  exit 1
fi

# Make sure EXPDIR is defined and exists
if [ ! "${EXPDIR}" ]; then
  echo "ERROR: \$EXPDIR is not defined!"
  exit 1
fi

# Make sure the background file exists
if [ ! -r "${BK_FILE}" ]; then
  echo "ERROR: ${BK_FILE} does not exist!"
  exit 1
fi

# Make sure OBS_ROOT is defined and exists
if [ ! "${OBS_ROOT}" ]; then
  echo "ERROR: \$OBS_ROOT is not defined!"
  exit 1
fi
if [ ! -d "${OBS_ROOT}" ]; then
  echo "ERROR: OBS_ROOT directory '${OBS_ROOT}' does not exist!"
  exit 1
fi

# Set the path to the GSI static files
if [ ! "${FIX_ROOT}" ]; then
  echo "ERROR: \$FIX_ROOT is not defined!"
  exit 1
fi
if [ ! -d "${FIX_ROOT}" ]; then
  echo "ERROR: fix directory '${FIX_ROOT}' does not exist!"
  exit 1
fi

# Set the path to the CRTM coefficients
if [ ! "${CRTM_ROOT}" ]; then
  echo "ERROR: \$CRTM_ROOT is not defined!"
  exit 1
fi
if [ ! -d "${CRTM_ROOT}" ]; then
  echo "ERROR: fix directory '${CRTM_ROOT}' does not exist!"
  exit 1
fi


# Make sure the GSI executable exists
if [ ! -x "${GSI_EXE}" ]; then
  echo "ERROR: ${GSI_EXE} does not exist!"
  exit 1
fi

##################################################################################
# Create the ram work directory and cd into it

workdir=${EXPDIR}
echo " Create working directory:" ${workdir}

if [ -d "${workdir}" ]; then
  rm -rf ${workdir}
fi
mkdir -p ${workdir}/logs
cd ${workdir}


##################################################################################

echo " Copy GSI executable, background file, and link observation bufr to working directory"

# Save a copy of the GSI executable in the workdir
#cp ${GSI_EXE} gsi.x

# Bring over background field (it's modified by GSI so we can't link to it)
cp ${BK_FILE} ./wrf_inout
if [ ${if_4DEnVar} = Yes ] ; then
  cp ${BK_FILE_P1} ./wrf_inou3
  cp ${BK_FILE_M1} ./wrf_inou1
fi

if [ $chem -eq 1 ]; then
  # Link to the observation data
  if [ ${obs_type} = MODISAOD ] ; then
   ln -s ${PREPBUFRchem} ./modisbufr
   ln -s ${PREPBUFRchemTXT} ./modis.data
  fi
  if [ ${obs_type} = VIIRSAOD ] ; then                              # adicionado
   ln -s ${PREPBUFRchemVIIRS} ./viirsbufr                           # adicionado
   ln -s ${PREPBUFRchemTXTVIIRS} ./viirs.data                       # adicionado
  fi                                                                # adicionado
  if [ ${obs_type} = PM25 ] ; then
   ln -s ${PREPBUFRchem} ./pm25bufr
  fi
 else
  # Link to the prepbufr data
  ln -s ${PREPBUFR} ./prepbufr
fi

# ln -s ${OBS_ROOT}/gdas1.t${HH}z.sptrmm.tm00.bufr_d tmirrbufr
# Link to the radiance data
srcobsfile[1]=${OBS_ROOT}/gdas.t${HH}z.satwnd.tm00.bufr_d
#srcobsfile[1]=${OBS_ROOT}/cptec.SATWND.20220516.t12z.prepbufr.qc
gsiobsfile[1]=satwndbufr
srcobsfile[2]=${OBS_ROOT}/gdas1.t${HH}z.1bamua.tm00.bufr_d
gsiobsfile[2]=amsuabufr
srcobsfile[3]=${OBS_ROOT}/gdas1.t${HH}z.1bhrs4.tm00.bufr_d
gsiobsfile[3]=hirs4bufr
srcobsfile[4]=${OBS_ROOT}/gdas1.t${HH}z.1bmhs.tm00.bufr_d
gsiobsfile[4]=mhsbufr
srcobsfile[5]=${OBS_ROOT}/gdas1.t${HH}z.1bamub.tm00.bufr_d
gsiobsfile[5]=amsubbufr
srcobsfile[6]=${OBS_ROOT}/gdas1.t${HH}z.ssmisu.tm00.bufr_d
gsiobsfile[6]=ssmirrbufr
# srcobsfile[7]=${OBS_ROOT}/gdas1.t${HH}z.airsev.tm00.bufr_d
gsiobsfile[7]=airsbufr
srcobsfile[8]=${OBS_ROOT}/gdas1.t${HH}z.sevcsr.tm00.bufr_d
gsiobsfile[8]=seviribufr
srcobsfile[9]=${OBS_ROOT}/gdas1.t${HH}z.iasidb.tm00.bufr_d
gsiobsfile[9]=iasibufr
srcobsfile[10]=${OBS_ROOT}/gdas1.t${HH}z.gpsro.tm00.bufr_d
gsiobsfile[10]=gpsrobufr
srcobsfile[11]=${OBS_ROOT}/gdas1.t${HH}z.amsr2.tm00.bufr_d
gsiobsfile[11]=amsrebufr
srcobsfile[12]=${OBS_ROOT}/gdas1.t${HH}z.atms.tm00.bufr_d
gsiobsfile[12]=atmsbufr
srcobsfile[13]=${OBS_ROOT}/gdas1.t${HH}z.geoimr.tm00.bufr_d
gsiobsfile[13]=gimgrbufr
srcobsfile[14]=${OBS_ROOT}/gdas1.t${HH}z.gome.tm00.bufr_d
gsiobsfile[14]=gomebufr
srcobsfile[15]=${OBS_ROOT}/gdas1.t${HH}z.omi.tm00.bufr_d
gsiobsfile[15]=omibufr
srcobsfile[16]=${OBS_ROOT}/gdas1.t${HH}z.osbuv8.tm00.bufr_d
gsiobsfile[16]=sbuvbufr
srcobsfile[17]=${OBS_ROOT}/gdas1.t${HH}z.eshrs3.tm00.bufr_d
gsiobsfile[17]=hirs3bufrears
srcobsfile[18]=${OBS_ROOT}/gdas1.t${HH}z.esamua.tm00.bufr_d
gsiobsfile[18]=amsuabufrears
srcobsfile[19]=${OBS_ROOT}/gdas1.t${HH}z.esmhs.tm00.bufr_d
gsiobsfile[19]=mhsbufrears
srcobsfile[20]=${OBS_ROOT}/rap.t${HH}z.nexrad.tm00.bufr_d
gsiobsfile[20]=l2rwbufr
srcobsfile[21]=${OBS_ROOT}/rap.t${HH}z.lgycld.tm00.bufr_d
gsiobsfile[21]=larcglb
srcobsfile[22]=${OBS_ROOT}/gdas1.t${HH}z.glm.tm00.bufr_d
gsiobsfile[22]=glm
ii=2
#while [[ $ii -le 21 ]]; do
echo ${srcobsfile[$ii]}
while [[ $ii -le 1 ]]; do
   if [ -r "${srcobsfile[$ii]}" ]; then
      cp ${srcobsfile[$ii]}  ${gsiobsfile[$ii]}
      echo "link source obs file ${srcobsfile[$ii]}"
   fi
   (( ii = $ii + 1 ))
done

##################################################################################

ifhyb=.false.
if [ ${if_hybrid} = Yes ] ; then
  ls ${ENSEMBLE_FILE_mem}* > filelist02
  if [ ${if_4DEnVar} = Yes ] ; then
    ls ${ENSEMBLE_FILE_mem_p1}* > filelist03
    ls ${ENSEMBLE_FILE_mem_m1}* > filelist01
  fi

  nummem=`more filelist02 | wc -l`
  nummem=$((nummem -3 ))

  if [[ ${nummem} -ge 5 ]]; then
    ifhyb=.true.
    ${ECHO} " GSI hybrid uses ${ENSEMBLE_FILE_mem} with n_ens=${nummem}"
  fi
fi
if4d=.false.
if [[ ${ifhyb} = .true. && ${if_4DEnVar} = Yes ]] ; then
  if4d=.true.
fi

##################################################################################

echo " Copy fixed files and link CRTM coefficient files to working directory"

# Set fixed files
#   berror   = forecast model background error statistics
#   specoef  = CRTM spectral coefficients
#   trncoef  = CRTM transmittance coefficients
#   emiscoef = CRTM coefficients for IR sea surface emissivity model
#   aerocoef = CRTM coefficients for aerosol effects
#   cldcoef  = CRTM coefficients for cloud effects
#   satinfo  = text file with information about assimilation of brightness temperatures
#   satangl  = angle dependent bias correction file (fixed in time)
#   pcpinfo  = text file with information about assimilation of prepcipitation rates
#   ozinfo   = text file with information about assimilation of ozone data
#   errtable = text file with obs error for conventional data (regional only)
#   convinfo = text file with information about assimilation of conventional data
#   lightinfo= text file with information about assimilation of GLM lightning data
#   bufrtable= text file ONLY needed for single obs test (oneobstest=.true.)
#   bftab_sst= bufr table for sst ONLY needed for sst retrieval (retrieval=.true.)

if [ ${bkcv_option} = GLOBAL ] ; then
  echo ' Use global background error covariance'
  BERROR=${FIX_ROOT}/${BYTE_ORDER}/nam_glb_berror.f77.gcv
  OBERROR=${FIX_ROOT}/prepobs_errtable.global
  if [ ${bk_core} = NMM ] ; then
     ANAVINFO=${FIX_ROOT}/anavinfo_ndas_netcdf_glbe
  fi
  if [ ${bk_core} = ARW ] ; then
    ANAVINFO=${FIX_ROOT}/anavinfo_arw_netcdf_glbe
  fi
  if [ ${bk_core} = NMMB ] ; then
    ANAVINFO=${FIX_ROOT}/anavinfo_nems_nmmb_glb
  fi
else
  if [ $chem -eq 1 ]; then
   if [ ${bk_core} = WRFCHEM_GOCART ] ; then
    BERROR=${FIX_ROOT}/${BYTE_ORDER}/wrf_chem_berror_big_endian
    BERROR_CHEM=${FIX_ROOT}/${BYTE_ORDER}/wrf_chem_berror_big_endian
    ANAVINFO=${FIX_ROOT}/anavinfo_wrfchem_gocart
   fi
   if [ ${bk_core} = WRFCHEM_PM25 ] ; then
    BERROR=${FIX_ROOT}/${BYTE_ORDER}/wrf_chem_berror_big_endian
    BERROR_CHEM=${FIX_ROOT}/${BYTE_ORDER}/wrf_chem_berror_big_endian
    ANAVINFO=${FIX_ROOT}/anavinfo_wrfchem_pm25
   fi
  else
   echo ' Use NAM background error covariance'
   BERROR=${FIX_ROOT}/${BYTE_ORDER}/nam_nmmstat_na.gcv
   if [ ${bk_core} = NMM ] ; then
      ANAVINFO=${FIX_ROOT}/anavinfo_ndas_netcdf
   fi
   if [ ${bk_core} = ARW ] ; then
      ANAVINFO=${FIX_ROOT}/anavinfo_arw_netcdf
   fi
   if [ ${bk_core} = NMMB ] ; then
      ANAVINFO=${FIX_ROOT}/anavinfo_nems_nmmb
   fi
  fi
  OBERROR=${FIX_ROOT}/nam_errtable.r3dv
fi

AEROINFO=${FIX_ROOT}/aeroinfo_aod.txt
SATANGL=${FIX_ROOT}/global_satangbias.txt
SATINFO=${FIX_ROOT}/global_satinfo.txt
CONVINFO=${FIX_ROOT}/global_convinfo.txt
OZINFO=${FIX_ROOT}/global_ozinfo.txt
PCPINFO=${FIX_ROOT}/global_pcpinfo.txt
LIGHTINFO=${FIX_ROOT}/global_lightinfo.txt

#  copy Fixed fields to working directory
cp $ANAVINFO anavinfo
cp $BERROR   berror_stats
cp $SATANGL  satbias_angle
cp $SATINFO  satinfo
cp $CONVINFO convinfo
cp $OZINFO   ozinfo
cp $PCPINFO  pcpinfo
# cp $LIGHTINFO lightinfo
cp $OBERROR  errtable
CRTM_ROOT_ORDER=${CRTM_ROOT}/${BYTE_ORDER}
if [ $chem -eq 1 ]; then
 cp $BERROR_CHEM   berror_stats_chem
 cp $AEROINFO aeroinfo
 for file in `awk '{if($1!~"!"){print $1}}' ./aeroinfo | sort | uniq` ; do
  file=$(echo $file |  cut -c 3-)
  ln -s ${CRTM_ROOT_ORDER}/v.${file}.SpcCoeff.bin ./v.${file}.SpcCoeff.bin
  ln -s ${CRTM_ROOT_ORDER}/${file}.TauCoeff.bin ./v.${file}.TauCoeff.bin
  done
fi
#
#    # CRTM Spectral and Transmittance coefficients
emiscoef_IRwater=${CRTM_ROOT_ORDER}/Nalli.IRwater.EmisCoeff.bin
emiscoef_IRice=${CRTM_ROOT_ORDER}/NPOESS.IRice.EmisCoeff.bin
emiscoef_IRland=${CRTM_ROOT_ORDER}/NPOESS.IRland.EmisCoeff.bin
emiscoef_IRsnow=${CRTM_ROOT_ORDER}/NPOESS.IRsnow.EmisCoeff.bin
emiscoef_VISice=${CRTM_ROOT_ORDER}/NPOESS.VISice.EmisCoeff.bin
emiscoef_VISland=${CRTM_ROOT_ORDER}/NPOESS.VISland.EmisCoeff.bin
emiscoef_VISsnow=${CRTM_ROOT_ORDER}/NPOESS.VISsnow.EmisCoeff.bin
emiscoef_VISwater=${CRTM_ROOT_ORDER}/NPOESS.VISwater.EmisCoeff.bin
emiscoef_MWwater=${CRTM_ROOT_ORDER}/FASTEM6.MWwater.EmisCoeff.bin
aercoef=${CRTM_ROOT_ORDER}/AerosolCoeff.bin
cldcoef=${CRTM_ROOT_ORDER}/CloudCoeff.bin

ln -s $emiscoef_IRwater ./Nalli.IRwater.EmisCoeff.bin
ln -s $emiscoef_IRice ./NPOESS.IRice.EmisCoeff.bin
ln -s $emiscoef_IRsnow ./NPOESS.IRsnow.EmisCoeff.bin
ln -s $emiscoef_IRland ./NPOESS.IRland.EmisCoeff.bin
ln -s $emiscoef_VISice ./NPOESS.VISice.EmisCoeff.bin
ln -s $emiscoef_VISland ./NPOESS.VISland.EmisCoeff.bin
ln -s $emiscoef_VISsnow ./NPOESS.VISsnow.EmisCoeff.bin
ln -s $emiscoef_VISwater ./NPOESS.VISwater.EmisCoeff.bin
ln -s $emiscoef_MWwater ./FASTEM6.MWwater.EmisCoeff.bin
ln -s $aercoef  ./AerosolCoeff.bin
ln -s $cldcoef  ./CloudCoeff.bin

# Copy CRTM coefficient files based on entries in satinfo file
#for file in `awk '{if($1!~"!"){print $1}}' ./satinfo | sort | uniq` ;do
#   ln -s ${CRTM_ROOT_ORDER}/${file}.SpcCoeff.bin ./
#   ln -s ${CRTM_ROOT_ORDER}/${file}.TauCoeff.bin ./
#done

# Only need this file for single obs test
 bufrtable=${FIX_ROOT}/prepobs_prep.bufrtable
 cp $bufrtable ./prepobs_prep.bufrtable

# for satellite bias correction
# Users may need to use their own satbias files for correct bias correction
cp ${FIX_ROOT}/comgsi_satbias_in ./satbias_in
cp ${FIX_ROOT}/comgsi_satbias_pc_in ./satbias_pc_in

#
##################################################################################
# Set some parameters for use by the GSI executable and to build the namelist
echo " Build the namelist "

# default is NAM
#   as_op='1.0,1.0,0.5 ,0.7,0.7,0.5,1.0,1.0,'
vs_op='1.0,'
hzscl_op='0.373,0.746,1.50,'
if [ ${bkcv_option} = GLOBAL ] ; then
#   as_op='0.6,0.6,0.75,0.75,0.75,0.75,1.0,1.0'
   vs_op='0.7,'
   hzscl_op='1.7,0.8,0.5,'
fi
if [ ${bk_core} = NMMB ] ; then
   vs_op='0.6,'
fi

# default is NMM
   bk_core_arw='.false.'
   bk_core_nmm='.true.'
   bk_core_nmmb='.false.'
   bk_if_netcdf='.true.'
if [ ${bk_core} = ARW ] ; then
   bk_core_arw='.true.'
   bk_core_nmm='.false.'
   bk_core_nmmb='.false.'
   bk_if_netcdf='.true.'
fi
if [ ${bk_core} = NMMB ] ; then
   bk_core_arw='.false.'
   bk_core_nmm='.false.'
   bk_core_nmmb='.true.'
   bk_if_netcdf='.false.'
fi
if [ ${bk_core} = WRFCHEM_GOCART ] ; then
   bk_core_arw='.true.'
   bk_if_netcdf='.true.'
   bk_core_cmaq='.false.'
   bk_wrf_pm2_5='.false.'
   bk_laeroana_gocart='.true.'
fi
if [ ${bk_core} = WRFCHEM_PM25 ] ; then
   bk_core_arw='.true.'
   bk_if_netcdf='.true.'
   bk_core_cmaq='.false.'
   bk_wrf_pm2_5='.true.'
   bk_laeroana_gocart='.false.'
fi

if [ ${if_observer} = Yes ] ; then
  nummiter=0
  if_read_obs_save='.true.'
  if_read_obs_skip='.false.'
else
  nummiter=2
  if_read_obs_save='.false.'
  if_read_obs_skip='.false.'
fi

# Build the GSI namelist on-the-fly
. $GSI_NAMELIST

# modify the anavinfo vertical levels based on wrf_inout for WRF ARW and NMM
if [ ${bk_core} = ARW ] || [ ${bk_core} = NMM ] ; then
bklevels=`ncdump -h wrf_inout | grep "bottom_top =" | awk '{print $3}' `
bklevels_stag=`ncdump -h wrf_inout | grep "bottom_top_stag =" | awk '{print $3}' `
anavlevels=`cat anavinfo | grep ' sf ' | tail -1 | awk '{print $2}' `  # levels of sf, vp, u, v, t, etc
anavlevels_stag=`cat anavinfo | grep ' prse ' | tail -1 | awk '{print $2}' `  # levels of prse
sed -i 's/ '$anavlevels'/ '$bklevels'/g' anavinfo
sed -i 's/ '$anavlevels_stag'/ '$bklevels_stag'/g' anavinfo
fi

#
###################################################
#  run  GSI
###################################################
echo ' Run GSI with' ${bk_core} 'background'


NTASKS=12 #96
JNAME="GSI"

cat > GSI.sh <<EOF0
#!/bin/bash

#SBATCH --time=00:15:00
#SBATCH --ntasks=$NTASKS
#SBATCH --job-name=$JNAME
#SBATCH --partition=PESQ1
#SBATCH --exclusive

ulimit -s unlimited

cd ${EXPDIR}
echo "########################################"
echo "ESTOU AQUI \$(pwd)"

date

echo  "STARTING AT \`date\` "
Start=\`date +%s.%N\`
echo \$Start > Timing.gsi

time mpirun -np ${NTASKS} ${GSI_EXE} &> ${LOGDIR}/log.gsi

##################################################################
#  run time error check
##################################################################
error=\$?

if [ \${error} -ne 0 ]; then
  echo "ERROR: GSI crashed  Exit status=\${error}"
  exit \${error}
fi

#
##################################################################
#
#   GSI updating satbias_in
#
# GSI updating satbias_in (only for cycling assimilation)

# Copy the output to more understandable names
ln -s wrf_inout   wrfanl.${LABELI}
ln -s fort.201    fit_p1.${LABELI}
ln -s fort.202    fit_w1.${LABELI}
ln -s fort.203    fit_t1.${LABELI}
ln -s fort.204    fit_q1.${LABELI}
ln -s fort.207    fit_rad1.${LABELI}

# Loop over first and last outer loops to generate innovation
# diagnostic files for indicated observation types (groups)
#
# NOTE:  Since we set miter=2 in GSI namelist SETUP, outer
#        loop 03 will contain innovations with respect to
#        the analysis.  Creation of o-a innovation files
#        is triggered by write_diag(3)=.true.  The setting
#        write_diag(1)=.true. turns on creation of o-g
#        innovation files.
#

loops="01 03"
for loop in \$loops; do

case \$loop in
  01) string=ges;;
  03) string=anl;;
   *) string=\$loop;;
esac

 listall=\`ls pe*_?? | cut -f2 -d"." | awk '{print substr(\$0, 0, length(\$0)-3)}' | sort | uniq \`

   for type in \$listall; do
      count=\`ls pe*\${type}*_\${loop}* | wc -l\`
      if [[ \$count -gt 0 ]]; then
         cat pe*\${type}_\${loop}* > diag_\${type}_\${string}.${LABELI}
      fi
   done
done

#  Clean working directory to save only important files
ls -l * > list_run_directory
if [[ ${if_clean} = clean  &&  ${if_observer} != Yes ]]; then
  echo ' Clean working directory after GSI run'
  rm -f *Coeff.bin     # all CRTM coefficient files
  rm -f pe0*           # diag files on each processor
  rm -f obs_input.*    # observation middle files
  rm -f siganl sigf0?  # background middle files
  rm -f fsize_*        # delete temperal file for bufr size
fi

exit 0
EOF0

chmod +x ${EXPDIR}/GSI.sh

sbatch ${EXPDIR}/GSI.sh


#
#
#################################################
# start to calculate diag files for each member
#################################################
#
if [ ${if_observer} = Yes ] ; then
  string=ges
  for type in $listall; do
    count=0
    if [[ -f diag_${type}_${string}.${LABELI} ]]; then
       mv diag_${type}_${string}.${LABELI} diag_${type}_${string}.ensmean
    fi
  done
  mv wrf_inout wrf_inout_ensmean

# Build the GSI namelist on-the-fly for each member
  nummiter=0
  if_read_obs_save='.false.'
  if_read_obs_skip='.true.'
. $GSI_NAMELIST

# Loop through each member
  loop="01"
  ensmem=1
  while [[ $ensmem -le $no_member ]];do

     rm pe0*

     print "\$ensmem is $ensmem"
     ensmemid=`printf %3.3i $ensmem`

# get new background for each member
     if [[ -f wrf_inout ]]; then
       rm wrf_inout
     fi

     BK_FILE=${BK_FILE_mem}${ensmemid}
     echo $BK_FILE
     ln -s $BK_FILE wrf_inout

#  run  GSI
     echo ' Run GSI with' ${bk_core} 'for member ', ${ensmemid}

     case $ARCH in
        'IBM_LSF')
           ${RUN_COMMAND} ./gsi.x < gsiparm.anl > stdout_mem${ensmemid} 2>&1  ;;

        * )
           ${RUN_COMMAND} ./gsi.x > stdout_mem${ensmemid} 2>&1 ;;
     esac

     NTASKS=1
     JNAME="GSI-"${ensmemid}

     cat > GSI${ensmemid}.sh <<EOF0
     #!/bin/bash

     #SBATCH --time=00:10:00
     #SBATCH --ntasks=$NTASKS
     #SBATCH --job-name=$JNAME

     ulimit -s unlimited

     echo $HDF5
     echo $LD_LIBRARY_PATH

     cd ${EXPDIR}

     date

     echo  "STARTING AT \`date\` "
     Start=\`date +%s.%N\`
     echo \$Start > Timing.gsi

     time mpirun -np ${NTASKS} ${GSI_EXE} &> ${LOGDIR}/log${ensmemid}.gsi

     exit 0
EOF0

     chmod +x GSI${ensmemid}.sh

     sbatch ./GSI${ensmemid}.sh

#  run time error check and save run time file status
     error=$?

     if [ ${error} -ne 0 ]; then
       echo "ERROR: ${GSI} crashed for member ${ensmemid} Exit status=${error}"
       exit ${error}
     fi

     ls -l * > list_run_directory_mem${ensmemid}

# generate diag files

     for type in $listall; do
           count=`ls pe*${type}_${loop}* | wc -l`
        if [[ $count -gt 0 ]]; then
           cat pe*${type}_${loop}* > diag_${type}_${string}.mem${ensmemid}
        fi
     done

# next member
     (( ensmem += 1 ))

  done

fi

exit 0
