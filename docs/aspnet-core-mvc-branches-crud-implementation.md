### ASP.NET Core 8.0 MVC를 사용한 CRUD 구현: Branches 테이블 (VisualAcademy 프로젝트)

이 가이드는 **Branches** 테이블에 대해 **ASP.NET Core 8.0 MVC**에서 **CRUD(Create, Read, Update, Delete)** 기능을 구현하는 방법을 안내합니다. 프로젝트는 **VisualAcademy**라는 이름으로 진행되며, **모델**, **리포지토리**, **서비스** 계층을 포함하고, **Web API**와 **jQuery**를 사용해 동적 CRUD를 구현합니다. **Create**와 **Edit**는 **Bootstrap 5** 모달을 사용하여 팝업 형태로 데이터를 입력하고 수정합니다.

---

## 1. **SQL Server 테이블 생성**

`VisualAcademy.SqlServer` 프로젝트가 있으면 열고, 없으면 생성한 후에 다음 테이블을 추가하세요.

`Branches`라는 이름의 테이블을 SQL Server에서 생성합니다. 이 테이블에는 지점의 정보를 저장하기 위한 구조를 포함하며, 고유 식별자(Id), 지점 이름, 위치, 연락처, 설립일, 활성 상태를 관리합니다.

```sql
CREATE TABLE Branches (
    Id INT IDENTITY(1,1) PRIMARY KEY,  -- 자동 증가하는 고유 ID
    BranchName NVARCHAR(100),          -- 지점 이름
    Location NVARCHAR(255),            -- 지점 위치 (주소 또는 지역)
    ContactNumber NVARCHAR(20),        -- 지점 연락처 번호
    EstablishedDate DATE,              -- 지점 설립일
    IsActive BIT                       -- 지점 활성 상태 (1: Active, 0: Inactive)
);
```

### 설명:
- **Id**: 각 지점을 고유하게 식별하는 `INT`형 기본 키입니다. `IDENTITY(1,1)`로 설정하여 자동 증가하도록 합니다.
- **BranchName**: 지점의 이름을 저장하는 `NVARCHAR(100)` 필드입니다.
- **Location**: 지점의 위치를 저장하는 `NVARCHAR(255)` 필드입니다.
- **ContactNumber**: 지점의 연락처 정보를 저장하는 `NVARCHAR(20)` 필드입니다.
- **EstablishedDate**: 지점 설립일을 기록하는 `DATE` 필드입니다.
- **IsActive**: 지점의 활성화 여부를 저장하는 `BIT` 필드로, 1은 활성 상태, 0은 비활성 상태를 나타냅니다.

이 테이블은 Branches 데이터를 관리하기 위한 기본 구조를 제공합니다.

---

## 2. **ASP.NET Core MVC 프로젝트 설정**

`VisualAcademy` 프로젝트가 있으면 열고, 그렇지 않으면 새롭게 만드세요.

### Step 1: **프로젝트 생성**
1. **Visual Studio**를 실행하고, `파일(File)` > `새로 만들기(New)` > `프로젝트(Project)`로 이동합니다.
2. **ASP.NET Core Web App (Model-View-Controller)** 템플릿을 선택하고, 프로젝트 이름을 **VisualAcademy**로 지정한 후, **.NET 8.0**을 사용합니다.

### Step 2: **Entity Framework Core 설치**
1. 패키지 관리 콘솔을 열고, 다음 명령어를 입력하여 **Entity Framework Core**와 **SQL Server** 관련 패키지를 설치합니다.

```bash
Install-Package Microsoft.EntityFrameworkCore.SqlServer
Install-Package Microsoft.EntityFrameworkCore.Tools
```

2. `appsettings.json` 파일에 데이터베이스 연결 문자열을 추가합니다.

```json
"ConnectionStrings": {
  "DefaultConnection": "Server=.;Database=VisualAcademyDb;Trusted_Connection=True;MultipleActiveResultSets=true"
}
```

### Step 3: **Program.cs 설정**
**Entity Framework Core**를 사용하기 위해 **Dependency Injection(DI)**를 설정합니다.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllersWithViews();

// Register DbContext
builder.Services.AddDbContext<BranchesDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Register repository and service for DI
builder.Services.AddScoped<IBranchRepository, BranchRepository>();
builder.Services.AddScoped<BranchService>();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
}

app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Branches}/{action=Index}/{id?}");

app.Run();
```

이렇게 하면 Entity Framework Core를 사용하여 데이터베이스와 상호작용할 수 있도록 설정되고, `Branches` 컨트롤러가 기본적으로 라우팅됩니다.

---

## 3. **Model, Repository, Service 클래스 생성**

### Step 1: **Branches 모델 클래스**
`Models` 폴더에 `Branch.cs` 파일을 생성하고 다음 코드를 작성합니다.

```csharp
public class Branch
{
    public int Id { get; set; }               // 자동 증가하는 고유 ID
    public string BranchName { get; set; }    // 지점 이름
    public string Location { get; set; }      // 지점 위치
    public string ContactNumber { get; set; } // 지점 연락처
    public DateTime EstablishedDate { get; set; } // 지점 설립일
    public bool IsActive { get; set; }        // 지점 활성 상태
}
```

### Step 2: **BranchesDbContext 설정**
`Data` 폴더에 `BranchesDbContext.cs` 파일을 생성하고 다음 코드를 작성합니다.

```csharp
public class BranchesDbContext : DbContext
{
    public BranchesDbContext(DbContextOptions<BranchesDbContext> options) : base(options) { }

