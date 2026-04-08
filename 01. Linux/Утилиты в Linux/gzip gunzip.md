#linux 
***
### Сжатие файлов
```bash
# Сжатие файла (создает file.txt.gz)
gzip file.txt

# С максимальным сжатием
gzip -9 file.txt

# Сохранение исходного файла
gzip -c file.txt > file.txt.gz

# Сжатие с указанием имени выходного файла
gzip -c file.txt > compressed.gz
```

### Распаковка
```bash
# Распаковка
gzip -d file.txt.gz

# Или используя gunzip
gunzip file.txt.gz

# Распаковка в stdout
gunzip -c file.txt.gz
```
### Основные флаги:
- `-d` - распаковка
- `-c` - вывод в stdout
- `-1` до `-9` - уровень сжатия (1 - быстро, 9 - лучшее сжатие)
- `-v` - вывод информации о сжатии
- `-l` - информация о сжатом файле
