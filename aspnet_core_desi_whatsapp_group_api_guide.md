# ASP.NET Core Web API for Desi WhatsApp Group App

# Step-by-Step Complete Guide

This guide helps you build a scalable backend API using:

- ASP.NET Core 9 Web API
- Entity Framework Core
- MySQL
- JWT Authentication
- Clean Architecture Style
- REST APIs for Mobile Apps

---

# Project Goal

Build APIs for:

- User Login/Register
- Create WhatsApp Groups
- Search Groups
- Category Listing
- City Based Groups
- Join Requests
- Admin Moderation
- Trending Groups
- Notifications

---

# Step 1 — Create ASP.NET Core API Project

## Install .NET SDK

Download:

```txt
https://dotnet.microsoft.com/download
```

Check version:

```bash
dotnet --version
```

---

# Create Project

```bash
dotnet new webapi -n DesiWhatsappApi
```

Open project:

```bash
cd DesiWhatsappApi
```

---

# Step 2 — Install Packages

Install Entity Framework + MySQL:

```bash
dotnet add package Pomelo.EntityFrameworkCore.MySql
```

Install JWT Authentication:

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

Install Swagger:

```bash
dotnet add package Swashbuckle.AspNetCore
```

---

# Step 3 — Folder Structure

Create folders:

```txt
/Controllers
/Models
/Data
/Services
/DTOs
/Repositories
/Helpers
```

---

# Step 4 — Configure MySQL Connection

## appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "server=localhost;database=desiwhatsapp;user=root;password=yourpassword"
  },
  "Jwt": {
    "Key": "THIS_IS_SUPER_SECRET_KEY_123",
    "Issuer": "desiapi"
  }
}
```

---

# Step 5 — Create Database Models

# Models/User.cs

```csharp
using System.ComponentModel.DataAnnotations;

namespace DesiWhatsappApi.Models
{
    public class User
    {
        [Key]
        public int UserId { get; set; }

        [Required]
        public string Username { get; set; }

        [Required]
        public string Email { get; set; }

        [Required]
        public string PasswordHash { get; set; }

        public string Avatar { get; set; }

        public DateTime CreatedDate { get; set; } = DateTime.UtcNow;
    }
}
```

---

# Models/WhatsappGroup.cs

```csharp
using System.ComponentModel.DataAnnotations;

namespace DesiWhatsappApi.Models
{
    public class WhatsappGroup
    {
        [Key]
        public int GroupId { get; set; }

        public string GroupName { get; set; }

        public string Description { get; set; }

        public string WhatsappLink { get; set; }

        public string Category { get; set; }

        public string Country { get; set; }

        public string City { get; set; }

        public string Language { get; set; }

        public int UserId { get; set; }

        public bool IsApproved { get; set; } = false;

        public int TotalMembers { get; set; }

        public DateTime CreatedDate { get; set; } = DateTime.UtcNow;
    }
}
```

---

# Step 6 — Create DbContext

# Data/AppDbContext.cs

```csharp
using Microsoft.EntityFrameworkCore;
using DesiWhatsappApi.Models;

namespace DesiWhatsappApi.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options)
            : base(options)
        {
        }

        public DbSet<User> Users { get; set; }

        public DbSet<WhatsappGroup> WhatsappGroups { get; set; }
    }
}
```

---

# Step 7 — Configure Program.cs

```csharp
using DesiWhatsappApi.Data;
using Microsoft.EntityFrameworkCore;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseMySql(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        ServerVersion.AutoDetect(
            builder.Configuration.GetConnectionString("DefaultConnection")
        )
    )
);

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = false,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"])
            )
        };
    });

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

---

# Step 8 — Create Register API

# Controllers/AuthController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using DesiWhatsappApi.Data;
using DesiWhatsappApi.Models;