    public DbSet<Branch> Branches { get; set; }
}
```

### Step 3: **Repository 인터페이스 및 클래스**
`Repositories` 폴더에 `IBranchRepository.cs` 인터페이스를 생성합니다.

```csharp
public interface IBranchRepository
{
    Task<IEnumerable<Branch>> GetAllBranches();
    Task<Branch> GetBranchById(int id);
    Task AddBranch(Branch branch);
    Task UpdateBranch(Branch branch);
    Task DeleteBranch(int id);
}
```

`BranchRepository.cs` 파일을 생성하고 `IBranchRepository` 인터페이스를 구현합니다.

```csharp
public class BranchRepository : IBranchRepository
{
    private readonly BranchesDbContext _context;

    public BranchRepository(BranchesDbContext context)
    {
        _context = context;
    }

    public async Task<IEnumerable<Branch>> GetAllBranches()
    {
        return await _context.Branches.ToListAsync();
    }

    public async Task<Branch> GetBranchById(int id)
    {
        return await _context.Branches.FindAsync(id);
    }

    public async Task AddBranch(Branch branch)
    {
        _context.Branches.Add(branch);
        await _context.SaveChangesAsync();
    }

    public async Task UpdateBranch(Branch branch)
    {
        _context.Branches.Update(branch);
        await _context.SaveChangesAsync();
    }

    public async Task DeleteBranch(int id)
    {
        var branch = await _context.Branches.FindAsync(id);
        if (branch != null)
        {
            _context.Branches.Remove(branch);
            await _context.SaveChangesAsync();
        }
    }
}
```

### Step 4: **Branch 서비스 클래스**
`Services` 폴더에 `BranchService.cs` 파일을 생성하고 다음 코드를 작성합니다.

```csharp
public class BranchService
{
    private readonly IBranchRepository _repository;

    public BranchService(IBranchRepository repository)
    {
        _repository = repository;
    }

    public async Task<IEnumerable<Branch>> GetAllBranches()
    {
        return await _repository.GetAllBranches();
    }

    public async Task<Branch> GetBranchById(int id)
    {
        return await _repository.GetBranchById(id);
    }

    public async Task AddBranch(Branch branch)
    {
        await _repository.AddBranch(branch);
    }

    public async Task UpdateBranch(Branch branch)
    {
        await _repository.UpdateBranch(branch);
    }

    public async Task DeleteBranch(int id)
    {
        await _repository.DeleteBranch(id);
    }
}
```

---

## 4. **MVC 컨트롤러 생성**

### Step 1: **Branches MVC Controller**
`Controllers` 폴더에 `BranchesController.cs`를 생성하고 다음과 같이 CRUD 액션을 정의합니다.

```csharp
public class BranchesController : Controller
{
    private readonly BranchService _service;

    public BranchesController(BranchService service)
    {
        _service = service;
    }

    // Index 액션: 지점 목록을 보여주는 페이지
    public async Task<IActionResult> Index()
    {
        var branches = await _service.GetAllBranches();
        return View(branches);
    }

    // Create 액션: 새 지점 추가 (모달에서 호출)
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] Branch branch)
    {
        if (ModelState.IsValid)
        {
            await _service.AddBranch(branch);
            return Ok();
        }
        return BadRequest();
    }

    // Edit 액션: 지점 수정 (모달에서 호출)
    [HttpPut("{id}")]
    public async Task<IActionResult> Edit(int id, [FromBody] Branch branch)
    {
        if (id

 != branch.Id || !ModelState.IsValid)
        {
            return BadRequest();
        }

        await _service.UpdateBranch(branch);
        return Ok();
    }

    // Delete 액션: 지점 삭제
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        await _service.DeleteBranch(id);
        return Ok();
    }
}
```

---

## 5. **jQuery와 Bootstrap 5 모달을 사용한 CSHTML 페이지**

### Step 1: **Index.cshtml 설정**
`Views/Branches/Index.cshtml` 파일을 생성하고 다음과 같이 **Create** 및 **Edit** 모달을 사용한 페이지를 작성합니다.

```html
@model IEnumerable<Branch>

@{
    ViewData["Title"] = "Branches Management";
}

<h2>Branches Management</h2>

<!-- Create Button -->
<button type="button" class="btn btn-primary" data-bs-toggle="modal" data-bs-target="#createBranchModal">Add Branch</button>

