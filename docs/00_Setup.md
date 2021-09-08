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


Более подробную иструкцию по установке `WSL` см. [здесь](https://docs.microsoft.com/ru-ru/windows/wsl/install-win10).


Для того чтобы попасть в командную строку Linux, необходимо открыть `Windows Terminal` <img src="Images/windows_terminal_icon.png" width="50" title="Windows Terminal"> и ввести:<br/>
```
wsl
```

При первом запуске Linux потребуется создать новую учетную запись пользователя.


Чтобы получить доступ к файлам "внутри WSL" через Проводник Windows можно ввести:
```
explorer.exe .
```
(точка в конце строки важна! - она означает "текущий каталог").


## 02. Установка ПО для биоинфорационного анализа

Дальнейшие команды необходимо запускать **в коммандной строке Linux**.<br/>

1. Установка менеджера пакетов [`conda`](https://conda.io/miniconda.html)<br/>

В коммандной строке ввести

    ```bash
    wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda.sh
    bash ~/miniconda.sh -b -p $HOME/miniconda
    ~/miniconda/bin/conda init bash
    source ~/.bashrc
    conda update --all --yes -c bioconda -c conda-forge
    conda install --yes -c conda-forge mamba
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

- Загрузка и распаковка демонстрационных файлов и баз данных:
    ```bash
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


_________________

Примеры анализа данных с использованием:
- [USEARCH](01_USEARCH.md)
- [DADA2](02_DADA2.md)
- [PipeCraft2](03_PipeCraft2.md)
