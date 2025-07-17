
## 背景

如何用tar命令进行多线程并行压缩大文件？

用pbzip2或libzip2 替代原生的bzip2


```bash
tar --use-compress-program=pbzip2 -cpf /run/media/aipc/23debf99-4ffb-4ed4-874b-03935c9f784d/home-aipctar.bz2 --exclude=.cache --exclude=*.tmp aipc/
```
