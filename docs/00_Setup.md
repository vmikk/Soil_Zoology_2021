# Установка необходимого ПО

Для установки необходимых программ нам понадобится `Windows 10 + WSL` или любой дистрибутив `Linux`.
Для более ранних версий Windows можно использовать виртальную машину (например, [`VirtualBox`](https://download.virtualbox.org/virtualbox/6.1.26/VirtualBox-6.1.26-145957-Win.exe) и образ [`Ubuntu` 20.04](https://ubuntu.com/download/desktop/thank-you?version=20.04.3&architecture=amd64)).


## 01. Windows Subsystem for Linux (WSL)

В случае использования Windows необходимо установить WSL (Windows Subsystem for Linux) - слой совместимости для запуска Linux-приложений.

1. Включение WSL
Запустить `PowerShell` **от имени администратора**.<br/>
Ввести в консоли следующую команду:
```
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```
2. Включение компонента виртуальных машин (‘Virtual Machine Platform’)

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


При первом запуске Linux потребуется создать новую учетную запись пользователя.


Более подробную иструкцию см. [здесь](https://docs.microsoft.com/ru-ru/windows/wsl/install-win10).

