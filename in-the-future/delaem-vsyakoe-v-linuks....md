# Делаем всякое в линукс...

## Запуск программы в бэкграунде на сервере

```bash
nohup python3 ./<name>.py &
```

После закрытия терминала (например, с SSH) программа продолжит свою работу.

## Сжатие видео

```bash
ffmpeg -i <not_compressed>.mp4 -c:v libx264 -preset slow -crf 23 -c:a copy <compressed>.mp4
```

2.2 GB -> 112 MB

