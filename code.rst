iScore
======

Run as interactive session on the cluster. Not sure if it is possible to submit a job to a compute node from a compute node. In other words, can we qsub the following script, which parallel processes the brains by qsub?

.. code-block:: bash

  export ANTSPATH=/U1/hunsakern/apps/ants-20160404/bin/
  PATH=${ANTSPATH}:${PATH}

Copy preprocessed (insert reference) images for template construction:

.. code-block:: bash

  mkdir -p ~/templates/iScore
  for i in $(ls ~/data/iScore/); do
  cp -v ~/compute/images/MIOS/$i/t1/t1.nii.gz ~/templates/MIOS/img_${i}.nii.gz;
  done

Template
--------

First part takes 2.5 hours on cluster:

.. code-block:: bash

  cd ~/templates/iScore/
  buildtemplateparallel.sh -d 3 -m 1x0x0 -o pt1 img*.nii.gz

Second part takes 14.5 hours on cluster:

.. code-block:: bash

  buildtemplateparallel.sh -d 3 -z /U1/hunsakern/templates/iScore/pt1template.nii.gz -o pt2 img*.nii.gz

Monitor job progress

.. code-block:: bash

  watch --interval=150 qstat

Possible way to catch errors

.. code-block:: bash

  while :
  do
  qstat | awk '$5 == "Eqw" {cmd="qmod -cj " $1; system(cmd); close(cmd)}'
  sleep 30
  done

Cook Tissue Priors
------------------

.. code-block:: bash


  #!/bin/bash

  # SET ANTSPATH
  export ANTSPATH=/U1/hunsakern/apps/ants-20160404/bin/
  C3DPATH=/U1/hunsakern/apps/c3d-1.0.0-Linux-x86_64/bin/

  # SET VARIABLES
  DATA_DIR=/U1/hunsakern/templates/iScore
  TEMPLATE_DIR=/U1/hunsakern/templates/OASIS
  INPUT_IMAGE=${DATA_DIR}/iScore_template.nii.gz

  # RUN ANTSCORTICALTHICKNESS
  ${ANTSPATH}antsCorticalThickness.sh -d 3 \
  -a $INPUT_IMAGE \
  -e ${TEMPLATE_DIR}/T_template0.nii.gz \
  -m ${TEMPLATE_DIR}/T_template0_BrainCerebellumProbabilityMask.nii.gz \
  -f ${TEMPLATE_DIR}/T_template0_BrainCerebellumRegistrationMask.nii.gz \
  -p ${TEMPLATE_DIR}/Priors2/priors%d.nii.gz \
  -o ${DATA_DIR}/antsCT/ \
  -q 1 \
  -u 1

  # COPY MASK
  cp ${DATA_DIR}/antsCT/BrainExtractionMask.nii.gz ${DATA_DIR}/iScore_template_BrainCerebellumMask.nii.gz

  # EXTRACT BRAIN IMAGE
  ${ANTSPATH}/ImageMath 3 \
  ${DATA_DIR}/iScore_template_BrainCerebellum.nii.gz \
  m \
  ${DATA_DIR}/iScore_template_BrainCerebellumMask.nii.gz \
  $INPUT_IMAGE

  # CONVERT MASK ROI TO PROBABILITY MASK
  ${ANTSPATH}/SmoothImage \
  3 \
  ${DATA_DIR}/iScore_template_BrainCerebellumMask.nii.gz \
  1 \
  ${DATA_DIR}/iScore_template_BrainCerebellumProbabilityMask.nii.gz

  # DILATE MASK IMAGE TO GENERATE EXTRACTION MASK
  ${C3DPATH}/c3d \
  ${DATA_DIR}/iScore_template_BrainCerebellumMask.nii.gz \
  -dilate 1 28x28x28vox \
  -o ${DATA_DIR}/iScore_template_BrainCerebellumExtractionMask.nii.gz

  # DILATE MASK IMAGE TO GENERATE REGISTRATION MASK
  ${C3DPATH}/c3d \
  ${DATA_DIR}/iScore_template_BrainCerebellumMask.nii.gz \
  -dilate 1 18x18x18vox \
  -o ${DATA_DIR}/iScore_template_BrainCerebellumRegistrationMask.nii.gz

  # COPY TISSUE SEGMENTATION
  cp ${DATA_DIR}/antsCT/BrainSegmentation.nii.gz ${DATA_DIR}/iScore_template_6labels.nii.gz

  # COPY TISSUE PRIORS
  cp ${DATA_DIR}/antsCT/BrainSegmentationPosteriors1.nii.gz ${DATA_DIR}/priors/priors1.nii.gz
  cp ${DATA_DIR}/antsCT/BrainSegmentationPosteriors2.nii.gz ${DATA_DIR}/priors/priors2.nii.gz
  cp ${DATA_DIR}/antsCT/BrainSegmentationPosteriors3.nii.gz ${DATA_DIR}/priors/priors3.nii.gz
  cp ${DATA_DIR}/antsCT/BrainSegmentationPosteriors4.nii.gz ${DATA_DIR}/priors/priors4.nii.gz
  cp ${DATA_DIR}/antsCT/BrainSegmentationPosteriors5.nii.gz ${DATA_DIR}/priors/priors5.nii.gz
  cp ${DATA_DIR}/antsCT/BrainSegmentationPosteriors6.nii.gz ${DATA_DIR}/priors/priors6.nii.gz

