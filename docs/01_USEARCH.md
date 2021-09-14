## Анализ с использованием USEARCH

[USEARCH](https://www.drive5.com/usearch/) (Edgar, 2010) - небольшая, но очень популярная программа с большими возможностями. Представляет из себя "комбайн", позволяющий выполнить анализ данных метабаркодинга начиная с предобработки сиквенсов и заканчивая оценкой разнообразия сообществ.

Основные этапы пайплайна:
- Сборка парно-концевых прочтений
- Удаление участков с синтетическими последовательностями (всё, что находится за пределами сайтов, комплементарным праймерам)
- Фильтрация прочтений по качеству: максимальное ожидаемое количество ошибок (см.  Edgar and Flyvbjerg, 2015), количество неопределённых нуклеотидов, длина последовательности
- Дерепликация - поиск и подсчёт количества уникальных прочтений
- Удаление ошибочных и химерных последовательностей
- Кластеризация последовательностей на основе их сходства (формирование операциональных таксономических единиц, OTU)
- Подсчет количества последовательностей OTU в образцах
- Определение таксономической принадлежности OTU
- Оценка альфа- и бета-разнообразия


Перед началом работ проверить установлена ли программа `USEARCH`
```bash
which usearch11
```
(если ответа нет, то см. [инструкцию по установке](00_Setup.md))


## Формирование zOTU и OTU

OTU = операциональная таксономическая единица (operational taxonomic unit) - группа сходных последовательностей ДНК. Кластеризацию проводят на основе определенного порога сходства (например, 97%) условно соответствующего какому-либо таксономическому уровню (например, виду).

zOTU (zero-radius OTUs) - уникальные последовательности ампликонов (100% порог сходства), прошедшие этап удаления ошибочных последовательностей. Также могут называться ASV (amplicon sequence variant) или ESV (exact sequence variant). Могут соотвтетсвовать уровню штамма или гаплотипа (см. Callahan et al., 2017).


```bash
cd ~
mkdir -p USEARCH
cd USEARCH

# Переменная с количеством ядер процессора, 
# которые будут использоваться для анализа
NCORES=4

# Распакова архивов с прочтениями
mkdir -p Uncompressed
for i in ../data/*.gz; do 
  fn=$(basename $i .gz)
  gunzip -c "$i" > Uncompressed/"$fn"
done

# Сборка парно-концевых прочтений,
# добавление названий образцов в заголок последовательностей,
# объединение всех прочтений в один файл
usearch11 \
  -fastq_mergepairs Uncompressed/*R1.fq \
  -fastqout merged.fq \
  -relabel @

## Переименование образцов
# ("Field.G5300" -> "Field_G5300")
sed -i 's/Field./Field_/' merged.fq
sed -i 's/Forest./Forest_/' merged.fq


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
usearch11 -otutab merged.fq -otus otus.fa -sample_delim "." -otutabout otutab_raw.txt

# Таблица zOTU
# usearch11 -otutab merged.fq -otus zotus.fa -sample_delim "." -otutabout zotutab_raw.txt
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
так и на уровне OTU (использовать файлы `otus.fa` и `otutab.txt`). 
Для простоты мы остановимся на втором случае.


## Таксономическая аннотация

### BLAST

Таксономическая аннотация на основе локального выравнивания последовательностей с помощью BLAST. Используем алгоритм быстрого сравнения (`megablast`) с целью поиска высоко сходных последовательностей, для каждой OTU в результат сохраняем 10 лучших соответствий из БД.

```bash
blastn \
  -task megablast \
  -query otus.fa \
  -db ../db/COIv4_BLAST \
  -outfmt 6 \
  -strand both \
  -max_target_seqs 10 \
  -num_threads "$NCORES" \
  -out otus_tax.m8
```

Предпросмотр результатов

```bash
less -S otus_tax.m8
```
(чтобы покинуть просмотр нажмите `q`)

Проверим есть ли в результатах интересующий вас таксон (например, род `Entomobrya`):
```bash
grep "Entomobrya" otus_tax.m8
```

Сколько OTU потенциально относится к данному таксону?
```bash
grep "Entomobrya" otus_tax.m8 | awk '{ print $1 }' | sort | uniq | wc -l
```

Количество найденнных в БД соотвтетсвий с высоким сходством (> 95%)

```bash
awk '{if ($3 > 95) print $0}' otus_tax.m8 | cut -f1 | wc -l
```

Извлечение самого лучшего соответствий для каждой OTU, 
результат запишем в файл `best_hits.txt`.

```bash
awk '{ if(!x[$1]++) {print $0; bitscore=($14-1)} else { if($14>bitscore) print $0} }' otus_tax.m8 \
  > best_hits.txt
```


### SINTAX

`SINTAX` - таксономический классификатор на основе состава k-меров последовательностей (Edgar, 2016). В отличии от `BLAST`, для `SINTAX` имеет значение формат заголовков последоватеьностей в базе данных:

```
>SeqID1234;tax=d:Eukaryota,p:Arthropoda,c:Entognatha,o:Entomobryomorpha,f:Entomobryidae,g:Orchesella,s:Orchesella_cincta
GGGTGGACGGTTTATCCACCATTGGCAGCGGGTATTGCTCA...
```

Классификация на основе `SINTAX`:

```bash
usearch11 \
  -sintax otus.fa \
  -db ../db/COIv4_DB_SINTAX.fa \
  -strand both \
  -tabbedout sintax.txt \
  -sintax_cutoff 0.8
```
Параметр `sintax_cutoff` указывает на степень уверенности в аннотации (например, все что ниже значения 0.8 будет рассматриваться как неопределенное).


Сводный таксономический отчёт на уровне рода и типа

```bash
usearch11 -sintax_summary sintax.txt -otutabin otutab.txt -rank g -output summary_genus.txt
usearch11 -sintax_summary sintax.txt -otutabin otutab.txt -rank p -output summary_phylum.txt

# Просмотр отчёта
column -t summary_phylum.txt
```

Извлечение итоговой классификации

```bash
awk '{print $1 "\t" $4}' sintax.txt > sintax_confident.txt
```



## Анализ альфа- и бета-разнообразия

Оценка количества OTU (`richness`), индекса Шеннона (`shannon_e`), 
эффектвного количества OTU (`jost1`, число Хилла при `q` = 1), 
индекса выравненности Пиелу (`equitability`), 
а также количества прочтений (`reads`) в каждом образце.

```bash
usearch11 \
  -alpha_div otutab.txt \
  -output alpha.txt \
  -metrics richness,shannon_e,jost1,equitability,reads

# Просмотр таблицы с оценками разнообразия
column -t alpha.txt
```

Оценка сходства последовательностей OTU,
а также построение чернового филогенетического древа.

```bash
usearch11 -calc_distmx otus.fa -tabbedout dist.tree
usearch11 -cluster_aggd dist.tree -treeout otus.tree
```

Визуализация филогенетического древа

```bash
figtree otus.tree
```

Оценка бета-разнообразия - несходства образцов на основе
индекса Брея-Кертиса (с учётом обилия OTU), 
а также индекса UniFrac (без учёта обилия OTU, но с учётом филогенетических 
дистанций между ними; см. Lozupone et al., 2007).

```bash
usearch11 \
  -beta_div otutab.txt \
  -metrics bray_curtis,unifrac_binary \
  -tree otus.tree \
  -filename_prefix beta_ 
```

Просмотр сходства образцов на основе индекса UniFrac

```bash
figtree beta_unifrac_binary.tree
```

## Результаты

Результаты пайплайна доступны [здесь](https://github.com/vmikk/Soil_Zoology_2021/releases/download/v2/USEARCH.zip)

## Ссылки

- Callahan B, McMurdie P, Holmes S. Exact sequence variants should replace operational taxonomic units in marker-gene data analysis // _The ISME Journal_ 11 (2017). [DOI:10.1038/ismej.2017.119](https://www.nature.com/articles/ismej2017119)

- Camacho C, Coulouris G, Avagyan V. et al. **BLAST+**: Architecture and applications // _BMC Bioinformatics_ 10 (2009). [DOI:10.1186/1471-2105-10-421](https://bmcbioinformatics.biomedcentral.com/articles/10.1186/1471-2105-10-421)

- Edgar RC. **SINTAX**: a simple non-Bayesian taxonomy classifier for 16S and ITS sequences // _bioRxiv_ 074161 (2016). [DOI:10.1101/074161](https://www.biorxiv.org/content/10.1101/074161v1)

- Edgar RC. **UNOISE2**: improved error-correction for Illumina 16S and ITS amplicon sequencing // _bioRxiv_ 081257 (2016). [DOI:10.1101/081257](https://www.biorxiv.org/content/10.1101/081257v1)

- Edgar RC, Flyvbjerg H. Error filtering, pair assembly and error correction for next-generation sequencing reads // _Bioinformatics_ 31-21 (2015). [DOI:10.1093/bioinformatics/btv401](https://academic.oup.com/bioinformatics/article/31/21/3476/194979)

- Lozupone CA, Hamady M, Kelley ST, Knight R. Quantitative and qualitative beta diversity measures lead to different insights into factors that structure microbial communities // _Applied and environmental microbiology_ 73-5 (2007). [DOI:10.1128/AEM.01996-06](https://journals.asm.org/doi/10.1128/AEM.01996-06)

- Martin M. **Cutadapt** removes adapter sequences from high-throughput sequencing
reads // _EMBnet Journal_ 17 (2011). [DOI:10.14806/ej.17.1.200](https://journal.embnet.org/index.php/embnetjournal/article/view/200)

_________________

Примеры анализа данных с использованием:
- [DADA2](02_DADA2.md)
- [PipeCraft2](03_PipeCraft2.md)