namespace DesiWhatsappApi.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class AuthController : ControllerBase
    {
        private readonly AppDbContext _context;

        public AuthController(AppDbContext context)
        {
            _context = context;
        }

        [HttpPost("register")]
        public IActionResult Register(User model)
        {
            _context.Users.Add(model);
            _context.SaveChanges();

            return Ok(new
            {
                message = "User registered successfully"
            });
        }
    }
}
```

---

# Step 9 — Create Login API with JWT

```csharp
[HttpPost("login")]
public IActionResult Login(User model)
{
    var user = _context.Users
        .FirstOrDefault(x =>
            x.Email == model.Email &&
            x.PasswordHash == model.PasswordHash);

    if (user == null)
    {
        return Unauthorized();
    }

    var tokenHandler = new JwtSecurityTokenHandler();

    var key = Encoding.UTF8.GetBytes(_config["Jwt:Key"]);

    var tokenDescriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity(new[]
        {
            new Claim("UserId", user.UserId.ToString())
        }),

        Expires = DateTime.UtcNow.AddDays(30),

        SigningCredentials = new SigningCredentials(
            new SymmetricSecurityKey(key),
            SecurityAlgorithms.HmacSha256Signature
        ),

        Issuer = _config["Jwt:Issuer"]
    };

    var token = tokenHandler.CreateToken(tokenDescriptor);

    return Ok(new
    {
        token = tokenHandler.WriteToken(token)
    });
}
```

---

# Step 10 — Create Group API

# Controllers/GroupsController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Authorization;
using DesiWhatsappApi.Data;
using DesiWhatsappApi.Models;

namespace DesiWhatsappApi.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class GroupsController : ControllerBase
    {
        private readonly AppDbContext _context;

        public GroupsController(AppDbContext context)
        {
            _context = context;
        }

        [Authorize]
        [HttpPost("create")]
        public IActionResult CreateGroup(WhatsappGroup model)
        {
            _context.WhatsappGroups.Add(model);
            _context.SaveChanges();

            return Ok(new
            {
                message = "Group added successfully"
            });
        }

        [HttpGet]
        public IActionResult GetGroups()
        {
            var groups = _context.WhatsappGroups
                .Where(x => x.IsApproved)
                .OrderByDescending(x => x.CreatedDate)
                .ToList();

            return Ok(groups);
        }
    }
}
```

---

# Step 11 — Search API

```csharp
[HttpGet("search")]
public IActionResult Search(string keyword)
{
    var groups = _context.WhatsappGroups
        .Where(x =>
            x.GroupName.Contains(keyword) ||
            x.City.Contains(keyword) ||
            x.Language.Contains(keyword)
        )
        .ToList();

    return Ok(groups);
}
```

---

# Step 12 — Category API

```csharp
[HttpGet("category/{category}")]
public IActionResult ByCategory(string category)
{
    var groups = _context.WhatsappGroups
        .Where(x => x.Category == category)
        .ToList();

    return Ok(groups);
}
```

---

# Step 13 — Migration Commands

Install EF Tool:

```bash
dotnet tool install --global dotnet-ef
```

Create migration:

```bash
dotnet ef migrations add InitialCreate
```

Update database:

```bash
dotnet ef database update
```

---

# Step 14 — Run API

```bash
dotnet run
```

Swagger URL:

```txt
https://localhost:5001/swagger
```

---

# Step 15 — Recommended Production Features

## Security
- Password hashing with BCrypt
- Rate limiting
- API throttling
- Cloudflare protection
- Captcha

---

# Advanced APIs You Should Add

## Group Reporting API
Users can report spam groups.

## Admin APIs
- Approve group
- Delete spam
- Ban users

## Trending Algorithm
Sort groups based on:
- Daily joins
- Clicks
- Shares
- Activity

## Analytics APIs
Track:
- Page views
- Country traffic
- Top categories

## Notification APIs
Send:
- Event alerts
- Festival alerts
- Meetup reminders

---

# Recommended Database Indexes

```sql
CREATE INDEX idx_group_category
ON WhatsappGroups(Category);

CREATE INDEX idx_group_city
ON WhatsappGroups(City);

CREATE INDEX idx_group_language
ON WhatsappGroups(Language);

CREATE INDEX idx_group_approved
ON WhatsappGroups(IsApproved);
```

---

# Suggested Scalable Architecture

Mobile App
↓
Cloudflare CDN
↓
ASP.NET Core API
↓
Redis Cache
↓
MySQL Database

---

# APIs Your Mobile App Will Use

| API | Method |
|---|---|
| /api/auth/register | POST |
| /api/auth/login | POST |
| /api/groups | GET |
| /api/groups/create | POST |
| /api/groups/search | GET |
| /api/groups/category | GET |

---

# Future Expansion

You can later expand into:

- Desi social network
- Community feed
- Marketplace
- Matrimony
- Event booking
- AI moderation
- Local city communities
- Voice chat rooms
- Live events

---

# Recommended Deployment

## Linux VPS

Use:

- Ubuntu 24
- NGINX
- Kestrel
- Docker
- Cloudflare

---

# Docker Example

## Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "DesiWhatsappApi.dll"]
```

---

# Next Recommended Steps

After this, build:

1. Refresh token system
2. OTP login
3. Firebase push notifications
4. Redis caching
5. Real-time chat using SignalR
6. AI spam detection
7. Elasticsearch search
8. Geo-location based discovery
9. CDN optimization
10. Viral referral system

