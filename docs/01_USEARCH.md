## Анализ с использованием USEARCH

Перед началом работ проверить установлена ли программа `USEARCH`
```bash
which usearch11
```
(если ответа нет, то см. [инструкцию по установке](00_Setup.md))


Задать переменную с количеством ядер процессора, которые будут использоваться для анализа
```bash
NCORES=4
```


## Формирование zOTU и OTU

OTU = операциональная таксономическая единица (operational taxonomic unit) - группа сходных последовательностей ДНК. Кластеризацию проводят на основе определенного порога сходства (например, 97%) условно соответствующего какому-либо таксономическому уровню.

zOTU (zero-radius OTUs) - уникальные последовательности ампликонов, прошедшие этап удаления ошибочных последовательностей. Также могут называться ASV (amplicon sequence variant) или ESV (exact sequence variant). Могут соотвтетсвовать уровню штамма или гаплотипа (см. Callahan et al., 2017).


```bash
cd ~
mkdir -p USEARCH
cd USEARCH

## Распакова архивов с прочтениями
mkdir -p Uncompressed
for i in ../data/*.gz; do 
  fn=$(basename $i .gz)
  gunzip -c "$i" > Uncompressed/"$fn"
done

## Переименование обрзцов
rename 's/Forest./Forest_/' Uncompressed/*.fq
rename 's/Field./Field_/' Uncompressed/*.fq

# Сборка парно-концевых прочтений,
# добавление названий образцов в заголок последовательностей,
# объединение всех прочтений в один файл
usearch11 \
  -fastq_mergepairs Uncompressed/*R1.fq \
  -fastqout merged.fq \
  -relabel @

# Удаление участков, комплементарных праймерам
cutadapt \
  -g "GGWACWGGWTGAACWGTWTAYCCYCC...TGRTTYTTYGGNCAYCCNGARGTNTA" \
  --revcomp \
  --errors 2 --overlap 20 \
  --minimum-length 200 \
  --discard-untrimmed  \
  --cores "$NCORES" \
  -o trimmed_reads.fq \
  merged.fq

# Фильтрация прочтений по качеству
usearch11 \
  -fastq_filter trimmed_reads.fq \
  -fastq_maxee 1.0 \
  -fastq_maxns 1 \
  -fastaout filtered.fa

# Объединение уникальных прочтений (дерепликация)
usearch11 \
  -fastx_uniques filtered.fa \
  -sizeout \
  -relabel Uniq \
  -fastaout uniques.fa

# Удаление ошибочных последовательностей
# (на основе алгоритма UNOISE, Edgar)
# Удаление химерных последовательностей
# Формирование zOTU
usearch11 \
  -unoise3 uniques.fa \
  -minsize 8 \
  -zotus zotus.fa

# Кластеризация zOTU на основе порога сходства последовательностей (97%)
# Формирование OTU
usearch11 \
  -cluster_smallmem zotus.fa \
  -id 0.97 -strand both \
  -sortedby other \
  -centroids otus.fa \
  -relabel OTU
```


Формирование таблицы с количетсвом прочтений каждой из OTU (или zOTU) в образцах:

```bash
# Таблица OTU
usearch11 -otutab merged.fq -otus otus.fa -otutabout otutab_raw.txt

# Таблица zOTU
# usearch11 -otutab merged.fq -otus zotus.fa -otutabout zotutab_raw.txt
```

Проверим количество прочтений в образцах:

```bash
usearch11 -alpha_div otutab_raw.txt -output seq_depth.txt -metrics reads
cat seq_depth.txt
```

Поскольку количество прочтений может значительно отличаться между образцами, 
перед анализом можно провести рарефикацию:

```bash
usearch11 -otutab_rare otutab_raw.txt -sample_size 2500 -output otutab.txt
# usearch11 -otutab_rare zotutab_raw.txt -sample_size 2500 -output zotutab.txt
```

Таксономическая аннотация последовательностей ДНК, а также 
анализ альфа- и бета-разнообразия может быть проведен 
как на уровне zOTU (использовать файлы `zotus.fa` и `zotutab.txt`), 
так и уровне OTU (использовать файлы `otus.fa` и `otutab.txt`). 
Для простоты мы остановимся на втором случае.