Joint Label Fusion
------------------

.. code-block:: bash

  TEMPLATE_DIR=/U1/hunsakern/templates/iScore
  ATLAS_DIR=/U1/hunsakern/templates/OASIS-TRT-20_volumes
  LABEL_DIR=/U1/hunsakern/templates/OASIS-TRT-20_DKT31_CMA_labels_v2
  cd ${TEMPLATE_DIR}
  ${ANTSPATH}/antsJointLabelFusion.sh \
  -d 3 \
  -o ${TEMPLATE_DIR}/labels/ \
  -t ${TEMPLATE_DIR}/iScore_template_BrainCerebellum.nii.gz \
  -c 1 -j 2 -y s -k 0 \
  -g ${ATLAS_DIR}/OASIS-TRT-20-1/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-1_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-2/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-2_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-3/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-3_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-4/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-4_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-5/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-5_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-6/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-6_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-7/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-7_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-8/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-8_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-9/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-9_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-10/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-10_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-11/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-11_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-12/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-12_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-13/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-13_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-14/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-14_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-15/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-15_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-16/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-16_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-17/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-17_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-18/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-18_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-19/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-19_DKT31_CMA_labels.nii.gz \
  -g ${ATLAS_DIR}/OASIS-TRT-20-20/t1weighted_brain.nii.gz -l ${LABEL_DIR}/OASIS-TRT-20-20_DKT31_CMA_labels.nii.gz

Preprocessing & ANTs Cortical Thickness
---------------------------------------

.. code-block:: bash

  #!/bin/bash

  # LOAD MODULES AND SET ENVIRONMENTAL VARIABLES
  export ANTSPATH=/U1/hunsakern/apps/ants-20160404/bin/
  PATH=${ANTSPATH}:${PATH}
  ARTHOME=/U1/hunsakern/apps/art
  export ARTHOME

  # SET VARIABLES
  SUBJ_DIR=/U1/hunsakern/data/iScore/${1}
  TEMPLATE_DIR=/U1/hunsakern/templates/iScore

  # REORIENT
  /U1/hunsakern/apps/mricron/dcm2nii -r y -g n -o ${SUBJ_DIR}/t1/ ${SUBJ_DIR}/t1/t1.nii

  # ACPC ALIGN
  /U1/hunsakern/apps/art/acpcdetect -M \
  -o ${SUBJ_DIR}/t1/acpc.nii \
  -i ${SUBJ_DIR}/t1/ft1.nii
  rm ${SUBJ_DIR}/t1/ft1.nii

  # RESAMPLE TO 1 ISOTROPIC
  /U1/hunsakern/apps/c3d-1.0.0-Linux-x86_64/bin/c3d -verbose \
  ${SUBJ_DIR}/t1/acpc.nii \
  -resample-mm 1x1x1mm \
  -o ${SUBJ_DIR}/t1/resampled.nii.gz
  rm ${SUBJ_DIR}/t1/*.ppm
  rm ${SUBJ_DIR}/t1/*.txt
  rm ${SUBJ_DIR}/t1/acpc.nii

  # N4 BIAS FIELD CORRECTION
  /U1/hunsakern/apps/ants-20160404/bin/N4BiasFieldCorrection -d 3 -v 1 \
  -i ${SUBJ_DIR}/t1/resampled.nii.gz \
  -o ${SUBJ_DIR}/t1/n4.nii.gz \
  -s 4 \
  -b [200] \
  -c [50x50x50x50,0.000001]
  rm ${SUBJ_DIR}/t1/resampled.nii.gz
  mv ${SUBJ_DIR}/t1/n4.nii.gz ${SUBJ_DIR}/t1/t1.nii.gz
  mv ${SUBJ_DIR}/t1/t1.nii ${SUBJ_DIR}/t1/orig.nii

  # ANTS CORTICAL THICKNESS
  /U1/hunsakern/apps/ants-20160404/bin/antsCorticalThickness.sh -d 3 -k 0 \
  -a ${SUBJ_DIR}/t1/t1.nii.gz \
  -e ${TEMPLATE_DIR}/iScore_template.nii.gz \
  -t ${TEMPLATE_DIR}/iScore_template_BrainCerebellum.nii.gz \
  -m ${TEMPLATE_DIR}/iScore_template_BrainCerebellumProbabilityMask.nii.gz \
  -f ${TEMPLATE_DIR}/iScore_template_BrainCerebellumRegistrationMask.nii.gz \
  -p ${TEMPLATE_DIR}/priors/priors%d.nii.gz \
  -o ${SUBJ_DIR}/antsCT/ \
  -q 1 \
  -u 1

Batch Script
------------

.. code-block:: bash

  #!/bin/bash

  for subj in $(ls /U1/hunsakern/data/iScore); do
    DIR=/U1/hunsakern/data/iScore/${subj}/antsCT
    if [ ! -e $DIR ]; then
      echo "Processing" $subj
      qsub -o /U1/hunsakern/logfiles/2016-04-19-0830/output_${subj}.txt -j y -S /bin/bash /U1/hunsakern/scripts/antsCT.sh ${subj}
      sleep 1
    else
      echo ${subj} " Participant is already done"
    fi
  done