<table class="table table-striped">
    <thead>
        <tr>
            <th>Branch Name</th>
            <th>Location</th>
            <th>Actions</th>
        </tr>
    </thead>
    <tbody id="branchesTable">
        @foreach (var branch in Model)
        {
            <tr>
                <td>@branch.BranchName</td>
                <td>@branch.Location</td>
                <td>
                    <button class="btn btn-sm btn-primary editBranch" data-id="@branch.Id">Edit</button>
                    <button class="btn btn-sm btn-danger deleteBranch" data-id="@branch.Id">Delete</button>
                </td>
            </tr>
        }
    </tbody>
</table>

<!-- Create Modal -->
<div class="modal fade" id="createBranchModal" tabindex="-1" aria-labelledby="createBranchModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title" id="createBranchModalLabel">Create Branch</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>
      <div class="modal-body">
        <form id="createBranchForm">
          <div class="mb-3">
            <label for="branchName" class="form-label">Branch Name</label>
            <input type="text" class="form-control" id="branchName">
          </div>
          <div class="mb-3">
            <label for="location" class="form-label">Location</label>
            <input type="text" class="form-control" id="location">
          </div>
          <div class="mb-3">
            <label for="contactNumber" class="form-label">Contact Number</label>
            <input type="text" class="form-control" id="contactNumber">
          </div>
          <button type="button" class="btn btn-primary" id="saveBranch">Save</button>
        </form>
      </div>
    </div>
  </div>
</div>

<!-- Edit Modal -->
<div class="modal fade" id="editBranchModal" tabindex="-1" aria-labelledby="editBranchModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title" id="editBranchModalLabel">Edit Branch</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>
      <div class="modal-body">
        <form id="editBranchForm">
          <input type="hidden" id="editBranchId">
          <div class="mb-3">
            <label for="editBranchName" class="form-label">Branch Name</label>
            <input type="text" class="form-control" id="editBranchName">
          </div>
          <div class="mb-3">
            <label for="editLocation" class="form-label">Location</label>
            <input type="text" class="form-control" id="editLocation">
          </div>
          <div class="mb-3">
            <label for="editContactNumber" class="form-label">Contact Number</label>
            <input type="text" class="form-control" id="editContactNumber">
          </div>
          <button type="button" class="btn btn-primary" id="updateBranch">Update</button>
        </form>
      </div>
    </div>
  </div>
</div>

<script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
<script>
    $(document).ready(function() {
        // Load all branches
        function loadBranches() {
            $.get('/Branches/Index', function(data) {
                $('#branchesTable').html(data);
            });
        }

        // Create new branch
        $('#saveBranch').click(function() {
            let branch = {
                BranchName: $('#branchName').val(),
                Location: $('#location').val(),
                ContactNumber: $('#contactNumber').val(),
                IsActive: true
            };

            $.ajax({
                url: '/Branches/Create',
                type: 'POST',
                contentType: 'application/json',
                data: JSON.stringify(branch),
                success: function() {
                    $('#createBranchModal').modal('hide');
                    loadBranches();
                }
            });
        });

        // Load branch for editing
        $(document).on('click', '.editBranch', function() {
            let id = $(this).data('id');

            $.get(`/Branches/GetBranch/${id}`, function(data) {
                $('#editBranchId').val(data.id);
                $('#editBranchName').val(data.branchName);
                $('#editLocation').val(data.location);
                $('#editContactNumber').val(data.contactNumber);
                $('#editBranchModal').modal('show');
            });
        });

        // Update branch
        $('#updateBranch').click(function() {
            let branch = {
                Id: $('#editBranchId').val(),
                BranchName: $('#editBranchName').val(),
                Location: $('#editLocation').val(),
                ContactNumber: $('#editContactNumber').val(),
                IsActive: true
            };

            $.ajax({
                url: `/Branches/Edit/${branch.Id}`,
                type: 'PUT',
                contentType: 'application/json',
                data: JSON.stringify(branch),
                success: function() {
                    $('#editBranchModal').modal('hide');
                    loadBranches();
                }
            });
        });

        // Delete branch
        $(document).on('click', '.deleteBranch', function() {
            let id = $(this).data('id');

            if (confirm('Are you sure to delete this branch?')) {
                $.ajax({
                    url: `/Branches/Delete/${id}`,
                    type: 'DELETE',
                    success: function() {
                        loadBranches();
                    }
                });
            }
        });
    });
</script>
```

---

## 6. **마무리**

1. **Web API**를 통해 데이터 CRUD 작업을 수행하고, **jQuery**와 **Bootstrap 5**를 사용하여 페이지에서 동적으로 **Create**, **Edit**, **Delete** 작업을 모달 창으로 구현합니다.
2. **데이터베이스 마이그레이션**을 통해 데이터베이스를 설정하고 CRUD 기능을 테스트합니다.

```bash
Add-Migration InitBranches
Update-Database
```

이로써 **Branches** 테이블에 대한 CRUD 기능을 구현할 수 있으며, **Bootstrap 5 모달**을 사용하여 사용자가 친숙한 UI에서 지점을 관리할 수 있습니다.

