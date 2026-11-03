# 15. gRPC и Protobuf

Основной способ синхронного общения микросервисов на Go. Работает поверх HTTP/2, данные в бинарном формате Protobuf, клиент и сервер генерируются из `.proto`-файла.

## Плюсы и минусы относительно REST
- \+ строгий контракт, кодогенерация, быстрее и компактнее JSON, стриминг, дедлайны из коробки
- − не читается глазами, из браузера напрямую не вызвать (нужен grpc-gateway или Connect), сложнее отлаживать

## Установка на Mac
```bash
brew install protobuf grpcurl
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```
Альтернатива `protoc`: инструмент **buf** (`brew install bufbuild/buf/buf`): удобнее, с линтером и проверкой обратной совместимости.

## Пример
`proto/users/v1/users.proto`:
```proto
syntax = "proto3";

package users.v1;

option go_package = "github.com/you/project/gen/users/v1;usersv1";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (stream User); // серверный стриминг
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest { int64 id = 1; }
message GetUserResponse { User user = 1; }
message ListUsersRequest { int32 limit = 1; }
```

Генерация:
```bash
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       proto/users/v1/users.proto
```

Сервер:
```go
type server struct {
    usersv1.UnimplementedUserServiceServer
    repo UserRepo
}

func (s *server) GetUser(ctx context.Context, req *usersv1.GetUserRequest) (*usersv1.GetUserResponse, error) {
    u, err := s.repo.GetByID(ctx, req.GetId())
    if errors.Is(err, ErrNotFound) {
        return nil, status.Error(codes.NotFound, "user not found")
    }
    if err != nil {
        return nil, status.Error(codes.Internal, "internal error")
    }
    return &usersv1.GetUserResponse{User: &usersv1.User{Id: u.ID, Name: u.Name, Email: u.Email}}, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":9090")
    gs := grpc.NewServer(grpc.ChainUnaryInterceptor(loggingInterceptor))
    usersv1.RegisterUserServiceServer(gs, &server{repo: newRepo()})
    reflection.Register(gs) // чтобы работал grpcurl
    gs.Serve(lis)
}
```

Клиент:
```go
conn, err := grpc.NewClient("localhost:9090", grpc.WithTransportCredentials(insecure.NewCredentials()))
client := usersv1.NewUserServiceClient(conn)
ctx, cancel := context.WithTimeout(ctx, time.Second)
defer cancel()
resp, err := client.GetUser(ctx, &usersv1.GetUserRequest{Id: 1})
```

Проверка из терминала:
```bash
grpcurl -plaintext localhost:9090 list
grpcurl -plaintext -d '{"id": 1}' localhost:9090 users.v1.UserService/GetUser
```

## Что знать
- 4 типа RPC: unary, server streaming, client streaming, bidirectional streaming.
- Статус-коды gRPC (`codes.NotFound`, `InvalidArgument`, `Unavailable`, `DeadlineExceeded`…) и как они соотносятся с HTTP-кодами.
- **Дедлайны** передаются по цепочке сервисов через context.
- **Interceptors** (аналог middleware): логирование, метрики, аутентификация, recover.
- **Metadata** (аналог заголовков).
- Правила эволюции proto: не менять номера полей, не переиспользовать удалённые (`reserved`).
- `grpc-gateway` — автоматический REST-фасад над gRPC-сервисом.

## Ресурсы
- grpc.io/docs/languages/go/quickstart
- protobuf.dev/programming-guides/proto3
- buf.build/docs

## Задачи
- [ ] ⭐ **Quickstart.** Пройди официальный quickstart grpc.io для Go и вызови сервис через grpcurl.
- [ ] ⭐⭐ **Users service.** Отдельный сервис пользователей на gRPC: `CreateUser`, `GetUser`, `ListUsers` (серверный стриминг), хранилище в Postgres. Правильные gRPC-коды ошибок, валидация входа → `InvalidArgument`.
- [ ] ⭐⭐ **Interceptors.** Unary interceptor логирования (метод, код, длительность), recover от panic, проверка токена из metadata (`authorization`).
- [ ] ⭐⭐ **Связать сервисы.** Tasks API при создании задачи проверяет через gRPC, что пользователь существует. Таймаут 500 мс; если Users service недоступен, вернуть клиенту 503.
- [ ] ⭐⭐ **Тесты.** Тесты gRPC-сервера через `google.golang.org/grpc/test/bufconn` (сервер в памяти без сети).
- [ ] ⭐⭐⭐ **Bidi-стриминг.** Простой чат: клиенты подключаются через bidirectional stream, сообщение одного рассылается всем. CLI-клиент читает stdin.
- [ ] ⭐⭐⭐ **grpc-gateway.** Добавь к Users service REST-фасад через grpc-gateway и сгенерируй OpenAPI-спецификацию.

## Вопросы для самопроверки
1. Чем gRPC отличается от REST? Когда что выбрать?
2. Почему gRPC использует HTTP/2?
3. Что будет, если изменить номер поля в proto?
4. Как работают дедлайны в цепочке сервисов?
5. Что такое interceptor?
