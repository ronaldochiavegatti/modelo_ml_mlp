# SVD Voice Pathology Classification Pipeline (C)

This repository implements a full, modular CLI pipeline in C for classifying vocal pathologies using the Saarbrucken Voice Database (SVD). The baseline model is an MLP (FANN), with evaluation metrics and a confusion matrix visualization.

## Scope and classes

See `docs/scope.md` for the class selection and inclusion rules. The four classes are:
- laringite
- disfonia_funcional
- disfonia_psicogenica
- edema_de_reinke

## Directory layout

```
.
├── data/                     # Organized dataset: data/<classe>/<id>.wav
├── processed/                # Preprocessed frames: processed/<classe>/<id>.frames
├── features.csv              # Aggregated features per file
├── train.data                # FANN training data
├── test.data                 # FANN test data
├── classes.txt               # Class order used for one-hot encoding
├── model.net                 # Trained FANN model
├── results.csv               # Metrics + confusion matrix
├── confusion.svg             # Confusion matrix plot
├── model_silence_best.svg    # Best model architecture diagram
├── docs/
│   └── scope.md
└── src/
    └── *.c
```

## Dependencies

- C compiler (gcc/clang)
- libsndfile (audio I/O)
- FANN (MLP training)
- libsvm (optional SVM baseline)

Ubuntu/Debian example:
```
sudo apt install libsndfile1-dev libfann-dev libsvm-dev
```

## Build

```
make all
```

Binaries are built into `bin/`.

## Main flows (quick testing)

### 1) Pipeline completa (default)

```
make all
./bin/organize_dataset --metadata metadata.csv --output data --log logs/verify.log
./bin/preprocess --input data --output processed --frame-ms 30 --hop-ms 10
./bin/extract_features --input processed --metadata metadata.csv --output features.csv --n-mfcc 13 --n-mels 26 --rolloff 0.85
./bin/split_normalize --features features.csv --metadata metadata.csv --train train.data --test test.data --classes classes.txt --train-ratio 0.7 --seed 42 --scaler scaler.csv
./bin/train --train train.data --model model.net --hidden 32 --hidden2 0 --learning-rate 0.01 --max-epochs 500 --desired-error 0.001 --log train.log
./bin/evaluate --model model.net --test test.data --classes classes.txt --output results.csv
./bin/plot_confusion --input results.csv --output confusion.svg --title "SVD Confusion Matrix"
```

### 2) Pipeline com remocao de silencio

```
make all
./bin/organize_dataset --metadata metadata.csv --output data --log logs/verify.log
./bin/preprocess --input data --output processed_silence --frame-ms 30 --hop-ms 10 --remove-silence --silence-threshold 0.1
./bin/extract_features --input processed_silence --metadata metadata.csv --output features_silence.csv --n-mfcc 13 --n-mels 26 --rolloff 0.85
./bin/split_normalize --features features_silence.csv --metadata metadata.csv --train train_silence.data --test test_silence.data --classes classes_silence.txt --train-ratio 0.7 --seed 42 --scaler scaler_silence.csv
./bin/train --train train_silence.data --model model_silence.net --hidden 32 --hidden2 0 --learning-rate 0.01 --max-epochs 500 --desired-error 0.001 --log train_silence.log
./bin/evaluate --model model_silence.net --test test_silence.data --classes classes_silence.txt --output results_silence.csv
./bin/plot_confusion --input results_silence.csv --output confusion_silence.svg --title "SVD Confusion Matrix (Silence)"
```

### 3) Melhor modelo (linha observada em CV)

```
./bin/train --train train_silence.data --model model_silence_best.net --hidden 256 --hidden2 0 --learning-rate 0.001 --max-epochs 600 --desired-error 0.001 --log train_silence_best.log
./bin/evaluate --model model_silence_best.net --test test_silence.data --classes classes_silence.txt --output results_silence_best.csv
```

A acuracia e impressa pelo `./bin/evaluate`.
O `./bin/train` tenta gerar automaticamente `<modelo>.svg` usando `src/plot_network_svg.py`.

### Visualizar os pesos treinados

O SVG `model_silence_best.svg` mostra o layout das camadas, mas os valores reais estão em `model_silence_best.net`. Use `scripts/dump_fann_weights.py` para gerar uma planilha CSV por ligação entre camadas:

