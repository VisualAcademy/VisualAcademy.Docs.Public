## ASP.NET Core 8.0에서 CORS를 여러 번 설정하는 방법

### 1. `UseCors`의 호출 제한과 정책 설정

`app.UseCors`는 ASP.NET Core의 미들웨어 파이프라인에서 **한 번만 호출되어야** 합니다. 여러 번 호출하면 마지막에 호출된 `UseCors`만 적용되므로, 여러 CORS 정책을 설정하고자 할 때는 **하나의 호출에 각 정책을 적용하는 방법**을 사용해야 합니다.

그러나, 여러 CORS 정책을 정의한 후, 각 컨트롤러나 엔드포인트에 대해 다른 CORS 정책을 적용할 수 있습니다. 이를 위해서는 각 컨트롤러나 액션 메서드에서 `[EnableCors]` 속성을 사용해야 합니다.

### 2. 여러 CORS 정책 정의 및 사용 방법

#### 2.1 `Program.cs`에서 여러 CORS 정책 정의

먼저, `Program.cs`에서 여러 CORS 정책을 정의합니다. 예를 들어, "AllowAll"과 "AllowSpecificOrigins" 두 가지 정책을 정의할 수 있습니다.

```csharp
var builder = WebApplication.CreateBuilder(args);

// 여러 CORS 정책을 추가
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", builder =>
    {
        builder.AllowAnyOrigin()
               .AllowAnyMethod()
               .AllowAnyHeader();
    });

    options.AddPolicy("AllowSpecificOrigins", builder =>
    {
        builder.WithOrigins("https://dotnetnote.com")
               .AllowAnyMethod()
               .AllowAnyHeader();
    });
});

// Add services to the container.
builder.Services.AddControllers();

var app = builder.Build();

// 글로벌 CORS 정책 적용 (예: AllowAll)
app.UseCors("AllowAll");

app.UseAuthorization();

app.MapControllers();

app.Run();
```

이렇게 하면 두 가지 CORS 정책을 정의하고, 전체 애플리케이션에 "AllowAll" 정책을 적용하게 됩니다.

#### 2.2 특정 엔드포인트에 다른 CORS 정책 적용

특정 컨트롤러나 엔드포인트에 다른 CORS 정책을 적용하려면 `[EnableCors]` 속성을 사용합니다. 

```csharp
using Microsoft.AspNetCore.Cors;
using Microsoft.AspNetCore.Mvc;

[Route("api/[controller]")]
[ApiController]
public class SampleController : ControllerBase
{
    // 이 엔드포인트는 AllowSpecificOrigins 정책을 적용
    [HttpGet]
    [EnableCors("AllowSpecificOrigins")]
    public IActionResult Get()
    {
        return Ok("This endpoint allows specific origins.");
    }

    // 이 엔드포인트는 글로벌 정책을 따름 (AllowAll)
    [HttpPost]
    public IActionResult Post()
    {
        return Ok("This endpoint follows the global AllowAll policy.");
    }
}
```

위 코드에서 `[EnableCors("AllowSpecificOrigins")]` 속성을 사용하여 특정 엔드포인트에서 "AllowSpecificOrigins" 정책을 적용합니다.

### 3. 핵심 요점

- `app.UseCors`는 파이프라인에서 **한 번만** 호출되어야 합니다.
- 여러 CORS 정책을 정의한 후, 글로벌 정책을 적용하거나 특정 엔드포인트에서 `[EnableCors]`로 다른 정책을 사용할 수 있습니다.
- 특정 엔드포인트에서 CORS를 비활성화하려면 `[DisableCors]` 속성을 사용할 수 있습니다.

이렇게 하면 애플리케이션의 CORS 설정을 더 세밀하게 제어할 수 있습니다.
