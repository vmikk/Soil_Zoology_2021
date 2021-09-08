## Анализ с использованием USEARCH

[USEARCH](https://www.drive5.com/usearch/) (Edgar, 2010) - небольшая, но очень популярная программа с большими возможностями. Представляет из себя "комбайн", позволяющий выполнить анализ данных метабаркодинга начиная с предобработки сиквенсов и заканчивая оценкой разнообразия сообществ.

Основные этапы пайплайна:
- Сборка парно-концевых прочтений
- Удаление участков с синтетическими последовательностями (всё, что находящихся за пределами сайтов, комплементарным праймерам)
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


## Таксономическая аннотация

### BLAST

Таксономическая аннотация на основе локального выравнивания последовательностей с помощью BLAST. Используем алгоритм быстрого сравнения с целью поиска высоко сходных последовательностей (`megablast`), для каждой OTU в результат сохраняем 10 лучших соответствий из БД.

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

Количество найденнных в БД соотвтетсвий с высоким сходством (> 95%)

```bash
awk '{if ($3 > 95) print $0}' otus_tax.m8 | cut -f1 | wc -l
```

Извлечение самого лучшего соответствий для каждой OTU, 
результат запишем в файл `best_hits.txt`.

```bash
awk '{ if(!x[$1]++) {print $0; bitscore=($14-1)} else { if($14>bitscore) print $0} }' blastout.txt \
  > best_hits.txt
```

