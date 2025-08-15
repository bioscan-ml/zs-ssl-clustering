Zero-shot SSL clustering (BIOSCAN-5M)
=====================================

This modified version of the original
zero-shot SSL clustering [repository](https://github.com/scottclowe/zs-ssl-clustering)
([paper](https://arxiv.org/abs/2406.02465))
adds the experiments included in the
[BIOSCAN-5M paper](https://arxiv.org/abs/2406.12723).

The changes implemented in this version of the code add support for:
- loading the BIOSCAN-5M dataset ([datasets/bioscan5m.py](https://github.com/bioscan-ml/zs-ssl-clustering/blob/master/zs_ssl_clustering/datasets/bioscan5m.py), and [datasets/api.py](https://github.com/bioscan-ml/zs-ssl-clustering/blob/master/zs_ssl_clustering/datasets/api.py#L553-L596))
- handling multimodal datasets by merging embeddings from different modalities together before clustering, optionally with independent normalization beforehand ([io.py](https://github.com/bioscan-ml/zs-ssl-clustering/blob/master/zs_ssl_clustering/io.py#L89-L177))


Installation
------------

To download the code, install it and its dependencies, run the following code:
```bash
git clone git@github.com:bioscan-ml/zs-ssl-clustering.git
cd zs-ssl-clustering
pip install -e .
```


Execution
---------

To use the code, you must first generate cached embeddings for the dataset using the module [zs_ssl_clustering/embed.py](https://github.com/bioscan-ml/zs-ssl-clustering/blob/master/zs_ssl_clustering/embed.py), e.g.
```bash
python zs_ssl_clustering/embed.py \
    --dataset=bioscan5m \
    --modality=image \
    --partition=test \
    --model=dino_vitb16 \
    --data-dir=PATH_TO_DIR_PREDOWNLOADED_DATASET
```
For more details on parameters for the embedding script, run `python zs_ssl_clustering/embed.py --help`.

You can cluster the cached embeddings using the module [zs_ssl_clustering/cluster.py](https://github.com/bioscan-ml/zs-ssl-clustering/blob/master/zs_ssl_clustering/cluster.py), e.g.
```bash
python zs_ssl_clustering/cluster.py \
    --dataset=bioscan5m \
    --modality=image \
    --partition=test \
    --model=dino_vitb16 \
    --dim-reducer-man=UMAP --ndim-reduced-man=50 \
    --clusterer=AgglomerativeClustering \
    --log-wandb --wandb-entity=YOUR_WANDB_ENTITY
```
If `--log-wandb` is supplied, clustering results (e.g. AMI) will be saved to your [Weights & Biases](https://wandb.ai/) dashboard.
Generated cluster prediction labels will also be saved to `./y_pred/` in .npz format, which can be used for downstream analysis.
For more details on parameters for the clustering script, run `python zs_ssl_clustering/cluster.py --help`.


Replicating the experiments in the BIOSCAN-5M paper
---------------------------------------------------

To reproduce the experiments shown in the paper, embeddings need to first be created and cached. Afterwards, the clusterer will run on the cached embeddings.

1. First create cached embeddings for the dataset using the pretrained models.

**Image models**: To generate all embeddings used in our experiments, you will need to generate embeddings on both test partitions for each image encoder model as follows. The generated embeddings will take around 5GB of storage space.
We use `--dataset=bioscan5m` to create embeddings for the full dataset and `--dataset=bioscan5m_per-barcode-dedupNs-660` to generate embeddings for the dataset after deduplicating repeated barcodes (described in Appendix B.2 of the [BIOSCAN-5M paper](https://arxiv.org/abs/2406.02465)).
```bash
DATA_DIR="PATH_TO_DIR_PREDOWNLOADED_DATASET"
for DATASET in bioscan5m bioscan5m_per-barcode-dedupNs-660;
do
    for PARTITION in test test_unseen;
    do
        for MODEL in \
            resnet50 dino_resnet50 mocov3_resnet50 vicreg_resnet50 \
            vitb16 timm_vit_base_patch16_224.mae dino_vitb16 mocov3_vit_base \
            mae_pretrain_vit_base_global mae_pretrain_vit_base_cls \
            clip_RN50 clip_vitb16;
        do
            python zs_ssl_clustering/embed.py \
                --dataset="$DATASET" --model="$MODEL" --partition="$PARTITION" \
                --modality="image" \
                --data-dir="$DATA_DIR"
        done
    done
done
```

**DNA models**:
Code to generate the embeddings as used in the paper is not currently available, but pre-generated embeddings as used for the experiments in the paper can be downloaded from [GDrive](https://drive.google.com/file/d/18SMBe0FBVpcKOU-erosmW__yRKly61nm/).
These embeddings should be downloaded, extracted, and placed in the `./embeddings/` directory.


2. Create cluster predictions on both the test and test_unseen partitions, using the cached embeddings.

To generate single-model cluster predictions using image embeddings, run the following code:
```bash
for DATASET in bioscan5m bioscan5m_per-barcode-dedupNs-660;
do
    for MODEL in \
        resnet50 dino_resnet50 mocov3_resnet50 vicreg_resnet50 \
        vitb16 timm_vit_base_patch16_224.mae dino_vitb16 mocov3_vit_base \
        mae_pretrain_vit_base_global mae_pretrain_vit_base_cls \
        clip_RN50 clip_vitb16;
    do
        python zs_ssl_clustering/cluster.py \
            --dataset="$DATASET" --partition test test_unseen \
            --modality=image --model="$MODEL" \
            --dim-reducer-man=UMAP --ndim-reduced-man=50 \
            --clusterer=AgglomerativeClustering \
            --log-wandb --wandb-entity=YOUR_WANDB_ENTITY
    done
done
```
And for barcode DNA embeddings, run the following code:
```bash
for DATASET in bioscan5m bioscan5m_per-barcode-dedupNs-660;
do
    for DNA_MODEL in dnabert-s dnabert-2 hyenadna NucleotideTransformer barcodebert;
    do
        python zs_ssl_clustering/cluster.py \
            --dataset="$DATASET" --partition test test_unseen \
            --modality=dna --dna-model="$DNA_MODEL" \
            --dim-reducer-man=UMAP --ndim-reduced-man=50 \
            --clusterer=AgglomerativeClustering \
            --log-wandb --wandb-entity=YOUR_WANDB_ENTITY
    done
done
```

To generate multi-modal cluster predicitons, using a concatenation of z-scored image and barcode DNA embeddings, run the following code:
```bash
for DATASET in bioscan5m_per-barcode-dedupNs-660;
do
    for MODEL in \
        resnet50 dino_resnet50 mocov3_resnet50 vicreg_resnet50 \
        vitb16 timm_vit_base_patch16_224.mae dino_vitb16 mocov3_vit_base \
        mae_pretrain_vit_base_global mae_pretrain_vit_base_cls \
        clip_RN50 clip_vitb16;
    do
        for DNA_MODEL in dnabert-s dnabert-2 hyenadna NucleotideTransformer barcodebert;
        do
            python zs_ssl_clustering/cluster.py \
                --dataset="$DATASET" --partition test test_unseen \
                --modality image dna --prenorm=elementwise_zscore \
                --model="$MODEL" --dna-model="$DNA_MODEL" \
                --dim-reducer-man=UMAP --ndim-reduced-man=50 \
                --clusterer=AgglomerativeClustering \
                --log-wandb --wandb-entity=YOUR_WANDB_ENTITY
        done
    done
done
```

With the argument `--log-wandb` supplied, clustering results (e.g. AMI) will be saved to your [Weights & Biases](https://wandb.ai/) dashboard.
Cluster prediction labels will be saved in the directory `./y_pred/` in .npz format, which can be used for downstream analysis.


Citation
--------

If you find this work useful, please consider citing the corresponding papers:
```bibtex
@inproceedings{gharaee2024bioscan5m,
    title={{BIOSCAN-5M}: A Multimodal Dataset for Insect Biodiversity},
    booktitle={Advances in Neural Information Processing Systems},
    author={Zahra Gharaee and Scott C. Lowe and ZeMing Gong and Pablo Millan Arias
        and Nicholas Pellegrino and Austin T. Wang and Joakim Bruslund Haurum
        and Iuliia Zarubiieva and Lila Kari and Dirk Steinke and Graham W. Taylor
        and Paul Fieguth and Angel X. Chang
    },
    editor={A. Globerson and L. Mackey and D. Belgrave and A. Fan and U. Paquet and J. Tomczak and C. Zhang},
    pages={36285--36313},
    publisher={Curran Associates, Inc.},
    year={2024},
    volume={37},
    url={https://proceedings.neurips.cc/paper_files/paper/2024/file/3fdbb472813041c9ecef04c20c2b1e5a-Paper-Datasets_and_Benchmarks_Track.pdf},
}

@misc{zsc,
    title={An Empirical Study into Clustering of Unseen Datasets with Self-Supervised Encoders},
    author={Scott C. Lowe and Joakim Bruslund Haurum
        and Sageev Oore and Thomas B. Moeslund and Graham W. Taylor
    },
    year={2024},
    eprint={2406.02465},
    archivePrefix={arXiv},
    primaryClass={cs.LG},
    url={https://arxiv.org/abs/2406.02465},
    doi={10.48550/arxiv.2406.02465},
}
```
