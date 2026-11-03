# 14. Объектное хранилище S3

Файлы (аватарки, вложения, отчёты, бэкапы) не хранят в Postgres и не кладут на диск сервиса: сервис должен быть stateless, а дисков у подов в Kubernetes по сути нет. Файлы кладут в **объектное хранилище** с API Amazon S3, а в БД хранят только метаданные.

S3-совместимый API есть почти у всех облаков: AWS S3, Yandex Object Storage, Selectel, VK Cloud, Cloudflare R2. Научишься работать с одним — сможешь со всеми.

**Когда проходить:** после файла 13. Для основной части хватит Docker из файла 08, а в последней задаче понадобится очередь из файлов 11–12.

## Понятия

- **Bucket** — контейнер для объектов (имя уникально в рамках хранилища).
- **Object** — файл + метаданные (`Content-Type`, свои заголовки `x-amz-meta-*`). Адресуется **ключом**: `attachments/2026/11/3f2a….pdf`.
- Папок на самом деле нет: `/` в ключе — просто символ, а консоль показывает **префиксы** как папки. `ListObjectsV2` с `Prefix` и `Delimiter` имитирует обход каталогов.
- **ETag** — хеш содержимого (для обычной загрузки — MD5), удобен для проверки целостности и кэширования.
- **Multipart upload** — большой файл грузится частями (минимум 5 МБ на часть, кроме последней), части можно слать параллельно и повторять при ошибке. Незавершённые загрузки занимают место: их чистят правилом lifecycle.
- **Presigned URL** — подписанная ссылка с ограниченным сроком, по которой клиент может загрузить (`PUT`) или скачать (`GET`) конкретный объект **напрямую из хранилища, мимо твоего сервиса**. Ключи доступа при этом остаются у сервиса.
- **Versioning** — хранит старые версии объектов. **Lifecycle rules** — удалять или перемещать в «холодное» хранилище через N дней.
- **Доступ:** bucket policy и IAM. По умолчанию бакет приватный, и пусть таким и остаётся.
- AWS S3 даёт strong read-after-write consistency: после успешной записи чтение сразу видит новую версию. У других S3-совместимых хранилищ гарантии уточняй в документации.

## Локальное хранилище

Долгое время стандартом для локальной разработки был MinIO, но его бесплатная версия больше не поддерживается: официальные Docker-образы перестали публиковать, а репозиторий заархивирован. В старых туториалах ты ещё встретишь `minio/minio`, этот образ больше не скачивается. Для локальной разработки и тестов возьми S3-совместимую замену, например **RustFS**:

```bash
docker run -d --name rustfs -p 9000:9000 -p 9001:9001 \
  -e RUSTFS_ACCESS_KEY=DEVACCESSKEY \
  -e RUSTFS_SECRET_KEY=devsecretkey123 \
  -e RUSTFS_CONSOLE_ENABLE=true \
  -v rustfs-data:/data \
  rustfs/rustfs:latest /data
```
- S3 API: http://localhost:9000, веб-консоль: http://localhost:9001 (вход по access key и secret key).
- Используй именованный том (`rustfs-data`), как выше. Если монтируешь папку с хоста, у неё должен быть владелец с UID 10001, иначе будет `permission denied`.
- Другие варианты: **Garage**, **SeaweedFS**. Твой Go-код от выбора не зависит, меняется только адрес и ключи.

Бакет удобно создать через AWS CLI:
```bash
brew install awscli
export AWS_ACCESS_KEY_ID=DEVACCESSKEY AWS_SECRET_ACCESS_KEY=devsecretkey123 AWS_REGION=us-east-1
export AWS_REQUEST_CHECKSUM_CALCULATION=when_required   # новые версии CLI иначе шлют контрольные суммы, которые понимают не все хранилища
aws --endpoint-url http://localhost:9000 s3 mb s3://attachments
aws --endpoint-url http://localhost:9000 s3 cp ./README.md s3://attachments/test/readme.md
aws --endpoint-url http://localhost:9000 s3 ls s3://attachments --recursive
```

## S3 из Go

Официальный SDK: `github.com/aws/aws-sdk-go-v2` (модули `config`, `credentials`, `service/s3`).

```go
import (
    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/credentials"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

cfg, err := config.LoadDefaultConfig(ctx,
    config.WithRegion("us-east-1"),
    config.WithCredentialsProvider(
        credentials.NewStaticCredentialsProvider(os.Getenv("S3_ACCESS_KEY"), os.Getenv("S3_SECRET_KEY"), "")),
)
if err != nil { return err }

client := s3.NewFromConfig(cfg, func(o *s3.Options) {
    o.BaseEndpoint = aws.String(os.Getenv("S3_ENDPOINT")) // http://localhost:9000; для AWS не задаётся
    o.UsePathStyle = true // http://host/bucket/key вместо http://bucket.host/key
    // SDK по умолчанию добавляет новые контрольные суммы, которые понимают не все S3-совместимые хранилища
    o.RequestChecksumCalculation = aws.RequestChecksumCalculationWhenRequired
    o.ResponseChecksumValidation = aws.ResponseChecksumValidationWhenRequired
})
```

