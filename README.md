# moflow_plus

Please refer to our paper:

Zang, Chengxi, and Fei Wang. "MoFlow: an invertible flow model for generating molecular graphs." In Proceedings of the 26th ACM SIGKDD International Conference on Knowledge Discovery & Data Mining, pp. 617-626. 2020.

```
https://arxiv.org/abs/2006.10137
```
```
@inproceedings{zang2020moflow,
  title={MoFlow: an invertible flow model for generating molecular graphs},
  author={Zang, Chengxi and Wang, Fei},
  booktitle={Proceedings of the 26th ACM SIGKDD International Conference on Knowledge Discovery \& Data Mining},
  pages={617--626},
  year={2020}
}
```

## 0. Install Libs:
```
conda create --name moflow python pandas matplotlib    (conda 4.6.7, python 3.8.5, pandas 1.1.2, matplotlib  3.3.2)
conda activate moflow
conda install pytorch torchvision cudatoolkit=10.2 -c pytorch (pytorch 1.6.0, torchvision 0.7.0)
conda install rdkit  (rdkit 2020.03.6)
conda install orderedset  (orderset 2.0.3)
conda install tabulate  (tabulate 0.8.7)
conda install networkx  (networkx 2.5)
conda install scipy  (scipy 1.5.0)
conda install seaborn  (seaborn 0.11.0)
pip install cairosvg (cairosvg 2.4.2)
pip install tqdm  (tqdm 4.50.0)
```
To clone code from this project, say
```
git clone https://github.com/calvin-zcx/moflow.git moflow
```
## 1. Data Preprocessing
To generate molecular graphs from SMILES strings
```
cd data
python data_preprocess.py --data_name qm9
```

## 2. Model Training
#### Training model for QM9 dataset:
```
cd mflow
python train_model.py --data_name qm9  --batch_size 256  --max_epochs 80 --gpu 0  --debug True  --save_dir=results/qm9_64gnn_128-64lin_1-1mask_0d6noise_convlu1  --b_n_flow 10  --b_hidden_ch 216,216  --a_n_flow 27 --a_hidden_gnn 64  --a_hidden_lin 128,64  --mask_row_size_list 1 --mask_row_stride_list 1 --noise_scale 0.6 --b_conv_lu 1  2>&1 | tee qm9_64gnn_128-64lin_1-1mask_0d6noise_convlu1.log
```
## 3. Model Testing

### 3.1-Experiment: Random generation  

#### Random Generation from sampling from latent space, QM9 model
10000 samples * 5 times:
```
python generate.py --model_dir results/qm9_64gnn_128-64lin_1-1mask_0d6noise_convlu1 -snapshot model_snapshot_epoch_80 --gpu 0 --data_name qm9 --hyperparams-path moflow-params.json --batch-size 10000 --temperature 0.85 --delta 0.05 --n_experiments 5 --save_fig false --correct_validity true 2>&1 | tee qm9_random_generation.log
```
#####  Results
```
novelty: 99.980%, abs novelty: 98.330%
validity: mean=100.00%, sd=0.00%, vals=[100.0, 100.0, 100.0, 100.0, 100.0]
novelty: mean=99.99%, sd=0.01%, vals=[99.98983533238463, 99.9898363654843, 100.0, 100.0, 99.97966446365022]
uniqueness: mean=98.26%, sd=0.24%, vals=[98.38, 98.39, 97.78, 98.42, 98.35000000000001]
abs_novelty: mean=98.26%, sd=0.24%, vals=[98.37, 98.38, 97.78, 98.42, 98.33]
abs_uniqueness: mean=98.26%, sd=0.24%, vals=[98.38, 98.39, 97.78, 98.42, 98.35000000000001]
Task random generation done! Time 185.09 seconds, Data: Wed May 18 17:44:15 2022
# Above is just one random result. Tuning:
    --batch-size for the number of  mols to be generated
    --temperature for different generation results, 
    --correct_validity false for results without correction
    --save_fig true for figures of generated mols, set batch-size a resoanble number for dump figures 
# more details see parameter configuration in generate.py
# Output details are in qm9_random_generation.log
```

