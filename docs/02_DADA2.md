## Анализ данных с использованием DADA2 и dadaist2

`DADA2` (Callahan et al., 2016) ...
Программа `dadaist2` (Ansorge et al., 2021) - позволяет легко автоматизировать анализ ....

ASV

Формируем таблицу с соответствием образцов файлам с парно-концевыми прочтениями:

```bash
dadaist2-metadata -i data/ > metadata.tsv
cat metadata.tsv
```

Далее команда `dadaist2` запускает пайплан DADA2, в который входит:
- фильтрация данных по качеству (c помощью [`SeqFu`](https://telatin.github.io/seqfu2/); Telatin et al., 2021)
- удаление участков, комплементарных праймерам (с помощью [`Cutadapt`](https://cutadapt.readthedocs.io); Martin, 2011)
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
`-i` - директория, в которой находятся FASTQ файлы<br/>
`-m` - указывает на файл с метаданными<br/>
`-d` - путь к базе данных для таксономической аннотиации<br/>
`--primers` указывает на последовательности используемых праймеров (Leray et al. 2013):<br/>
    - mlCOIintF - GGWACWGGWTGAACWGTWTAYCCYCC<br/>
    - jgHCO2198 - TAIACYTCIGGRTGICCRAARAAYCA<br/>
`-t` - количество ядер процессора<br/>
`-l` - файл с отчетом о выполнении программ<br/>
`-o` - папка с результатми анализа<br/>


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


## Rhea

В подпапке `Rhea` находятся результаты

- анализа разнообразия образцов
    ```bash
    cat DADA2_results/Rhea/
    ```
- таблицы с нормализированным абсолютным (`OTUs_Table-norm-tax.tab`) и относительным (`OTUs_Table-norm-rel-tax.tab`) обилием ASV в образцах
- кривые насыщения ASV в зависимости от количества прочтений (`RarefactionCurve.pdf`)


## Ссылки

- Callahan B, McMurdie P, Rosen M et al. **DADA2**: High-resolution sample inference from Illumina amplicon data // _Nature Methods_ 13 (2016). [DOI:10.1038/nmeth.3869](https://www.nature.com/articles/nmeth.3869)

- Ansorge R, Birolo G, James SA, Telatin A. **Dadaist2**: A Toolkit to automate and simplify statistical analysis and plotting of metabarcoding experiments // _International Journal of Molecular Sciences_ 22 (2021). [DOI:10.3390/ijms22105309](https://www.mdpi.com/1422-0067/22/10/5309)

- Ewels P, Magnusson M, Lundin S, Käller M. **MultiQC**: Summarize analysis results for multiple tools and samples in a single report // _Bioinformatics_ (2016) [DOI:10.1093/bioinformatics/btw354](https://academic.oup.com/bioinformatics/article/32/19/3047/2196507)

- Lagkouvardos I, Fischer S, Kumar N, Clavel T. **Rhea**: a transparent and modular R pipeline for microbial profiling based on 16S rRNA gene amplicons // _PeerJ_ 5 (2017) [DOI:10.7717/peerj.2836](https://peerj.com/articles/2836/)

- Chong J, Liu P, Zhou G et al. Using **MicrobiomeAnalyst** for comprehensive statistical, functional, and meta-analysis of microbiome data // _Nature Protocols_ 15 (2020). [DOI:10.1038/s41596-019-0264-1](https://www.nature.com/articles/s41596-019-0264-1)

- Leray M, Yang JY, Meyer CP et al. A new versatile primer set targeting a short fragment of the mitochondrial COI region for metabarcoding metazoan diversity: application for characterizing coral reef fish gut contents // _Frontiers in Zoology_ 10-34 (2013). [DOI:10.1186/1742-9994-10-34](https://frontiersinzoology.biomedcentral.com/articles/10.1186/1742-9994-10-34)

- Martin M. **Cutadapt** removes adapter sequences from high-throughput sequencing
reads // _EMBnet Journal_ 17 (2011). [DOI:10.14806/ej.17.1.200](https://journal.embnet.org/index.php/embnetjournal/article/view/200)

- McMurdie PJ, Holmes S. **phyloseq**: An R Package for Reproducible Interactive Analysis and Graphics of Microbiome Census Data // _PLOS ONE_ 8-4 (2013). [DOI:10.1371/journal.pone.0061217](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0061217)

- Telatin A, Fariselli P, Birolo G. **SeqFu**: A Suite of Utilities for the Robust and Reproducible Manipulation of Sequence Files // _Bioengineering_ 8 (2021). [DOI:10.3390/bioengineering8050059](https://www.mdpi.com/2306-5354/8/5/59)

_________________

Примеры анализа данных с использованием:
- [USEARCH](01_USEARCH.md)
- [PipeCraft2](03_PipeCraft2.md)
