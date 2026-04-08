### Создание архива
```bash
# Базовое создание архива
zip archive.zip file1.txt file2.txt

# Рекурсивное архивирование директории
zip -r archive.zip directory/

# С максимальным сжатием
zip -9 archive.zip file.txt

# С паролем
zip -e secure.zip file.txt
```

### Распаковка
```bash
# Распаковка архива
unzip archive.zip

# В определенную директорию
unzip archive.zip -d /target/directory/

# Просмотр содержимого без распаковки
unzip -l archive.zip
```

### Основные флаги:
- `-r` - рекурсивное архивирование
- `-e` - шифрование с паролем
- `-9` - максимальное сжатие
- `-v` - подробный вывод
- `-q` - тихий режим