### 3.2-Experiment: Molecular optimization & constrained optimization
#### Optimizing QM9 w.r.t plogp property
#### Training an additional MLP from latent space to plogp property
```
python optimize_property.py -snapshot model_snapshot_epoch_125  --hyperparams_path moflow-params.json --batch_size 256 --model_dir results/qm9_64gnn_128-64lin_1-1mask_0d6noise_convlu1   --gpu 0 --max_epochs 3  --weight_decay 1e-3  --data_name qm9  --hidden 16,  --temperature 1.0  --property_name plogp 2>&1 | tee  training_optimize_qm9_plogp.log
# Output: a molecular property prediction model for optimization, say named as plogp_model.pt
# e.g. saving qed regression model to: results/qm9_64gnn_128-64lin_1-1mask_0d6noise_convlu1/plogp_model.pt
# Train and save model done! Time 477.87 seconds 
# Can tune:
#         --max_epochs 3  
#         --weight_decay 1e-3  
#         --hidden 16
# etc.
```
#### To optimize existing molecules to get novel molecules with optimized plogp scores and constrained similarity
```
python optimize_property.py -snapshot model_snapshot_epoch_125  --hyperparams_path moflow-params.json --batch_size 256 --model_dir results/qm9_64gnn_128-64lin_1-1mask_0d6noise_convlu1   --gpu 0    --data_name qm9    --property_name plogp --topk 2000 --property_model_path plogp_model.pt   --consopt  --sim_cutoff 0 2>&1 | tee  qm9_constrain_optimize_plogp.log
# Input: --property_model_path qed_model.pt or plogp_model.pt is the regression model
         --sim_cutoff 0 (or 0.2, 0.4 etc for similarity)
         --topk 2000 (choose first 2000 molecules with worset property values for improving)
# Output: 
# Using qed_model.pt for optimizing plogp with 
# Because qed and plogp have some correlations, here we use both qed/plogp model for 2 optimization tasks
# --sim_cutoff 0:
#   similarity: 0.300610 +/- 0.201674 
#   Improvement:  8.612461 +/- 5.436995
#   success rate: 0.98875
# --sim_cutoff 0.2:
#   similarity:  0.434700 +/-  0.196490 
#   Improvement:  7.057115 +/-  5.041250
#   success rate: 0.9675
# --sim_cutoff 0.4:
#   similarity:  0.608440 +/-   0.177670 
#   Improvement:  4.712418 +/-   4.549682
#   success rate: 0.8575
# --sim_cutoff 0.6:
#   similarity:   0.792550 +/- 0.144577   
#   Improvement:   2.095266 +/-   2.858545  
#   success rate:  0.5825

# Using plogp_model.pt for optimizing plogp with 
# --sim_cutoff 0:
#    similarity:  0.260503 +/- 0.195945
#   Improvement:  9.238813 +/-  6.279859
#   success rate: 0.9925
# --sim_cutoff 0.2:
#    similarity:  0.425541 +/- 0.198020
#    Improvement:  7.246221 +/-  5.325543
#    success rate: 0.9575
# --sim_cutoff 0.4:
#    similarity:   0.625976 +/- 0.189293
#    Improvement:  4.504411 +/-  4.712031
#    success rate: 0.8425
# --sim_cutoff 0.6:
#    similarity:   0.810805 +/- 0.146080
#    Improvement:  1.820525 +/-  2.595302
#    success rate: 0.565
```
More configurations please refer to our codes optimize_property.py and the optimization chapter in our paper.
#### One illustration of optimizing plogp
![optimiation plogp](mflow/fig/copt.png)

### 3.3-Experiment: Diversity Measurement
```
python calc_diversity.py --csv_dir plogp_constrain_optimization.csv
```