```
python scripts/dump_fann_weights.py --model model_silence_best.net
```

Os arquivos resultantes vão para `weights/model_silence_best/`. Cada linha lista um neurônio de destino e cada coluna corresponde a um neurônio da camada anterior (a coluna `src_global_<n>` aponta para o índice global do `n`-ésimo neurônio anterior), o que permite cruzar os valores numéricos com o diagrama SVG.

### 4) Cross-validation + tuning (k=5)

```
./bin/cross_validate --features features.csv --metadata metadata.csv --k 5 \
  --hidden 32,64,128 --hidden2 0,32 --learning-rate 0.01,0.001 --max-epochs 300 --seed 42 \
  --output cv_report.csv
```

### 5) Baseline SVM

```
./bin/svm_baseline --train train.data --test test.data --classes classes.txt --output svm_results.csv --c 1.0 --gamma 0.0 --model svm_model.svm
```

### Reset do ambiente (limpeza total)

```
make clean
rm -rf data processed processed_silence logs
rm -f features.csv features_silence.csv train.data test.data classes.txt classes_silence.txt \
  model.net model_silence.net model_silence_best.net results.csv results_silence.csv \
  confusion.svg confusion_silence.svg train.log train_silence.log train_silence_best.log \
  scaler.csv scaler_silence.csv svm_model.svm svm_results.csv cv_report*.csv
```

## Step-by-step usage

### 1) Define metadata (no code)

Create `metadata.csv` using the schema described in `docs/scope.md`.
Use `metadata.example.csv` as a template.

### 2) Organize dataset

```
./bin/organize_dataset --metadata metadata.csv --output data --log logs/verify.log
```

Options:
- `--mode link` to create symlinks instead of copying
- `--overwrite` to overwrite existing files
- `--dry-run` to only log operations

### 3) Preprocess audio

```
./bin/preprocess --input data --output processed --frame-ms 30 --hop-ms 10
```

Optional silence removal:
```
./bin/preprocess --input data --output processed --frame-ms 30 --hop-ms 10 --remove-silence --silence-threshold 0.1
```

Output: `processed/<classe>/<id>.frames`

### 4) Extract features

```
./bin/extract_features --input processed --metadata metadata.csv --output features.csv --n-mfcc 13 --n-mels 26 --rolloff 0.85
```

Features per frame:
- MFCC (13)
- RMS energy
- ZCR
- Spectral centroid
- Spectral rolloff

Aggregated per file as mean and std.

### 5) Split + normalize (z-score)

```
./bin/split_normalize --features features.csv --metadata metadata.csv --train train.data --test test.data --classes classes.txt --train-ratio 0.7 --seed 42
```

This split is stratified by class and avoids speaker leakage.
If `speaker_id` is not present in metadata, the prefix before the first '_' in `id` is used.

### 6) Train MLP baseline (FANN)

```
./bin/train --train train.data --model model.net --hidden 32 --hidden2 0 --learning-rate 0.01 --max-epochs 500 --desired-error 0.001 --log train.log
```

### 7) Cross-validation + tuning (k=5)

```
./bin/cross_validate --features features.csv --metadata metadata.csv --k 5 \
  --hidden 32,64,128 --hidden2 0,32 --learning-rate 0.01,0.001 --max-epochs 300 --seed 42 \
  --output cv_report.csv
```

### 8) Alternative model (SVM, libsvm)

```
./bin/svm_baseline --train train.data --test test.data --classes classes.txt --output svm_results.csv --c 1.0 --gamma 0.0 --model svm_model.svm
```

### 9) Final evaluation

```
./bin/evaluate --model model.net --test test.data --classes classes.txt --output results.csv
```

### 10) Confusion matrix plot

```
./bin/plot_confusion --input results.csv --output confusion.svg --title "SVD Confusion Matrix"
```

## Notes

- All steps are independent CLI executables; outputs from one step are inputs to the next.
- For consistent MFCCs, keep a fixed sample rate across the dataset (resample during conversion if needed).
- The baseline target accuracy is ~87% following common SVD literature practices.

## Future work

Trabalhos futuros: uso de Transformer como extrator de embeddings.
# modelo_ml_mlp
