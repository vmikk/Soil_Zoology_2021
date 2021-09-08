# Установка необходимого ПО

Для установки необходимых программ нам понадобится `Windows 10 + WSL` или любой дистрибутив `Linux`.
Для более ранних версий Windows можно использовать виртальную машину (например, [`VirtualBox`](https://download.virtualbox.org/virtualbox/6.1.26/VirtualBox-6.1.26-145957-Win.exe) и образ [`Ubuntu` 20.04](https://ubuntu.com/download/desktop/thank-you?version=20.04.3&architecture=amd64)).

Системные требования - около 2GB свободного места на диске, >10GB оперативной памяти.


## 01. Windows Subsystem for Linux (WSL)

В случае использования Windows необходимо установить WSL (Windows Subsystem for Linux) - слой совместимости для запуска Linux-приложений.

1. Включение WSL
    Запустить `PowerShell` **от имени администратора**.<br/>
    Ввести в консоли следующую команду:
    ```
    dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
    ```
2. Включение компонента виртуальных машин (‘Virtual Machine Platform’)<br/>
    Для Windows 10 (2004):
    ```
    dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
    ```
    Для Windows 10 (1903, 1909):
    ```
    Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart
    ```

3. Перезупустить ПК

4. Скачать пакет обновления ядра Linux<br/>
    [Пакет обновления ядра Linux в WSL 2 для 64-разрядных компьютеров](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)


5. Выбор WSL 2 в качестве версии по умолчанию
    ```
    wsl --set-default-version 2
    ```

6. Установка дистрибутива Linux
    - Открыть `Microsoft Store` из меню `Пуск`;
    - Выбрать и установить дистрибутив Linux (например, `Ubuntu 20.04`);
    - Также установить терминал Windows (`Windows Terminal app`) из `Microsoft Store`.

При возникновении ошибок `0x80070003` или `0x80370102`, 
необходимо включить виртуализацию CPU в BIOS (на этапе загрузки компьютера). 
В настройках данный пункт обычно находится на вкладке "Advanced". 
Для процессоров Intel он может называться "Intel Virtualization Technology" или "Intel VT-x", 
для процессоров AMD - "AMD Secure Virtual Machine" или "AMD SVM".


Более подробную иструкцию по установке `WSL` см. [здесь](https://docs.microsoft.com/ru-ru/windows/wsl/install-win10).


Для того чтобы попасть в командную строку Linux, необходимо открыть `Windows Terminal` <img src="Images/windows_terminal_icon.png" width="50" title="Windows Terminal"> и ввести:<br/>
```
wsl
```

При первом запуске Linux потребуется создать новую учетную запись пользователя. 
Обратите внимание, что при вводе пароля сам пароль не показыватеся.


Чтобы получить доступ к файлам "внутри WSL" через Проводник Windows можно ввести:
```
explorer.exe .
```
(точка в конце строки важна! - она означает "текущий каталог").


## 02. Установка ПО для биоинфорационного анализа

Дальнейшие команды необходимо запускать **в командной строке Linux**.<br/>

При отстутстви прогрммы `wget`, необходимо её установить
```bash
sudo apt -y install wget
```

1. Установка менеджера пакетов [`conda`](https://conda.io/miniconda.html)

    ```bash
    cd ~
    wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda.sh
    bash ~/miniconda.sh -b -p $HOME/miniconda
    ~/miniconda/bin/conda init bash
    source ~/.bashrc
    conda update --all --yes -c bioconda -c conda-forge
    conda install --yes -c conda-forge mamba unzip
    rm ~/miniconda.sh
    ```


2. Установка [`DADA2`](https://benjjneb.github.io/dada2/index.html) (Callahan et al., 2016) и [`dadaist2`](https://quadram-institute-bioscience.github.io/dadaist2/) (Ansorge et al., 2021)

    ```bash
    mamba install --yes -c conda-forge -c bioconda -c r dadaist2-full
    ```


3. Установка `USEARCH` (Edgar, 2010)

    ```bash
    mkdir -p ~/bin
    wget https://www.drive5.com/downloads/usearch11.0.667_i86linux32.gz
    gunzip usearch11.0.667_i86linux32.gz
    mv usearch11.0.667_i86linux32 ~/bin/usearch11
    chmod +x ~/bin/usearch11
    ```

4. Установка [`BLAST+`](https://www.ncbi.nlm.nih.gov/books/NBK279690/) (Camacho et al., 2009)

    ```bash
    mamba install --yes -c bioconda blast
    ```

## 03. Загрузка демонстрационных файлов и баз данных

В качестве демонстрационного набора данных будут использованы последовательности 
ДНК участка гена субъединицы I цитохром-с-оксидазы (длиной порядка 305-315 нуклеотидов). 
Для примера рассматриваются сокращённые версии 10 образцов почвы (Anslan et al., 2021; SRA BioProject ID PRJNA743174),
префикс в названии файла обозначает место происхождения образца (`Forest_*` - лес, `Field_*` - поле).
Каждый из образцов представляет собой смешанную пробу (9 почвенных кернов грубиной 10 см и диаметром 5 см), 
отобраную в окрестностях г. Тарту (Эстония) на пробных площадях 30 x 30 м (сетка 3 х 3). 
Секвенирование ДНК выполнено с использованием платформы DNBSEQ-G400RS (MGI-Tech). 


Для таксономической идентификации будет использована "облегченная версия" базы данных последовательностей COI,
составленной Терезитой Портер (Porter, 2021), в котрую включены последовательности, 
потенциально амплифицируемые праймерами `mlCOIintF` и `jgHCO2198` (Leray et al. 2013), 
на основе которых были получены ампликоны исследуемых образцов.


- Загрузка и распаковка демонстрационных файлов и баз данных:

    ```bash
    cd ~
    wget https://github.com/vmikk/Soil_Zoology_2021/releases/download/v1/data.zip
    wget https://github.com/vmikk/Soil_Zoology_2021/releases/download/v1/db.zip
    
    unzip data.zip
    unzip db.zip
    gunzip db/COIv4_DB_SINTAX.fa.gz
    ```

- Создание базы для BLAST-поиска:

    ```bash
    gunzip -k db/COIv4_DB.fa.gz
    makeblastdb -in db/COIv4_DB.fa -dbtype nucl -out db/COIv4_BLAST
    rm db/COIv4_DB.fa
    ```

- Удаление временных файлов:

    ```bash
    rm data.zip
    rm db.zip
    ```


## 04. Ссылки

- Anslan S, Mikryukov V, Armolaitis K, Ankuda J, Lazdina D, Makovskis K, Vesterdal L, Schmidt IK, Tedersoo L. Highly comparable metabarcoding results from MGI-Tech and Illumina sequencing platforms // _PeerJ_ (2021, _in press_).

- Ansorge R, Birolo G, James SA, Telatin A. Dadaist2: A Toolkit to automate and simplify statistical analysis and plotting of metabarcoding experiments // _International Journal of Molecular Sciences_ 22 (2021). [DOI:10.3390/ijms22105309](https://www.mdpi.com/1422-0067/22/10/5309)

- Callahan B, McMurdie P, Rosen M et al. DADA2: High-resolution sample inference from Illumina amplicon data // _Nature Methods_ 13 (2016). [DOI:10.1038/nmeth.3869](https://www.nature.com/articles/nmeth.3869)

- Camacho C, Coulouris G, Avagyan V. et al. BLAST+: Architecture and applications // _BMC Bioinformatics_ 10 (2009). [DOI:10.1186/1471-2105-10-421](https://bmcbioinformatics.biomedcentral.com/articles/10.1186/1471-2105-10-421)

- Edgar RC. Search and clustering orders of magnitude faster than BLAST // _Bioinformatics_ 26-19 (2010). [DOI:10.1093/bioinformatics/btq461](https://academic.oup.com/bioinformatics/article/26/19/2460/230188)

- Leray M, Yang JY, Meyer CP et al. A new versatile primer set targeting a short fragment of the mitochondrial COI region for metabarcoding metazoan diversity: application for characterizing coral reef fish gut contents // _Frontiers in Zoology_ 10-34 (2013). [DOI:10.1186/1742-9994-10-34](https://frontiersinzoology.biomedcentral.com/articles/10.1186/1742-9994-10-34)

- Porter TM. Eukaryote CO1 reference set for the RDP classifier (Version v4.0.1) // _Zenodo_. DOI:10.5281/zenodo.4741447 URL:https://github.com/terrimporter/CO1Classifier


_________________

Примеры анализа данных с использованием:
- [USEARCH](01_USEARCH.md)
- [DADA2](02_DADA2.md)
- [PipeCraft2](03_PipeCraft2.md)
