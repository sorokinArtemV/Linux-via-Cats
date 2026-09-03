---
title: Linux на маке
tags:
  - linux
  - macos
  - multipass
  - terminal
  - cheatsheet
status: living
---

# Linux на маке

> [!abstract] Читшит · живой документ
> Как поднять и не сломать Linux-ВМ на маке через Multipass, и на что натыкаешься в терминале, когда работаешь с ней по ssh.

---

## Часть 1. Multipass

### Зачем именно ВМ, а не Docker

| Критерий | Docker-контейнер | ВМ (Multipass) |
|---|---|---|
| Что это | изолированный процесс | целая ОС со своим ядром |
| systemd, демоны | обычно нет | есть |
| Для чего | гонять сервисы, паковать приложения | учить администрирование |

Контейнер не даст потрогать systemd-юниты, загрузку, права, сеть, firewall. На маке Docker вообще крутится внутри скрытой Linux-ВМ.

Рабочая схема: **Multipass локально для «сломал-пересоздал» + VPS за $5 для настоящей сети, публичного IP и ssh снаружи.**

### Установка

```bash
brew install --cask multipass
multipass version
```

Именно `--cask`. Разница:

- **формула** (`brew install wget`) — CLI-утилита в песочницу `/opt/homebrew/`, без рута
- **каска** (`brew install --cask X`) — обычное macOS-приложение (.app/.pkg), ставится в систему, просит пароль

Multipass — каска, потому что тянет привилегированный демон `multipassd`, управляющий гипервизором (на M-чипах — Apple Virtualization.framework).

Снос: `brew uninstall --cask multipass`

### Создание и жизненный цикл

```bash
multipass launch --name dev --cpus 2 --memory 4G --disk 20G 24.04
multipass shell dev              # войти внутрь
exit                             # или Ctrl+D — выйти (ВМ продолжает работать)

multipass list                   # имя, состояние, IP
multipass info dev               # детали, потребление

multipass stop dev               # честный shutdown, старт потом 10-20 сек
multipass start dev
multipass suspend dev            # заморозка, старт за 2-3 сек, но состояние протухает
multipass stop --all

multipass delete dev --purge     # снести совсем и вернуть место
```

Без флагов дефолт скромный: 1 CPU / 1 ГБ / 5 ГБ — мало. Внутри юзер `ubuntu`, sudo без пароля.

Промпт — главный индикатор «где я»: `ubuntu@dev:/$` = ты в ВМ, `$` обычный юзер, `#` root.

### Ресурсы: что отдаётся, а что нет

**RAM и CPU — отдаются полностью.** `--memory 4G` это потолок, а не резерв. Выключил — память вернулась. Даже работающая простаивающая ВМ ест ~800 МБ благодаря memory ballooning. vCPU — просто лимит планировщика.

**Диск — не отдаётся.** Образ разреженный (sparse): растёт по мере заполнения, свежая ВМ ~2-3 ГБ вместо 20. Но **обратно не сжимается**: скачал 10 ГБ внутри и удалил — на маке они остались занятыми. Вернуть место только через `delete --purge`.

```bash
sudo du -sh /var/root/Library/Application\ Support/multipassd/
```

### Снапшоты — главная страховка

Порядок: **настроил → снапшот → ломаешь → restore**.

```bash
multipass stop lab
multipass snapshot lab --name clean       # только на остановленной ВМ
multipass start lab

# ...ломаешь...

multipass stop lab                        # --force если повисла намертво
multipass restore lab.clean               # имя через точку
multipass start lab

multipass list --snapshots
```

### Общая папка с маком

```bash
multipass mount ~/projects dev:/home/ubuntu/projects
multipass umount dev:/home/ubuntu/projects
multipass transfer file.txt dev:/tmp/
```

> [!warning] Это одни и те же файлы, а не копия
> `rm -rf` внутри ВМ удалит их на маке. На ломательную ВМ не монтируй ничего. Держи `dev` (рабочую, с маунтом) и `lab` (ломательную, без).

### SSH по-нормальному

```bash
ssh-copy-id ubuntu@<IP из multipass list>
ssh ubuntu@<IP>
```

Заодно первое упражнение: ключи, `~/.ssh/authorized_keys`, права 600.

### Что ВМ не сломает на маке

Форк-бомбы, забитый диск, выжранные inode, OOM — всё упирается в лимиты ВМ. Хост не заметит. Реально опасны только два: **смонтированные каталоги хоста** и **место под sparse-образы**.

### Хранение на внешнем SSD

На macOS — только через симлинк (переменной окружения нет). Диск обязательно **APFS или HFS+**, не exFAT (там нет unix-прав, симлинков и sparse-файлов).

```bash
multipass stop --all
sudo su
launchctl unload /Library/LaunchDaemons/com.canonical.multipassd.plist
mv /var/root/Library/Application\ Support/multipassd /Volumes/MySSD/multipassd
ln -s /Volumes/MySSD/multipassd /var/root/Library/Application\ Support/multipassd
launchctl load /Library/LaunchDaemons/com.canonical.multipassd.plist
exit
```

Диск отключён = Multipass мёртв. Никогда не выдёргивать при работающей ВМ. Для учёбы обычно не нужно: три ВМ это ~10 ГБ.

### Демон в автозагрузке

`multipassd` висит в фоне независимо от того, запущена ли ВМ. Отключается тумблером в System Settings → General → Login Items (под капотом `launchctl disable`/`bootout`). После отключения CLI перестанет работать — он ходит к демону через сокет. Может включиться обратно при обновлении через brew.

---

## Часть 2. Терминал на маке

### Цвета не те (OneDark: зелёный вместо красного)

Причина: скрипт проверяет `$COLORTERM` на `truecolor`/`24bit` и падает в 256-цветный режим, если переменной нет. **SSH не пробрасывает переменные окружения** — по протоколу через `pty-req` едет только `TERM`. Внутри ВМ `COLORTERM` пустой.

Фикс — в `~/.bashrc` **до** подключения скрипта:

```bash
export COLORTERM=truecolor
```

Смежное:

- `source script.sh` (или `. script.sh`) выполняет в текущем шелле, `./script.sh` — в субшелле, и переменные не переживут выход
- `TERM=xterm-ghostty` вызывает `unknown terminal` на серверах без соответствующей terminfo

### Ghostty

```bash
brew install --cask ghostty
touch ~/.config/ghostty/config      # open -e требует существующий файл
open -e ~/.config/ghostty/config
ghostty +show-config | grep font
```

SF Mono лежит внутри Terminal.app, Ghostty его не увидит, пока не скопируешь:

```bash
cp /System/Applications/Utilities/Terminal.app/Contents/Resources/Fonts/SFMono-*.otf ~/Library/Fonts/
```

`+show-config` показывает, что записано в конфиге, но **не** нашёлся ли шрифт — откат на JetBrains Mono происходит молча.
