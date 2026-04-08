#linux 
***
### Создание архивов
```bash
# Создание tar архива
tar -cf archive.tar file1.txt file2.txt

# Создание сжатого gzip архива (.tar.gz)
tar -czf archive.tar.gz directory/

# Создание сжатого bzip2 архива (.tar.bz2)
tar -cjf archive.tar.bz2 directory/

# Создание сжатого xz архива (.tar.xz)
tar -cJf archive.tar.xz directory/

# Рекурсивное архивирование с сохранением прав
tar -cpzf archive.tar.gz directory/
```

### Распаковка
```bash
# Распаковка tar архива
tar -xf archive.tar

# Распаковка в определенную директорию
tar -xf archive.tar -C /target/directory/

# Распаковка gzip архива
tar -xzf archive.tar.gz

# Распаковка с выводом списка файлов
tar -xzvf archive.tar.gz
```

### Просмотр содержимого
```bash
# Просмотр содержимого
tar -tf archive.tar

# Подробный просмотр
tar -tvf archive.tar.gz
```

### Основные флаги:
- `-c` - создание архива
- `-x` - извлечение из архива
- `-t` - просмотр содержимого
- `-f` - указание файла архива
- `-z` - работа с gzip сжатием
- `-j` - работа с bzip2 сжатием
- `-J` - работа с xz сжатием
- `-v` - подробный вывод
- `-p` - сохранение прав доступа
- `-C` - извлечение в указанную директорию

## Практические примеры

### Создание бэкапа
```bash
# Создание сжатого архива с датой в имени
tar -czf backup-$(date +%Y%m%d).tar.gz /path/to/important/data
```

### Распаковка с фильтрацией
```bash
# Распаковка только определенных файлов
tar -xzf archive.tar.gz "*.txt"
```

### Сравнение сжатия
```bash
# Создание архивов с разными методами сжатия
tar -cf archive.tar largefile.txt
tar -czf archive.tar.gz largefile.txt
tar -cjf archive.tar.bz2 largefile.txt
tar -cJf archive.tar.xz largefile.txt

# Проверка размеров
ls -lh archive.tar*
```

### Пакетное сжатие
```bash
# Сжатие всех txt файлов в директории
gzip *.txt

# Или с сохранением оригиналов
for file in *.txt; do
    gzip -c "$file" > "${file}.gz"
done
```

***
