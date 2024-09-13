# ASP.NET Core Minimal APIs를 활용하여 TODO API 만들기: 단계별 가이드

이 가이드에서는 ASP.NET Core의 Minimal APIs를 사용하여 간단한 TODO API를 만드는 과정을 살펴봅니다. Minimal APIs는 간단하고 가벼운 방식으로 API를 구축할 수 있도록 해주며, 특히 작은 규모의 프로젝트나 프로토타이핑에 유용합니다. 이 글에서는 기본적인 CRUD 기능을 제공하는 TODO API를 만드는 것부터 시작해, 데이터 저장을 위한 in-memory 데이터베이스 설정과 API 호출을 테스트하는 방법까지 단계별로 다루겠습니다.

## 목차
1. [프로젝트 설정 및 시작하기](#1-프로젝트-설정-및-시작하기)
2. [In-Memory 데이터베이스 설정](#2-in-memory-데이터베이스-설정)
3. [Minimal API로 TODO 엔드포인트 만들기](#3-minimal-api로-todo-엔드포인트-만들기)
4. [HTTP 파일을 사용하여 API 테스트하기](#4-http-파일을-사용하여-api-테스트하기)
5. [마무리 및 다음 단계](#5-마무리-및-다음-단계)

---

## 1. 프로젝트 설정 및 시작하기

먼저, ASP.NET Core 프로젝트를 생성합니다. Minimal APIs는 ASP.NET Core 6.0부터 지원되므로, 이 버전 이상의 SDK가 설치되어 있어야 합니다.

### 프로젝트 생성

다음 명령어를 통해 새로운 웹 프로젝트를 생성합니다:

```bash
dotnet new web -n DotNetNote
```

`DotNetNote` 디렉터리로 이동합니다:

```bash
cd DotNetNote
```

이제, 이 프로젝트를 열고 `Program.cs` 파일을 수정할 준비를 합니다.

## 2. In-Memory 데이터베이스 설정

이 예제에서는 간단한 in-memory 데이터베이스를 사용하여 TODO 데이터를 저장할 것입니다. 이를 위해 `Program.cs` 파일에서 서비스를 추가하고, 초기 데이터를 삽입하겠습니다.

### `Program.cs` 설정

```csharp
public partial class Program
{
    public static async Task Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);

        // In-memory 데이터베이스 설정
        builder.Services.AddDbContext<DotNetNote.Components.TodoContext>(options =>
            options.UseInMemoryDatabase("TodoList"));

        var app = builder.Build();

        // 초기 데이터 삽입
        using (var scope = app.Services.CreateScope())
        {
            var scopedServices = scope.ServiceProvider;
            var todoContext = scopedServices.GetRequiredService<DotNetNote.Components.TodoContext>();

            if (todoContext != null)
            {
                todoContext.Todos.Add(new DotNetNote.Components.Todo { Id = -2, Title = "Angular", IsDone = false });
                todoContext.Todos.Add(new DotNetNote.Components.Todo { Id = -1, Title = "ASP.NET Core", IsDone = true });
                todoContext.SaveChanges();
            }
        }

        // 이후 API 엔드포인트를 설정할 것입니다.
    }
}
```

### 설명

1. **`AddDbContext`**: `builder.Services.AddDbContext`를 통해 In-memory 데이터베이스를 `TodoContext`로 설정합니다.
2. **초기 데이터 삽입**: 앱이 시작될 때 몇 개의 TODO 항목을 미리 추가하여 데이터베이스를 초기화합니다.

## 3. Minimal API로 TODO 엔드포인트 만들기

이제 Minimal API를 사용하여 TODO 목록을 관리하는 엔드포인트를 만들어보겠습니다. 여기서는 기본적인 CRUD(Create, Read, Update, Delete) 기능을 구현합니다.

### 엔드포인트 정의

`Program.cs` 파일에서 다음 코드를 추가하여 TODO API를 만듭니다:

```csharp
#region Minimal APIs를 사용한 TODO 엔드포인트

// TODO 리스트를 저장할 메모리 리스트
var todos = new List<TodoRecord>();

// 전체 TODO 목록 가져오기
app.MapGet("/todos", () => todos);

// 특정 ID의 TODO 항목 가져오기
app.MapGet("/todos/{id}", Results<Ok<TodoRecord>, NotFound> (int id) =>
{
    var targetTodo = todos.SingleOrDefault(x => x.Id == id);
    return targetTodo is null
        ? TypedResults.NotFound()
        : TypedResults.Ok(targetTodo);
});

// 새로운 TODO 항목 추가
app.MapPost("/todos", (TodoRecord task) =>
{
    todos.Add(task);
    return TypedResults.Created($"/todos/{task.Id}", task);
});

// 특정 ID의 TODO 항목 삭제
app.MapDelete("/todos/{id}", (int id) =>
{
    todos.RemoveAll(t => id == t.Id);
    return TypedResults.NoContent();
});

#endregion
```

### 설명

1. **`MapGet("/todos")`**: 전체 TODO 목록을 가져오는 GET 요청을 처리합니다.
2. **`MapGet("/todos/{id}")`**: 특정 ID의 TODO 항목을 가져옵니다. 해당 ID가 없으면 404 응답을 반환합니다.
3. **`MapPost("/todos")`**: 새로운 TODO 항목을 추가하는 POST 요청을 처리합니다. 새 항목이 추가되면 201(Created) 응답을 반환합니다.
4. **`MapDelete("/todos/{id}")`**: 특정 ID의 TODO 항목을 삭제하는 DELETE 요청을 처리합니다.

## 4. HTTP 파일을 사용하여 API 테스트하기

API를 테스트하기 위해 HTTP 파일을 만들 수 있습니다. 이 파일을 통해 다양한 API 요청을 쉽게 테스트할 수 있습니다. Visual Studio에서는 `.http` 파일을 통해 이러한 요청을 보낼 수 있습니다.

### `Todo.http` 파일 예제

```
# 전체 TODO 목록 가져오기
GET https://localhost:5001/todos

###

# 새로운 TODO 항목 추가
POST https://localhost:5001/todos
Content-Type: application/json

{
  "id": 1,
  "title": "Todo Demo",
  "isCompleted": false
}

###

# 특정 ID의 TODO 항목 가져오기
GET https://localhost:5001/todos/1

###

# 특정 ID의 TODO 항목 삭제
DELETE https://localhost:5001/todos/1
```

### 설명

- **GET**: `GET https://localhost:5001/todos` 요청으로 전체 TODO 목록을 가져옵니다.
- **POST**: 새로운 TODO 항목을 추가하는 요청입니다. JSON 형식의 데이터를 전송합니다.
- **DELETE**: 특정 ID의 TODO 항목을 삭제하는 요청입니다.

이러한 파일을 통해 API의 각 기능을 쉽게 테스트할 수 있습니다.

## 5. 마무리 및 다음 단계

이 가이드에서는 Minimal APIs를 사용하여 간단한 TODO API를 만드는 방법을 배웠습니다. 이를 통해 RESTful API를 빠르게 구축할 수 있으며, 필요에 따라 더 복잡한 로직과 데이터베이스 연동, 인증/인가 기능을 추가할 수 있습니다.

다음 단계로는 다음과 같은 기능을 추가해볼 수 있습니다:
- **인증/인가**: 특정 사용자만 TODO 항목을 생성, 수정, 삭제할 수 있도록 보안을 강화합니다.
- **데이터베이스 연동**: In-memory 데이터베이스 대신 실제 데이터베이스(MySQL, SQL Server 등)로 데이터를 저장하도록 변경합니다.
- **프론트엔드 연동**: Angular, React, 또는 Blazor와 같은 프론트엔드 프레임워크를 사용하여 이 API를 소비하는 클라이언트 애플리케이션을 구축해보세요.

이 가이드를 통해 Minimal APIs를 활용한 간단한 웹 API 구축에 대한 기본적인 이해를 얻었기를 바랍니다. Minimal APIs는 작은 규모의 애플리케이션이나 프로토타이핑에 적합한 도구이므로 적극적으로 활용해보세요!