**Загрузка, скачивание, удаление:**
```go
_, err = client.PutObject(ctx, &s3.PutObjectInput{
    Bucket:      aws.String("attachments"),
    Key:         aws.String(key),
    Body:        file, // io.Reader; multipart.File из формы подходит
    ContentType: aws.String("application/pdf"),
})

out, err := client.GetObject(ctx, &s3.GetObjectInput{Bucket: aws.String("attachments"), Key: aws.String(key)})
if err != nil {
    var nsk *types.NoSuchKey // import "github.com/aws/aws-sdk-go-v2/service/s3/types"
    if errors.As(err, &nsk) { /* 404 */ }
    return err
}
defer out.Body.Close()
_, err = io.Copy(w, out.Body) // стримим клиенту, не читаем весь файл в память

_, err = client.DeleteObject(ctx, &s3.DeleteObjectInput{Bucket: aws.String("attachments"), Key: aws.String(key)})
```
Для больших файлов используй multipart. В SDK для этого есть пакет `feature/s3/transfermanager` (старый `feature/s3/manager` объявлен устаревшим).

**Presigned URL:**
```go
presigner := s3.NewPresignClient(client)

up, err := presigner.PresignPutObject(ctx, &s3.PutObjectInput{
    Bucket:      aws.String("attachments"),
    Key:         aws.String(key),
    ContentType: aws.String(contentType),
}, s3.WithPresignExpires(15*time.Minute))
// up.URL — клиент делает PUT по этой ссылке с тем же Content-Type

down, err := presigner.PresignGetObject(ctx, &s3.GetObjectInput{
    Bucket: aws.String("attachments"),
    Key:    aws.String(key),
}, s3.WithPresignExpires(5*time.Minute))
```

### Загрузка через presigned URL (частый вопрос на system design)
```
1. Клиент: POST /tasks/42/attachments {filename, content_type, size}
2. API: проверяет права, тип и размер; генерирует key = attachments/<uuid>;
   пишет в Postgres строку со status = 'pending'; возвращает presigned PUT URL
3. Клиент: PUT файла напрямую в S3 по ссылке
4. Клиент: POST /attachments/{id}/complete
5. API: HeadObject — файл есть и размер совпадает → status = 'ready'
6. Фоновая задача удаляет pending-записи и объекты старше суток
```
Сервис не гоняет через себя гигабайты трафика, а доступ всё равно контролирует.

### Безопасность
- Ключ объекта генерируй сам (`uuid`), а не бери имя файла от пользователя: иначе возможны перезапись чужих файлов и странные символы в путях. Оригинальное имя храни в БД.
- Ограничивай размер и тип файла. `Content-Type` от клиента — только подсказка, при необходимости проверяй первые байты (`http.DetectContentType`).
- Короткий срок жизни presigned-ссылок, приватный бакет, ключи доступа только из переменных окружения или секретов.

## Ресурсы
- docs.aws.amazon.com/AmazonS3/latest/userguide — разделы Objects, Multipart upload, Presigned URLs
- aws.github.io/aws-sdk-go-v2/docs — примеры для Go
- docs.rustfs.com — установка и настройка RustFS

## Задачи
- [ ] ⭐ **AWS CLI.** Подними RustFS, создай бакет, загрузи и скачай файлы, посмотри `ls` с префиксами. Создай presigned-ссылку через `aws s3 presign` и открой её в браузере.
- [ ] ⭐⭐ **Вложения в Tasks API (через сервис).** `POST /tasks/{id}/attachments` (multipart/form-data, до 10 МБ), `GET /attachments/{id}` (стриминг из S3), `DELETE`. Метаданные в Postgres. RustFS в docker-compose, бакет создаётся при старте, если его нет.
- [ ] ⭐⭐ **Presigned-загрузка.** Переделай загрузку на схему с presigned URL из раздела выше, включая шаг `complete` и фоновую очистку незавершённых загрузок.
- [ ] ⭐⭐ **Интеграционные тесты.** Поднимай S3-хранилище в testcontainers (`GenericContainer` с образом RustFS) и тестируй репозиторий вложений: загрузка, скачивание, несуществующий ключ.
- [ ] ⭐⭐ **Интерфейс хранилища.** Вынеси работу с файлами за интерфейс `FileStorage` (`Put`, `Get`, `Delete`, `PresignGet`). Сделай две реализации: S3 и локальная файловая система (для unit-тестов).
- [ ] ⭐⭐⭐ **Генерация отчёта.** `POST /reports` ставит задачу в очередь (Kafka или RabbitMQ). Воркер строит CSV по задачам пользователя, грузит его в S3 и сохраняет ключ. `GET /reports/{id}` возвращает статус и presigned-ссылку на скачивание, когда отчёт готов.

## Вопросы для самопроверки
1. Почему файлы не хранят в БД или на диске сервиса?
2. Что такое presigned URL и какую проблему он решает?
3. Зачем нужен multipart upload?
4. Есть ли в S3 папки?
5. Как сделать так, чтобы пользователь мог скачать только свои файлы, если бакет приватный?
6. Как не оставить «мусор» в хранилище, если клиент начал загрузку и не закончил?
