# ChatKBQA

##  General Setup 

### Environment Setup
```
conda create -n chatkbqa python=3.8
conda activate chatkbqa
pip install torch==1.13.1+cu117 torchvision==0.14.1+cu117 torchaudio==0.13.1 --extra-index-url https://download.pytorch.org/whl/cu117
pip install -r requirement.txt
```

###  Freebase KG Setup

Below steps are according to [Freebase Virtuoso Setup](https://github.com/dki-lab/Freebase-Setup). 
#### How to install virtuoso backend for Freebase KG.

1. Clone from `git@github.com:dki-lab/Freebase-Setup.git`:
```
cd Freebase-Setup
```

2. The latest offical data dump of [Freebase](https://developers.google.com/freebase) can be downloaded. However, in the official dump, the format of some literal types is not fully compatible with the N-Triples RDF standard (it's missing type decoration such as `^^<http://www.w3.org/2001/XMLSchema#integer>`), which may cause it to fail to load into triplestores like Virtuoso. We fixed this issue. Our processed Virtuoso DB file can be downloaded from [here](https://www.dropbox.com/s/q38g0fwx1a3lz8q/virtuoso_db.zip) or via wget (WARNING: 53G+ disk space is needed):
```
tar -zxvf virtuoso_db.zip
```

In case you'd like to load your own RDF into Virtuoso, see [here](http://vos.openlinksw.com/owiki/wiki/VOS/VirtBulkRDFLoader) for instructions. If you prefer some other triplestore, try fixing the literal format issue of Freebase using the script `fix_freebase_literal_format.py` to get the N-Triples-formatted data. 

3. Managing the Virtuoso service

We provide a wrapper script (`virtuoso.py`, adapted from [Sempre](https://github.com/percyliang/sempre)) for managing the Virtuoso service. To use it, first change the `virtuosoPath` in the script to your local Virtuoso directory. Assuming the Virtuoso db file is located in a directory named `virtuoso_db` under the same directory as the script `virtuoso.py` and 3001 is the intended HTTP port for the service, to start the Virtuoso service:

First, change 11st line of `virtuoso.py` to ```virtuosoPath = "../virtuoso-opensource"```.

Then,
```
python3 virtuoso.py start 3001 -d virtuoso_db
```

and to stop a currently running service at the same port:

```
python3 virtuoso.py stop 3001
```

A server with at least 100 GB RAM is recommended. You may adjust the maximum amount of RAM the service may use and other configurations via the provided script.

## Dataset

Experiments are conducted on 3 semantic parsing benchmarks WebQSP, CWQ and GrailQA.

### WebQSP

Download the WebQSP dataset from [here](https://www.microsoft.com/en-us/research/publication/the-value-of-semantic-parse-labeling-for-knowledge-base-question-answering-2/) and put them under `data/WebQSP/origin`. The dataset files should be named as `WebQSP.test[train].json`.

```
ChatKBQA/
└── data/
    ├── WebQSP                  
        ├── origin                    
            ├── WebQSP.train.json                    
            └── WebQSP.test.json                                       
```

### CWQ

Download the CWQ dataset [here](https://www.dropbox.com/sh/7pkwkrfnwqhsnpo/AACuu4v3YNkhirzBOeeaHYala) and put them under `data/CWQ/origin`. The dataset files should be named as `ComplexWebQuestions_test[train,dev].json`.

```
ChatKBQA/
└── data/
    ├── CWQ                 
        ├── origin                    
            ├── ComplexWebQuestions_train.json                   
            ├── ComplexWebQuestions_dev.json      
            └── ComplexWebQuestions_test.json                              
```

### GrailQA

This dataset contains the parallel data of natural language questions and the corresponding logical forms in **SPARQL**. It can be downloaded via the [offical website](https://dki-lab.github.io/GrailQA/) as provided by [Gu et al. (2020)](https://dl.acm.org/doi/abs/10.1145/3442381.3449992). To focus on the sole task of semantic parsing, we replace the entity IDs (e.g. `m.06mn7`) with their respective names (e.g. `Stanley Kubrick`) in the logical forms, thus eliminating the need for an explicit entity linking module. 

Then, replace the url in `data/grailqa/utils/sparql_executer.py` with your own Freebase KG virtuoso url.

Please note that such replacement can cause inequivalent execution results.  Thus, the performance reported in our paper may not be directly comparable to the other works. 

To pull the dependencies for running the GrailQA experiments, please run:

```sh
sudo bash pull_dependency_grailqa.sh
```

## Main Processing

(1) **Parse SPARQL queries to S-expressions** 

- WebQSP: Run `python parse_sparql_webqsp.py` and the augmented dataset files are saved as `data/WebQSP/sexpr/WebQSP.test[train,dev].json`. 

- CWQ: Run `python parse_sparql_cwq.py`, and it will augment the original dataset files with s-expressions. 
The augmented dataset files are saved as `data/CWQ/sexpr/CWQ.test[train,dev].json`.
 

(2) **Prepare data for training and predicttion**

- WebQSP: Run `python data_process.py --action merge_all --dataset WebQSP --split test[train]`. The merged data file will be saved as `data/WebQSP/generation/merged/WebQSP_test[train].json`.

- CWQ: Run `python data_process.py --action merge_all --dataset CWQ --split test[train,dev]` The merged data file will be saved as `data/CWQ/generation/merged/CWQ_test[train,dev].json`.


(2) **Prepare data for LLM model**

- WebQSP: Run `python process_NQ.py --dataset_type WebQSP`. The merged data file will be saved as `LLMs/data/WebQSP_Freebase_NQ_test[train]/examples.json`.

- CWQ: Run `python process_NQ.py --dataset_type CWQ` The merged data file will be saved as `LLMs/data/CWQ_Freebase_NQ_test[train,dev]/examples.json`.

(3) **Train and test LLM model for Logical Form Generation**

- WebQSP: 

Train LLMs for Logical Form Generation:
```bash
CUDA_VISIBLE_DEVICES=3 nohup python -u LLMs/LLaMA/src/train_bash.py --stage sft --model_name_or_path meta-llama/Llama-2-13b-hf --do_train  --dataset_dir LLMs/data --dataset WebQSP_Freebase_NQ_train --template default  --finetuning_type lora --lora_target q_proj,v_proj --output_dir Reading/LLaMA2-13b/WebQSP_Freebase_NQ_lora_epoch100/checkpoint --overwrite_cache --per_device_train_batch_size 4 --gradient_accumulation_steps 4  --lr_scheduler_type cosine --logging_steps 10 --save_steps 1000 --learning_rate 5e-5  --num_train_epochs 100.0 --plot_loss  --fp16 >> train_LLaMA2-13b_WebQSP_Freebase_NQ_lora_epoch100.txt 2>&1 &
```

Test LLMs for Logical Form Generation:
```bash
CUDA_VISIBLE_DEVICES=2 nohup python -u LLMs/LLaMA/src/train_bash.py --stage sft --model_name_or_path meta-llama/Llama-2-13b-hf --do_predict  --dataset_dir LLMs/data  --dataset WebQSP_Freebase_NQ_test --template default  --finetuning_type lora --checkpoint_dir Reading/LLaMA2-13b/WebQSP_Freebase_NQ_lora_epoch100/checkpoint --output_dir Reading/LLaMA2-13b/WebQSP_Freebase_NQ_lora_epoch100/evaluation --per_device_eval_batch_size 32 --predict_with_generate >> pred_LLaMA2-13b_WebQSP_Freebase_NQ_lora_epoch100.txt 2>&1 &
```

Beam-setting LLMs for Logical Form Generation:
```bash
CUDA_VISIBLE_DEVICES=5 nohup python -u LLMs/LLaMA/src/beam_output_eva.py --model_name_or_path meta-llama/Llama-2-13b-hf --dataset_dir LLMs/data --dataset WebQSP_Freebase_NQ_test --template default --finetuning_type lora --checkpoint_dir Reading/LLaMA2-13b/WebQSP_Freebase_NQ_lora_epoch100/checkpoint --num_beams 10 >> predbeam_LLaMA2-13b_WebQSP_Freebase_NQ_lora_epoch100.txt 2>&1 &
```


(4) **Evaluate KBQA result with Retrieval**

- WebQSP: 

Evaluate KBQA result with entity-retrieval and relation-retrieval:
```bash
CUDA_VISIBLE_DEVICES=0 nohup python -u eval_final.py >> predfinal_LLaMA2-13b_WebQSP_Freebase_NQ_lora_epoch100.txt 2>&1 &
```

Evaluate KBQA result with golden-entities and relation-retrieval:
```bash
CUDA_VISIBLE_DEVICES=1 nohup python -u eval_final.py --golden_ent >> predfinalgoldent_LLaMA2-13b_WebQSP_Freebase_NQ_lora_epoch100.txt 2>&1 &
```