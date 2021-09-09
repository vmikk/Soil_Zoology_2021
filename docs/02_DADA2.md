## Анализ данных с использованием DADA2 и dadaist2

Формируем таблицу с соответствием образцов файлам с парно-концевыми прочтениями:

```bash
dadaist2-metadata -i data/ > metadata.tsv
cat metadata.tsv
```

Далее команда `dadaist2` запускает пайплан DADA2, в который входит:
- фильтрация данных по качеству (c помощью [`SeqFu`](https://telatin.github.io/seqfu2/); Telatin et al., 2021)
- удаление участков, комплементарных праймерам (с помощью `Cutadapt`; Martin, 2011)
- сборку парно-концевых ридов
- удаление ошибок секвенирования и формирование ASV (c помощью [`DADA2`](https://benjjneb.github.io/dada2/tutorial.html); Callahan et al., 2016)
- удаление химерных последовательностей
- таксономическую аннотацию (c помощью функции [`assignTaxonomy`](https://benjjneb.github.io/dada2/assign.html))
- множественное выравнивание последовательностей ASV и построение филогенетического дерева
- оценку разнообразия с использованием пакета [`Rhea`](https://lagkouvardos.github.io/Rhea/) (Lagkouvardos et al., 2017)
- подготвку данных для других программ - [`phyloseq`](https://joey711.github.io/phyloseq/) (McMurdie, Holmes,2013) и [`MicrobiomeAnalyst`](https://www.microbiomeanalyst.ca/MicrobiomeAnalyst/upload/OtuUploadView.xhtml) (Chong et al., 2020)

```bash
dadaist2 \
  -i data/ \
  -m metadata.tsv \
  -d db/COIv4_DB.fa.gz \
  --primers "GGWACWGGWTGAACWGTWTAYCCYCC":"TANACYTCNGGRTGNCCRAARAAYCA" \
  -t 4 \
  -l DADA2_results/DADA2_log.txt \
  -o DADA2_results
```
`-i` - директория, в которой находятся FASTQ файлы 
`-m` - указывает на файл с метаданными
`-d` - путь к базе данных для таксономической аннотиации
`--primers` указывает на последовательности используемых праймеров (Leray et al. 2013):
    - mlCOIintF - GGWACWGGWTGAACWGTWTAYCCYCC
    - jgHCO2198 - TAIACYTCIGGRTGICCRAARAAYCA
`-t` - количество ядер процессора
`-l` - файл с отчетом о выполнении программ
`-o` - папка с результатми анализа


Основные результаты:
- `dada2_stats.tsv` - отчёт о количестве ридов после фильтрации
- `rep-seqs-tax.fasta` - последовательности ASV c таксономической аннотацией в заголовках
- `feature-table.tsv` - количество ридов ASV в образцах
- `rep-seqs.tree` - филогенетическое дерево ASV в формате Newick
- `rep-seqs.msa` - выравнивание последовательностей ASV


## MultiQC-отчёт

В интерактивном виде также можно посмотреть отчёт о 
количестве ридов после фильтрации, обесшумливания, 
сборки парно-концевых ридов и удаления химер, а также 
о соотношении обилий доминирующих последовательностей с 
учётом их таксономического положения.

```bash
## Подготовка данных для отчёта
dadaist2-mqc-report \
  -i DADA2_results/ \
  -t 10 \
  -o DADA2_results/mqc

## Формирование MultiQC-отчёта
multiqc -c DADA2_results/mqc/config.yaml DADA2_results/qc/

## Открываем отчет в браузере (например, в Firefox)
firefox multiqc_report.html 
```

_________________

Примеры анализа данных с использованием:
- [USEARCH](01_USEARCH.md)
- [PipeCraft2](03_PipeCraft2.md)
