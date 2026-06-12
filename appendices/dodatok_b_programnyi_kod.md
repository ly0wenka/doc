# ДОДАТОК Б

# ЛІСТИНГ ПРОГРАМНОГО КОДУ ІНФОРМАЦІЙНОЇ СИСТЕМИ GAMESINFOSYS

Структуру додатка сформовано за зразком програмного додатка І. О. Леонова. Нижче наведено фактичний код розробленого застосунку, згрупований за рівнями архітектури. Автоматично створені каталоги `bin` і `obj`, сторонні бібліотеки та великий демонстраційний JSON-набір не включено.

## Б.1 Файл проєкту

Файл: `GamesInfoSys/GamesInfoSys/GamesInfoSys.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.8">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.8" />
  </ItemGroup>

</Project>
```

## Б.2 Конфігурація застосунку

Файл: `GamesInfoSys/GamesInfoSys/Program.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.AspNetCore.Localization;
using System.Globalization;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddRazorPages();
builder.Services.AddLocalization();
builder.Services.AddMemoryCache();
builder.Services.AddSingleton<GamesInfoSys.Services.UiText>();

builder.Services.Configure<GamesInfoSys.Services.RawgOptions>(builder.Configuration.GetSection("Rawg"));
builder.Services.AddHttpClient<GamesInfoSys.Services.RawgClient>((sp, client) =>
{
    var options = sp.GetRequiredService<Microsoft.Extensions.Options.IOptions<GamesInfoSys.Services.RawgOptions>>().Value;
    client.BaseAddress = new Uri(options.BaseUrl);
    client.Timeout = TimeSpan.FromSeconds(15);
});

builder.Services.Configure<GamesInfoSys.Services.PricingOptions>(builder.Configuration.GetSection("Pricing"));
builder.Services.AddSingleton<GamesInfoSys.Services.RegionResolver>();

builder.Services.AddDbContext<GamesInfoSys.Data.AppDbContext>(o =>
{
    o.UseSqlite("Data Source=app.db");
});

builder.Services.Configure<GamesInfoSys.Services.CurrencyOptions>(builder.Configuration.GetSection("Currency"));
builder.Services.AddHttpClient<GamesInfoSys.Services.NbuRatesClient>((sp, client) =>
{
    var options = sp.GetRequiredService<Microsoft.Extensions.Options.IOptions<GamesInfoSys.Services.CurrencyOptions>>().Value;
    client.BaseAddress = new Uri(options.NbuBaseUrl);
    client.Timeout = TimeSpan.FromSeconds(10);
});
builder.Services.AddScoped<GamesInfoSys.Services.CurrencyConverter>();

builder.Services.Configure<GamesInfoSys.Services.ScrapingOptions>(builder.Configuration.GetSection("Scraping"));
builder.Services.AddHttpClient<GamesInfoSys.Services.UaMarketplaceScraper>(client =>
{
    client.Timeout = TimeSpan.FromSeconds(20);
    client.DefaultRequestHeaders.UserAgent.ParseAdd("GamesInfoSys/1.0 (UA price tracker; scraping MVP)");
    client.DefaultRequestHeaders.AcceptLanguage.ParseAdd("uk-UA,uk;q=0.9,en;q=0.6");
});

builder.Services.AddHttpClient<GamesInfoSys.Services.SteamStoreClient>(client =>
{
    client.BaseAddress = new Uri("https://store.steampowered.com/");
    client.Timeout = TimeSpan.FromSeconds(15);
});
builder.Services.AddHttpClient<GamesInfoSys.Services.CheapSharkClient>(client =>
{
    client.BaseAddress = new Uri("https://www.cheapshark.com/");
    client.Timeout = TimeSpan.FromSeconds(15);
    // CheapShark requires a descriptive User-Agent to avoid accidental blocking.
    client.DefaultRequestHeaders.UserAgent.ParseAdd("GamesInfoSys/1.0 (no-key; marketplace redirects)");
});
builder.Services.AddScoped<GamesInfoSys.Services.OfferAggregator>();

var app = builder.Build();

var supportedCultures = new[]
{
    new CultureInfo("en"),
    new CultureInfo("uk")
};

var localizationOptions = new RequestLocalizationOptions
{
    DefaultRequestCulture = new RequestCulture("en"),
    SupportedCultures = supportedCultures,
    SupportedUICultures = supportedCultures
};

localizationOptions.RequestCultureProviders.Insert(0, new QueryStringRequestCultureProvider());

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseRequestLocalization(localizationOptions);

app.UseRouting();

app.UseAuthorization();

app.MapGet("/set-language", (HttpContext httpContext, string culture, string? returnUrl) =>
{
    var normalizedCulture = supportedCultures.Any(x => string.Equals(x.Name, culture, StringComparison.OrdinalIgnoreCase))
        ? culture
        : "en";

    httpContext.Response.Cookies.Append(
        CookieRequestCultureProvider.DefaultCookieName,
        CookieRequestCultureProvider.MakeCookieValue(new RequestCulture(normalizedCulture)),
        new CookieOptions
        {
            Expires = DateTimeOffset.UtcNow.AddYears(1),
            IsEssential = true,
            Path = "/"
        });

    return Results.LocalRedirect(string.IsNullOrWhiteSpace(returnUrl) ? "/" : returnUrl);
});

app.MapStaticAssets();
app.MapRazorPages()
   .WithStaticAssets();

using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<GamesInfoSys.Data.AppDbContext>();
    db.Database.EnsureCreated();
}

app.Run();
```

## Б.3 Контекст бази даних

Файл: `GamesInfoSys/GamesInfoSys/Data/AppDbContext.cs`

```csharp
using GamesInfoSys.Data.Entities;
using Microsoft.EntityFrameworkCore;

namespace GamesInfoSys.Data;

public sealed class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }

    public DbSet<TrackedGame> TrackedGames => Set<TrackedGame>();
    public DbSet<StoreOffer> StoreOffers => Set<StoreOffer>();
    public DbSet<OfferPricePoint> OfferPricePoints => Set<OfferPricePoint>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<TrackedGame>(b =>
        {
            b.HasKey(x => x.Id);
            b.Property(x => x.Name).HasMaxLength(256);
            b.HasIndex(x => x.RawgGameId);
        });

        modelBuilder.Entity<StoreOffer>(b =>
        {
            b.HasKey(x => x.Id);
            b.Property(x => x.Store).HasMaxLength(64);
            b.Property(x => x.Platform).HasMaxLength(32);
            b.Property(x => x.Region).HasMaxLength(8);
            b.Property(x => x.Currency).HasMaxLength(8);
            b.Property(x => x.Title).HasMaxLength(256);
            b.Property(x => x.Url).HasMaxLength(1024);
            b.HasIndex(x => new { x.Store, x.ExternalId, x.Region }).IsUnique();
            b.HasIndex(x => new { x.TrackedGameId, x.Platform, x.Region });
        });

        modelBuilder.Entity<OfferPricePoint>(b =>
        {
            b.HasKey(x => x.Id);
            b.Property(x => x.Currency).HasMaxLength(8);
            b.HasIndex(x => new { x.StoreOfferId, x.AtUtc });
        });
    }
}
```

## Б.4 Сутність відеогри

Файл: `GamesInfoSys/GamesInfoSys/Data/Entities/TrackedGame.cs`

```csharp
namespace GamesInfoSys.Data.Entities;

public sealed class TrackedGame
{
    public long Id { get; set; }
    public int? RawgGameId { get; set; }
    public string? Name { get; set; }

    public string? SteamAppId { get; set; }
    public string? PsnProductId { get; set; }
    public string? XboxProductId { get; set; }
    public string? NintendoProductId { get; set; }
    public string? EpicOfferId { get; set; }
    public string? GogProductId { get; set; }

    public DateTime CreatedUtc { get; set; } = DateTime.UtcNow;
}
```

## Б.5 Сутність магазинної пропозиції

Файл: `GamesInfoSys/GamesInfoSys/Data/Entities/StoreOffer.cs`

```csharp
namespace GamesInfoSys.Data.Entities;

public sealed class StoreOffer
{
    public long Id { get; set; }

    public long TrackedGameId { get; set; }
    public TrackedGame? TrackedGame { get; set; }

    public string Store { get; set; } = "";
    public string Platform { get; set; } = "";
    public string Region { get; set; } = "";

    public string ExternalId { get; set; } = "";
    public string Title { get; set; } = "";
    public string Url { get; set; } = "";

    public string? Currency { get; set; }
    public long? PriceMinor { get; set; }
    public long? OriginalPriceMinor { get; set; }

    public DateTime LastSeenUtc { get; set; } = DateTime.UtcNow;
    public DateTime LastUpdatedUtc { get; set; } = DateTime.UtcNow;
}
```

## Б.6 Сутність історії ціни

Файл: `GamesInfoSys/GamesInfoSys/Data/Entities/OfferPricePoint.cs`

```csharp
namespace GamesInfoSys.Data.Entities;

public sealed class OfferPricePoint
{
    public long Id { get; set; }

    public long StoreOfferId { get; set; }
    public StoreOffer? StoreOffer { get; set; }

    public DateTime AtUtc { get; set; } = DateTime.UtcNow;
    public string Currency { get; set; } = "";
    public long PriceMinor { get; set; }
    public long? OriginalPriceMinor { get; set; }
}
```

## Б.7 Моделі каталогу

Файл: `GamesInfoSys/GamesInfoSys/Models/GameModels.cs`

```csharp
namespace GamesInfoSys.Models;

public sealed record GameSummary(
    int Id,
    string Name,
    string? Released,
    double? Rating,
    int? RatingsCount,
    string? BackgroundImage,
    IReadOnlyList<string> Genres,
    IReadOnlyList<string> Platforms
);

public sealed record GameDetails(
    int Id,
    string Name,
    string? Released,
    double? Rating,
    int? RatingsCount,
    int? Metacritic,
    string? BackgroundImage,
    string? Website,
    string? DescriptionPlain,
    IReadOnlyList<string> Genres,
    IReadOnlyList<string> Platforms,
    IReadOnlyList<string> Developers,
    IReadOnlyList<string> Publishers,
    IReadOnlyList<string> Tags,
    IReadOnlyList<ExternalStoreLink> ExternalStores
);

public sealed record GameScreenshot(string? Image);

public sealed record ExternalStoreLink(string Store, string Url);
```

## Б.8 Модель сторінки каталогу

Файл: `GamesInfoSys/GamesInfoSys/Pages/Games/Index.cshtml.cs`

```csharp
using GamesInfoSys.Models;
using GamesInfoSys.Services;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace GamesInfoSys.Pages.Games;

public sealed class IndexModel : PageModel
{
    private readonly RawgClient _rawg;

    public IndexModel(RawgClient rawg)
    {
        _rawg = rawg;
    }

    [BindProperty(SupportsGet = true, Name = "q")]
    public string? Query { get; set; }

    public bool IsDemoMode => _rawg.IsDemoMode;

    public IReadOnlyList<GameSummary> Games { get; private set; } = [];

    public async Task OnGetAsync()
    {
        Games = await _rawg.SearchGamesAsync(Query);
    }
}
```

## Б.9 Подання каталогу

Файл: `GamesInfoSys/GamesInfoSys/Pages/Games/Index.cshtml`

```html
@page "/Games"
@model GamesInfoSys.Pages.Games.IndexModel
@{
    ViewData["Title"] = Text["BrowseTitle"];
}

<div class="d-flex flex-column flex-md-row align-items-md-end justify-content-between gap-3 mb-3">
    <div>
        <h1 class="h3 mb-1">@Text["BrowseHeading"]</h1>
        <div class="text-body-secondary">
            @if (Model.IsDemoMode)
            {
                <span class="badge text-bg-warning">@Text["DemoMode"]</span>
                <span class="ms-2">@Text["SetRawgApiKey"]</span>
            }
            else
            {
                <span class="badge text-bg-success">@Text["Live"]</span>
                <span class="ms-2">@Text["DataViaRawg"]</span>
            }
        </div>
    </div>
    <form method="get" class="gis-search w-100 w-md-auto">
        <div class="input-group">
            <input class="form-control" type="search" name="q" value="@Model.Query" placeholder="@Text["SearchPlaceholder"]" />
            <button class="btn btn-primary" type="submit">@Text["Search"]</button>
        </div>
        <div class="d-flex justify-content-between mt-2">
            <div class="text-body-secondary small">@Text["TipTopGames"]</div>
            @if (!string.IsNullOrWhiteSpace(Model.Query))
            {
                <a class="small" asp-page="/Games/Index">@Text["Clear"]</a>
            }
        </div>
    </form>
</div>

@if (Model.Games.Count == 0)
{
    <div class="alert alert-info">@Text["NoGamesFound"]</div>
}
else
{
    <div class="row g-3">
        @foreach (var g in Model.Games)
        {
            <div class="col-12 col-sm-6 col-lg-4 col-xl-3">
                <a class="card h-100 text-decoration-none gis-card" asp-page="/Games/Details" asp-route-id="@g.Id">
                    @if (!string.IsNullOrWhiteSpace(g.BackgroundImage))
                    {
                        <img class="card-img-top gis-cover" src="@g.BackgroundImage" alt="@($"{g.Name} {Text["CoverAlt"]}")" loading="lazy"
                             onerror="this.onerror=null;this.src='/img/cover-placeholder.svg';" />
                    }
                    else
                    {
                        <img class="card-img-top gis-cover" src="/img/cover-placeholder.svg" alt="@($"{g.Name} {Text["CoverAlt"]}")" loading="lazy" />
                    }
                    <div class="card-body">
                        <div class="d-flex justify-content-between align-items-start gap-2">
                            <h2 class="h6 mb-1 text-body">@g.Name</h2>
                            @if (g.Rating is not null)
                            {
                                <span class="badge text-bg-secondary">* @g.Rating</span>
                            }
                        </div>
                        <div class="small text-body-secondary">
                            @if (!string.IsNullOrWhiteSpace(g.Released))
                            {
                                <span>@g.Released</span>
                            }
                            @if (g.Platforms.Count > 0)
                            {
                                <span class="ms-2">- @string.Join(", ", g.Platforms.Take(3))</span>
                            }
                        </div>
                        @if (g.Genres.Count > 0)
                        {
                            <div class="mt-2 gis-tags">
                                @foreach (var genre in g.Genres.Take(3))
                                {
                                    <span class="badge text-bg-light border">@genre</span>
                                }
                            </div>
                        }
                    </div>
                </a>
            </div>
        }
    </div>
}
```

## Б.10 Модель детальної сторінки

Файл: `GamesInfoSys/GamesInfoSys/Pages/Games/Details.cshtml.cs`

```csharp
using GamesInfoSys.Models;
using GamesInfoSys.Services;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace GamesInfoSys.Pages.Games;

public sealed class DetailsModel : PageModel
{
    private readonly RawgClient _rawg;
    private readonly OfferAggregator _offers;
    private readonly CurrencyConverter _fx;
    private readonly UiText _text;

    public DetailsModel(RawgClient rawg, OfferAggregator offers, CurrencyConverter fx, UiText text)
    {
        _rawg = rawg;
        _offers = offers;
        _fx = fx;
        _text = text;
    }

    public bool IsDemoMode => _rawg.IsDemoMode;

    public GameDetails? Game { get; private set; }
    public IReadOnlyList<GameScreenshot> Screenshots { get; private set; } = [];
    public IReadOnlyList<Data.Entities.StoreOffer> Offers { get; private set; } = [];
    public Dictionary<long, decimal?> OfferUahMajor { get; private set; } = new();
    public string GameNameForSearch => Game?.Name ?? "";
    public Data.Entities.StoreOffer? BestMarketplaceOffer { get; private set; }

    [BindProperty(SupportsGet = false)]
    public string? SteamAppIdOrUrl { get; set; }

    public async Task<IActionResult> OnGetAsync(int id)
    {
        Game = await _rawg.GetGameAsync(id);
        if (Game is null)
            return NotFound();

        await _offers.SyncOffersForRawgGameAsync(id, Game.Name, Game);
        Offers = await _offers.GetOffersForRawgGameAsync(id);
        BestMarketplaceOffer = Offers
            .Where(o => o.Store != "Steam" && o.PriceMinor is not null)
            .OrderBy(o => o.PriceMinor)
            .FirstOrDefault();
        OfferUahMajor = await ComputeUahAsync(Offers);

        Screenshots = await _rawg.GetScreenshotsAsync(id);
        return Page();
    }

    public async Task<IActionResult> OnPostSetSteamAsync(int id)
    {
        Game = await _rawg.GetGameAsync(id);
        if (Game is null)
            return NotFound();

        var parsed = TryParseSteamAppId(SteamAppIdOrUrl);
        if (string.IsNullOrWhiteSpace(parsed))
        {
            ModelState.AddModelError(nameof(SteamAppIdOrUrl), _text["SteamValidation"]);
            Offers = await _offers.GetOffersForRawgGameAsync(id);
            return Page();
        }

        await _offers.SetSteamAppIdAsync(id, parsed);
        await _offers.SyncOffersForRawgGameAsync(id, Game.Name);

        return RedirectToPage("/Games/Details", new { id });
    }

    private async Task<Dictionary<long, decimal?>> ComputeUahAsync(IReadOnlyList<Data.Entities.StoreOffer> offers)
    {
        var dict = new Dictionary<long, decimal?>();
        foreach (var o in offers)
        {
            if (o.PriceMinor is null || string.IsNullOrWhiteSpace(o.Currency))
            {
                dict[o.Id] = null;
                continue;
            }

            var major = o.PriceMinor.Value / 100m;
            dict[o.Id] = await _fx.ToUahAsync(major, o.Currency);
        }
        return dict;
    }

    private static string? TryParseSteamAppId(string? input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return null;

        input = input.Trim();
        if (int.TryParse(input, out _))
            return input;

        if (!Uri.TryCreate(input, UriKind.Absolute, out var uri) || uri is null)
            return null;

        if (!uri.Host.Contains("steampowered.com", StringComparison.OrdinalIgnoreCase))
            return null;

        var segments = uri.AbsolutePath.Trim('/').Split('/', StringSplitOptions.RemoveEmptyEntries);
        for (var i = 0; i < segments.Length - 1; i++)
        {
            if (!segments[i].Equals("app", StringComparison.OrdinalIgnoreCase))
                continue;
            var id = segments[i + 1];
            return int.TryParse(id, out _) ? id : null;
        }

        return null;
    }
}
```

## Б.11 Подання детальної сторінки

Файл: `GamesInfoSys/GamesInfoSys/Pages/Games/Details.cshtml`

```html
@page "/Games/{id:int}"
@model GamesInfoSys.Pages.Games.DetailsModel
@{
    ViewData["Title"] = Model.Game?.Name ?? Text["GameTitleFallback"];
}

@if (Model.Game is null)
{
    <div class="alert alert-warning">@Text["GameNotFound"]</div>
    <a class="btn btn-outline-secondary" asp-page="/Games/Index">@Text["Back"]</a>
}
else
{
    <div class="row g-4">
        <div class="col-12 col-lg-4">
            <div class="card">
                @if (!string.IsNullOrWhiteSpace(Model.Game.BackgroundImage))
                {
                    <img class="card-img-top gis-cover" src="@Model.Game.BackgroundImage" alt="@($"{Model.Game.Name} {Text["CoverAlt"]}")"
                         onerror="this.onerror=null;this.src='/img/cover-placeholder.svg';" />
                }
                else
                {
                    <img class="card-img-top gis-cover" src="/img/cover-placeholder.svg" alt="@($"{Model.Game.Name} {Text["CoverAlt"]}")" />
                }
                <div class="card-body">
                    <h1 class="h4 mb-2">@Model.Game.Name</h1>
                    <div class="d-flex flex-wrap gap-2 mb-3">
                        @if (!string.IsNullOrWhiteSpace(Model.Game.Released))
                        {
                            <span class="badge text-bg-light border">@Text["Released"]: @Model.Game.Released</span>
                        }
                        @if (Model.Game.Metacritic is not null)
                        {
                            <span class="badge text-bg-light border">@Text["Metacritic"]: @Model.Game.Metacritic</span>
                        }
                        @if (Model.Game.Rating is not null)
                        {
                            <span class="badge text-bg-light border">@Text["Rating"]: * @Model.Game.Rating</span>
                        }
                    </div>

                    @if (!string.IsNullOrWhiteSpace(Model.Game.Website))
                    {
                        <div class="mb-3">
                            <a class="btn btn-sm btn-primary w-100" href="@Model.Game.Website" target="_blank" rel="noreferrer">@Text["OfficialWebsite"]</a>
                        </div>
                    }

                    <a class="btn btn-outline-secondary w-100" asp-page="/Games/Index">@Text["BackToBrowse"]</a>
                </div>
            </div>
        </div>

        <div class="col-12 col-lg-8">
            <div class="card mb-3">
                <div class="card-body">
                    <div class="d-flex justify-content-between align-items-start gap-3 flex-wrap">
                        <div>
                            <h2 class="h5 mb-1">@Text["PricesHeading"]</h2>
                            <div class="text-body-secondary small">@Text["PricesSubheading"]</div>
                        </div>
                        <a class="btn btn-sm btn-outline-secondary" asp-page="/Games/Index">@Text["BrowseButton"]</a>
                    </div>

                    @if (Model.IsDemoMode && Model.Game.ExternalStores.Count > 0)
                    {
                        <div class="alert alert-warning mt-3 mb-0">
                            @Text["DemoModeNote"]
                        </div>
                    }

                    @if (Model.Game.ExternalStores.Count > 0)
                    {
                        <div class="mt-3 d-flex flex-wrap gap-2">
                            @foreach (var s in Model.Game.ExternalStores)
                            {
                                <a class="btn btn-sm btn-outline-secondary" href="@s.Url" target="_blank" rel="noreferrer">@Text.Format("OpenStore", s.Store)</a>
                            }
                        </div>
                    }

                    @if (Model.BestMarketplaceOffer is not null && Model.BestMarketplaceOffer.PriceMinor is not null)
                    {
                        <div class="alert alert-warning mt-3 mb-0">
                            @Text["BestMarketplaceDeal"] <strong>@FormatMoney(Model.BestMarketplaceOffer.PriceMinor.Value, Model.BestMarketplaceOffer.Currency ?? "USD")</strong>
                            @if (Model.OfferUahMajor.TryGetValue(Model.BestMarketplaceOffer.Id, out var uah) && uah is not null)
                            {
                                <span class="ms-2">(Approx. @($"{uah.Value:0.00} UAH"))</span>
                            }
                            <a class="ms-2" href="@Model.BestMarketplaceOffer.Url" target="_blank" rel="noreferrer">@Text["Open"]</a>
                        </div>
                    }

                    @if (Model.Offers.Count == 0)
                    {
                        <div class="alert alert-info mt-3 mb-0">@Text["NoOffers"]</div>
                    }
                    else
                    {
                        <div class="table-responsive mt-3">
                            <table class="table table-sm align-middle">
                                <thead>
                                    <tr>
                                        <th>@Text["Platform"]</th>
                                        <th>@Text["Store"]</th>
                                        <th>@Text["Region"]</th>
                                        <th>@Text["Price"]</th>
                                        <th></th>
                                    </tr>
                                </thead>
                                <tbody>
                                    @foreach (var o in Model.Offers)
                                    {
                                        var isMarketplace = o.Store != "Steam";
                                        <tr>
                                            <td>@o.Platform</td>
                                            <td>
                                                @o.Store
                                                @if (isMarketplace)
                                                {
                                                    <span class="badge text-bg-warning ms-2">@Text["Marketplace"]</span>
                                                }
                                            </td>
                                            <td>@o.Region</td>
                                            <td>
                                                @if (o.PriceMinor is null || string.IsNullOrWhiteSpace(o.Currency))
                                                {
                                                    <span class="text-body-secondary">-</span>
                                                }
                                                else
                                                {
                                                    <span class="fw-semibold">@FormatMoney(o.PriceMinor.Value, o.Currency)</span>
                                                    @if (o.OriginalPriceMinor is not null && o.OriginalPriceMinor.Value > o.PriceMinor.Value)
                                                    {
                                                        <span class="text-body-secondary ms-2"><s>@FormatMoney(o.OriginalPriceMinor.Value, o.Currency)</s></span>
                                                    }

                                                    @if (o.Currency != "UAH" && Model.OfferUahMajor.TryGetValue(o.Id, out var uah) && uah is not null)
                                                    {
                                                        <div class="small text-body-secondary mt-1">Approx. @($"{uah.Value:0.00} UAH")</div>
                                                    }
                                                }
                                            </td>
                                            <td class="text-end">
                                                <a class="btn btn-sm btn-primary" href="@o.Url" target="_blank" rel="noreferrer">
                                                    @(isMarketplace ? Text["OpenDeal"] : Text["Open"])
                                                </a>
                                            </td>
                                        </tr>
                                    }
                                </tbody>
                            </table>
                        </div>
                    }

                    <div class="border-top pt-3 mt-3">
                        <div class="d-flex align-items-center justify-content-between gap-2 flex-wrap">
                            <h3 class="h6 mb-0">@Text["SteamMappingOptional"]</h3>
                            <button class="btn btn-sm btn-outline-primary" type="button" data-bs-toggle="collapse" data-bs-target="#steamMap" aria-expanded="false" aria-controls="steamMap">
                                @Text["AddOrChange"]
                            </button>
                        </div>

                        <div class="collapse mt-3" id="steamMap">
                            <form method="post" asp-page-handler="SetSteam" class="row g-2 align-items-end">
                                <div class="col-12 col-md-8">
                                    <label class="form-label" asp-for="SteamAppIdOrUrl">@Text["SteamLinkOrAppId"]</label>
                                    <input class="form-control" asp-for="SteamAppIdOrUrl" placeholder="@Text["SteamInputPlaceholder"]" />
                                    <div class="text-danger small"><span asp-validation-for="SteamAppIdOrUrl"></span></div>
                                    <div class="form-text">@Text["SteamTip"]</div>
                                </div>
                                <div class="col-12 col-md-auto">
                                    <button class="btn btn-outline-primary" type="submit">@Text["SaveRefresh"]</button>
                                </div>
                            </form>
                        </div>
                    </div>
                </div>
            </div>

            <div class="card mb-3">
                <div class="card-body">
                    <h2 class="h5">@Text["About"]</h2>
                    <p class="mb-0">
                        @(Model.Game.DescriptionPlain ?? Text["NoDescription"])
                    </p>
                </div>
            </div>

            <div class="card mb-3">
                <div class="card-body">
                    <h2 class="h5">@Text["FindInUkraine"]</h2>
                    <div class="text-body-secondary small mb-3">@Text["FindInUkraineText"]</div>

                    <div class="d-flex flex-wrap gap-2">
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://www.cheapshark.com/search#q:", Model.GameNameForSearch)">
                            @Text["CheapSharkDeals"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://www.bing.com/search?q=site%3Astore.playstation.com+uk-ua+", Model.GameNameForSearch)">
                            @Text["PsStoreOfficial"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://www.bing.com/search?q=site%3Axbox.com+", Model.GameNameForSearch)">
                            @Text["XboxStoreOfficial"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://www.bing.com/search?q=site%3Anintendo.com+en-za+", Model.GameNameForSearch + " Nintendo Switch")">
                            @Text["NintendoStoreOfficial"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://rozetka.com.ua/search/?text=", Model.GameNameForSearch)">
                            Rozetka
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://prom.ua/ua/search?search_term=", Model.GameNameForSearch)">
                            Prom.ua
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://www.olx.ua/d/uk/list/q-", Slugify(Model.GameNameForSearch))">
                            OLX
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://www.youtube.com/results?search_query=", Model.GameNameForSearch + " review")">
                            @Text["YouTubeReview"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://www.youtube.com/results?search_query=", Model.GameNameForSearch + " gameplay")">
                            @Text["YouTubeGameplay"]
                        </a>
                    </div>

                    <hr class="my-3" />

                    <div class="text-body-secondary small mb-2">@Text["PlatformSearches"]</div>
                    <div class="d-flex flex-wrap gap-2">
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://prom.ua/ua/search?search_term=", Model.GameNameForSearch + " playstation")">
                            @Text["PromPs"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://www.olx.ua/uk/elektronika/igry-i-igrovye-pristavki/pristavki/q-", Slugify(Model.GameNameForSearch + " playstation"))">
                            @Text["OlxPs"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://rozetka.com.ua/search/?text=", Model.GameNameForSearch + " playstation")">
                            @Text["RozetkaPs"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://prom.ua/ua/search?search_term=", Model.GameNameForSearch + " xbox")">
                            @Text["PromXbox"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://www.olx.ua/uk/elektronika/igry-i-igrovye-pristavki/pristavki/q-", Slugify(Model.GameNameForSearch + " xbox"))">
                            @Text["OlxXbox"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://rozetka.com.ua/search/?text=", Model.GameNameForSearch + " xbox")">
                            @Text["RozetkaXbox"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://prom.ua/ua/search?search_term=", Model.GameNameForSearch + " nintendo switch")">
                            @Text["PromSwitch"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://www.olx.ua/uk/elektronika/igry-i-igrovye-pristavki/pristavki/q-", Slugify(Model.GameNameForSearch + " nintendo switch"))">
                            @Text["OlxSwitch"]
                        </a>
                        <a class="btn btn-sm btn-outline-secondary" target="_blank" rel="noreferrer"
                           href="@BuildSearchUrl("https://rozetka.com.ua/search/?text=", Model.GameNameForSearch + " nintendo switch")">
                            @Text["RozetkaSwitch"]
                        </a>
                    </div>
                </div>
            </div>

            <div class="row g-3">
                <div class="col-12 col-md-6">
                    <div class="card h-100">
                        <div class="card-body">
                            <h2 class="h6">@Text["Genres"]</h2>
                            @if (Model.Game.Genres.Count == 0)
                            {
                                <div class="text-body-secondary">-</div>
                            }
                            else
                            {
                                <div class="gis-tags">
                                    @foreach (var x in Model.Game.Genres)
                                    {
                                        <span class="badge text-bg-light border">@x</span>
                                    }
                                </div>
                            }
                        </div>
                    </div>
                </div>
                <div class="col-12 col-md-6">
                    <div class="card h-100">
                        <div class="card-body">
                            <h2 class="h6">@Text["Platforms"]</h2>
                            @if (Model.Game.Platforms.Count == 0)
                            {
                                <div class="text-body-secondary">-</div>
                            }
                            else
                            {
                                <div class="gis-tags">
                                    @foreach (var x in Model.Game.Platforms)
                                    {
                                        <span class="badge text-bg-light border">@x</span>
                                    }
                                </div>
                            }
                        </div>
                    </div>
                </div>
            </div>

            @if (!Model.IsDemoMode && Model.Screenshots.Count > 0)
            {
                <div class="card mt-3">
                    <div class="card-body">
                        <h2 class="h5 mb-3">@Text["Screenshots"]</h2>
                        <div class="row g-2">
                            @foreach (var s in Model.Screenshots)
                            {
                                if (string.IsNullOrWhiteSpace(s.Image)) { continue; }
                                <div class="col-6 col-md-4">
                                    <a href="@s.Image" target="_blank" rel="noreferrer">
                                        <img class="img-fluid rounded gis-shot" src="@s.Image" alt="@Text["ScreenshotAlt"]" loading="lazy" />
                                    </a>
                                </div>
                            }
                        </div>
                    </div>
                </div>
            }
        </div>
    </div>
}

@functions {
    private static string FormatMoney(long minor, string currency)
    {
        var major = minor / 100m;
        return $"{major:0.00} {currency}";
    }

    private static string BuildSearchUrl(string prefix, string query)
    {
        query ??= "";
        return prefix + Uri.EscapeDataString(query);
    }

    private static string Slugify(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            return "";
        value = value.Trim();
        value = value.Replace('’', '\'').Replace('–', '-').Replace('—', '-');
        value = System.Text.RegularExpressions.Regex.Replace(value, "\\s+", "-");
        value = System.Text.RegularExpressions.Regex.Replace(value, "[^a-zA-Z0-9\\-]+", "");
        return value.ToLowerInvariant();
    }
}
```

## Б.12 Клієнт каталогу RAWG і demo-режиму

Файл: `GamesInfoSys/GamesInfoSys/Services/RawgClient.cs`

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Text.RegularExpressions;
using GamesInfoSys.Models;
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.Options;

namespace GamesInfoSys.Services;

public sealed class RawgClient
{
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNameCaseInsensitive = true,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
    };

    private readonly HttpClient _http;
    private readonly IMemoryCache _cache;
    private readonly RawgOptions _options;
    private readonly IWebHostEnvironment _env;
    private readonly SteamStoreClient _steamStore;

    public RawgClient(
        HttpClient http,
        IMemoryCache cache,
        IOptions<RawgOptions> options,
        IWebHostEnvironment env,
        SteamStoreClient steamStore)
    {
        _http = http;
        _cache = cache;
        _options = options.Value;
        _env = env;
        _steamStore = steamStore;
    }

    public bool IsDemoMode => string.IsNullOrWhiteSpace(_options.ApiKey) && _options.UseDemoDataWhenNoApiKey;

    public async Task<IReadOnlyList<GameSummary>> SearchGamesAsync(
        string? query,
        int page = 1,
        int pageSize = 24,
        string? ordering = "-rating")
    {
        if (IsDemoMode)
            return await SearchDemoAsync(query);

        page = Math.Clamp(page, 1, 1000);
        pageSize = Math.Clamp(pageSize, 1, 40);
        ordering ??= "-rating";

        var cacheKey = $"rawg:search:q={query}|p={page}|ps={pageSize}|o={ordering}";
        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);

            var url = new UriBuilder(new Uri(_http.BaseAddress!, "games"));
            url.Query = BuildQuery(new Dictionary<string, string?>
            {
                ["key"] = _options.ApiKey,
                ["search"] = string.IsNullOrWhiteSpace(query) ? null : query,
                ["page"] = page.ToString(),
                ["page_size"] = pageSize.ToString(),
                ["ordering"] = ordering
            });

            using var res = await _http.GetAsync(url.Uri);
            res.EnsureSuccessStatusCode();

            await using var stream = await res.Content.ReadAsStreamAsync();
            var payload = await JsonSerializer.DeserializeAsync<RawgListResponse<RawgGameSummary>>(stream, JsonOptions);
            return (IReadOnlyList<GameSummary>)(payload?.Results ?? [])
                .Select(MapSummary)
                .ToList();
        }) ?? [];
    }

    public async Task<GameDetails?> GetGameAsync(int id)
    {
        if (IsDemoMode)
            return await GetDemoDetailsAsync(id);

        var cacheKey = $"rawg:game:id={id}";
        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1);

            var url = new UriBuilder(new Uri(_http.BaseAddress!, $"games/{id}"));
            url.Query = BuildQuery(new Dictionary<string, string?>
            {
                ["key"] = _options.ApiKey
            });

            using var res = await _http.GetAsync(url.Uri);
            if (res.StatusCode == System.Net.HttpStatusCode.NotFound)
                return null;
            res.EnsureSuccessStatusCode();

            await using var stream = await res.Content.ReadAsStreamAsync();
            var raw = await JsonSerializer.DeserializeAsync<RawgGameDetails>(stream, JsonOptions);
            if (raw is null)
                return null;

            return MapDetails(raw);
        });
    }

    public async Task<IReadOnlyList<GameScreenshot>> GetScreenshotsAsync(int id)
    {
        if (IsDemoMode)
            return [];

        var cacheKey = $"rawg:screens:id={id}";
        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1);

            var url = new UriBuilder(new Uri(_http.BaseAddress!, $"games/{id}/screenshots"));
            url.Query = BuildQuery(new Dictionary<string, string?>
            {
                ["key"] = _options.ApiKey,
                ["page_size"] = "8"
            });

            using var res = await _http.GetAsync(url.Uri);
            res.EnsureSuccessStatusCode();

            await using var stream = await res.Content.ReadAsStreamAsync();
            var payload = await JsonSerializer.DeserializeAsync<RawgListResponse<RawgScreenshot>>(stream, JsonOptions);
            return (IReadOnlyList<GameScreenshot>)(payload?.Results ?? [])
                .Select(s => new GameScreenshot(s.Image))
                .ToList();
        }) ?? [];
    }

    private static string BuildQuery(Dictionary<string, string?> values)
    {
        var encoded = values
            .Where(kvp => !string.IsNullOrWhiteSpace(kvp.Value))
            .Select(kvp => $"{Uri.EscapeDataString(kvp.Key)}={Uri.EscapeDataString(kvp.Value!)}");
        return string.Join("&", encoded);
    }

    private static GameSummary MapSummary(RawgGameSummary raw)
    {
        return new GameSummary(
            raw.Id,
            raw.Name ?? "(Unknown)",
            raw.Released,
            raw.Rating,
            raw.RatingsCount,
            raw.BackgroundImage,
            (raw.Genres ?? []).Select(g => g.Name).Where(n => !string.IsNullOrWhiteSpace(n)).ToList(),
            (raw.Platforms ?? []).Select(p => p.Platform?.Name).Where(n => !string.IsNullOrWhiteSpace(n)).Cast<string>().Distinct().ToList()
        );
    }

    private static GameDetails MapDetails(RawgGameDetails raw)
    {
        return new GameDetails(
            raw.Id,
            raw.Name ?? "(Unknown)",
            raw.Released,
            raw.Rating,
            raw.RatingsCount,
            raw.Metacritic,
            raw.BackgroundImage,
            raw.Website,
            StripHtmlToPlainText(raw.Description),
            (raw.Genres ?? []).Select(g => g.Name).Where(n => !string.IsNullOrWhiteSpace(n)).ToList(),
            (raw.Platforms ?? []).Select(p => p.Platform?.Name).Where(n => !string.IsNullOrWhiteSpace(n)).Cast<string>().Distinct().ToList(),
            (raw.Developers ?? []).Select(d => d.Name).Where(n => !string.IsNullOrWhiteSpace(n)).Distinct().ToList(),
            (raw.Publishers ?? []).Select(p => p.Name).Where(n => !string.IsNullOrWhiteSpace(n)).Distinct().ToList(),
            (raw.Tags ?? []).Select(t => t.Name).Where(n => !string.IsNullOrWhiteSpace(n)).Distinct().ToList(),
            (raw.Stores ?? [])
                .Select(s => new ExternalStoreLink(s.Store?.Name ?? "", s.Url ?? ""))
                .Where(x => !string.IsNullOrWhiteSpace(x.Store) && !string.IsNullOrWhiteSpace(x.Url))
                .Distinct()
                .ToList()
        );
    }

    private static string? StripHtmlToPlainText(string? html)
    {
        if (string.IsNullOrWhiteSpace(html))
            return null;

        var text = Regex.Replace(html, "<[^>]+>", " ");
        text = System.Net.WebUtility.HtmlDecode(text);
        text = Regex.Replace(text, "\\s+", " ").Trim();
        return string.IsNullOrWhiteSpace(text) ? null : text;
    }

    private async Task<IReadOnlyList<GameSummary>> SearchDemoAsync(string? query)
    {
        var games = await LoadDemoAsync();
        if (string.IsNullOrWhiteSpace(query))
            return games;

        query = query.Trim();
        return games
            .Where(g => g.Name.Contains(query, StringComparison.OrdinalIgnoreCase))
            .ToList();
    }

    private async Task<GameDetails?> GetDemoDetailsAsync(int id)
    {
        var games = await LoadDemoAsync();
        var game = games.FirstOrDefault(g => g.Id == id);
        if (game is null)
            return null;

        var demo = await LoadDemoRawAsync();
        var item = demo.FirstOrDefault(x => x.Id == id);

        var external = new List<ExternalStoreLink>();
        AddExternal(external, "Steam", item?.SteamUrl);
        AddExternal(external, "PlayStation", item?.PsnUrl);
        AddExternal(external, "Xbox", item?.XboxUrl);
        AddExternal(external, "Nintendo", item?.NintendoUrl);
        AddExternal(external, "Epic Games", item?.EpicUrl);
        AddExternal(external, "GOG", item?.GogUrl);

        return new GameDetails(
            game.Id,
            game.Name,
            game.Released,
            game.Rating,
            game.RatingsCount,
            null,
            await ResolveDemoBackgroundImageAsync(item, game.BackgroundImage),
            null,
            "Demo mode: set Rawg:ApiKey in appsettings.json or via environment variable RAWG__APIKEY to fetch live data.",
            game.Genres,
            game.Platforms,
            [],
            [],
            [],
            external
        );
    }

    private async Task<IReadOnlyList<GameSummary>> LoadDemoAsync()
    {
        return await _cache.GetOrCreateAsync("demo:games", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(12);

            var path = Path.Combine(_env.ContentRootPath, "Data", "demo-games.json");
            if (!File.Exists(path))
                return (IReadOnlyList<GameSummary>)[];

            var json = await File.ReadAllTextAsync(path);
            var payload = JsonSerializer.Deserialize<List<DemoGame>>(json, JsonOptions) ?? [];
            var games = await Task.WhenAll(payload.Select(async g => new GameSummary(
                    g.Id,
                    g.Name ?? "(Unknown)",
                    g.Released,
                    g.Rating,
                    g.RatingsCount,
                    await ResolveDemoBackgroundImageAsync(g, g.BackgroundImage),
                    g.Genres ?? [],
                    g.Platforms ?? []
                )));
            return games.ToList();
        }) ?? [];
    }

    private async Task<IReadOnlyList<DemoGame>> LoadDemoRawAsync()
    {
        return await _cache.GetOrCreateAsync("demo:games:raw", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(12);

            var path = Path.Combine(_env.ContentRootPath, "Data", "demo-games.json");
            if (!File.Exists(path))
                return (IReadOnlyList<DemoGame>)[];

            var json = await File.ReadAllTextAsync(path);
            return (IReadOnlyList<DemoGame>)(JsonSerializer.Deserialize<List<DemoGame>>(json, JsonOptions) ?? []);
        }) ?? [];
    }

    private sealed class RawgListResponse<T>
    {
        public List<T>? Results { get; set; }
    }

    private sealed class RawgName
    {
        public string Name { get; set; } = "";
    }

    private sealed class RawgPlatformWrap
    {
        public RawgName? Platform { get; set; }
    }

    private sealed class RawgGameSummary
    {
        public int Id { get; set; }
        public string? Name { get; set; }
        public string? Released { get; set; }
        public double? Rating { get; set; }
        public int? RatingsCount { get; set; }

        [JsonPropertyName("background_image")]
        public string? BackgroundImage { get; set; }

        public List<RawgName>? Genres { get; set; }
        public List<RawgPlatformWrap>? Platforms { get; set; }
    }

    private sealed class RawgGameDetails
    {
        public int Id { get; set; }
        public string? Name { get; set; }
        public string? Released { get; set; }
        public double? Rating { get; set; }
        public int? RatingsCount { get; set; }
        public int? Metacritic { get; set; }
        public string? Website { get; set; }
        public string? Description { get; set; }

        [JsonPropertyName("background_image")]
        public string? BackgroundImage { get; set; }

        public List<RawgName>? Genres { get; set; }
        public List<RawgPlatformWrap>? Platforms { get; set; }
        public List<RawgName>? Developers { get; set; }
        public List<RawgName>? Publishers { get; set; }
        public List<RawgName>? Tags { get; set; }

        public List<RawgStoreLink>? Stores { get; set; }
    }

    private sealed class RawgStoreLink
    {
        public RawgName? Store { get; set; }
        public string? Url { get; set; }
    }

    private sealed class RawgScreenshot
    {
        public string? Image { get; set; }
    }

    private sealed class DemoGame
    {
        public int Id { get; set; }
        public string? Name { get; set; }
        public string? Released { get; set; }
        public double? Rating { get; set; }
        public int? RatingsCount { get; set; }
        public string? BackgroundImage { get; set; }
        public List<string>? Genres { get; set; }
        public List<string>? Platforms { get; set; }

        public string? SteamUrl { get; set; }
        public string? PsnUrl { get; set; }
        public string? XboxUrl { get; set; }
        public string? NintendoUrl { get; set; }
        public string? EpicUrl { get; set; }
        public string? GogUrl { get; set; }
    }

    private static void AddExternal(List<ExternalStoreLink> list, string store, string? url)
    {
        if (string.IsNullOrWhiteSpace(url))
            return;
        list.Add(new ExternalStoreLink(store, url.Trim()));
    }

    private async Task<string?> ResolveDemoBackgroundImageAsync(DemoGame? game, string? currentImage)
    {
        if (!string.IsNullOrWhiteSpace(currentImage))
            return currentImage;

        var appId = TryParseSteamAppId(game?.SteamUrl);
        if (appId is null)
            return null;

        return await _cache.GetOrCreateAsync($"steam:header:{appId}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(12);
            var metadata = await _steamStore.GetAppMetadataAsync(appId);
            return string.IsNullOrWhiteSpace(metadata?.HeaderImage) ? null : metadata.HeaderImage;
        });
    }

    private static string? TryParseSteamAppId(string? steamUrl)
    {
        if (string.IsNullOrWhiteSpace(steamUrl))
            return null;
        if (!Uri.TryCreate(steamUrl, UriKind.Absolute, out var uri))
            return null;

        var segments = uri.AbsolutePath.Trim('/').Split('/', StringSplitOptions.RemoveEmptyEntries);
        for (var index = 0; index < segments.Length - 1; index++)
        {
            if (!segments[index].Equals("app", StringComparison.OrdinalIgnoreCase))
                continue;
            return int.TryParse(segments[index + 1], out _) ? segments[index + 1] : null;
        }

        return null;
    }
}
```

## Б.13 Агрегатор магазинних пропозицій

Файл: `GamesInfoSys/GamesInfoSys/Services/OfferAggregator.cs`

```csharp
using GamesInfoSys.Data;
using GamesInfoSys.Data.Entities;
using GamesInfoSys.Models;
using Microsoft.EntityFrameworkCore;

namespace GamesInfoSys.Services;

public sealed class OfferAggregator
{
    private readonly AppDbContext _db;
    private readonly RegionResolver _regions;
    private readonly SteamStoreClient _steam;
    private readonly CheapSharkClient _cheapShark;
    private readonly UaMarketplaceScraper _uaScraper;

    public OfferAggregator(
        AppDbContext db,
        RegionResolver regions,
        SteamStoreClient steam,
        CheapSharkClient cheapShark,
        UaMarketplaceScraper uaScraper)
    {
        _db = db;
        _regions = regions;
        _steam = steam;
        _cheapShark = cheapShark;
        _uaScraper = uaScraper;
    }

    public async Task<IReadOnlyList<StoreOffer>> GetOffersForRawgGameAsync(int rawgGameId)
    {
        var trackedId = await _db.TrackedGames
            .Where(g => g.RawgGameId == rawgGameId)
            .Select(g => (long?)g.Id)
            .FirstOrDefaultAsync();

        if (trackedId is null)
            return [];

        return await _db.StoreOffers
            .AsNoTracking()
            .Where(o => o.TrackedGameId == trackedId.Value)
            .OrderBy(o => o.Platform)
            .ThenBy(o => o.Store)
            .ToListAsync();
    }

    public async Task SyncOffersForRawgGameAsync(int rawgGameId, string? rawgName)
    {
        await SyncOffersForRawgGameAsync(rawgGameId, rawgName, null);
    }

    public async Task SyncOffersForRawgGameAsync(int rawgGameId, string? rawgName, GameDetails? details)
    {
        var tracked = await _db.TrackedGames.FirstOrDefaultAsync(g => g.RawgGameId == rawgGameId);
        if (tracked is null)
        {
            tracked = new TrackedGame
            {
                RawgGameId = rawgGameId,
                Name = rawgName
            };
            _db.TrackedGames.Add(tracked);
            await _db.SaveChangesAsync();
        }
        else if (!string.IsNullOrWhiteSpace(rawgName) && tracked.Name != rawgName)
        {
            tracked.Name = rawgName;
            await _db.SaveChangesAsync();
        }

        if (string.IsNullOrWhiteSpace(tracked.SteamAppId) && details is not null)
        {
            var inferred = TryInferSteamAppId(details);
            if (!string.IsNullOrWhiteSpace(inferred))
            {
                tracked.SteamAppId = inferred;
                await _db.SaveChangesAsync();
            }
        }

        if (!string.IsNullOrWhiteSpace(tracked.SteamAppId))
        {
            await SyncSteamAsync(tracked);
            await SyncCheapSharkAsync(tracked);
        }

        if (!string.IsNullOrWhiteSpace(tracked.Name))
        {
            await SyncUaMarketplacesAsync(tracked, platform: "Xbox", query: $"{tracked.Name} xbox");
            await SyncUaMarketplacesAsync(tracked, platform: "Switch", query: $"{tracked.Name} nintendo switch");
        }

        // TODO: add platform ingestors here (PSN/Xbox/Nintendo/Epic/GOG/etc).
    }

    public async Task SetSteamAppIdAsync(int rawgGameId, string steamAppId)
    {
        var tracked = await _db.TrackedGames.FirstOrDefaultAsync(g => g.RawgGameId == rawgGameId);
        if (tracked is null)
        {
            tracked = new TrackedGame
            {
                RawgGameId = rawgGameId,
                SteamAppId = steamAppId
            };
            _db.TrackedGames.Add(tracked);
        }
        else
        {
            tracked.SteamAppId = steamAppId;
        }

        await _db.SaveChangesAsync();
    }

    private async Task SyncSteamAsync(TrackedGame game)
    {
        var region = _regions.ForPlatform(GamePlatform.Pc);
        var result = await _steam.GetAppPriceAsync(game.SteamAppId!, region);
        if (result is null)
            return;

        var now = DateTime.UtcNow;

        var existing = await _db.StoreOffers.FirstOrDefaultAsync(o =>
            o.Store == "Steam" &&
            o.ExternalId == game.SteamAppId &&
            o.Region == region);

        if (existing is null)
        {
            existing = new StoreOffer
            {
                TrackedGameId = game.Id,
                Store = "Steam",
                Platform = "PC",
                Region = region,
                ExternalId = game.SteamAppId!,
                Title = result.Name,
                Url = $"https://store.steampowered.com/app/{game.SteamAppId}/",
                Currency = result.Currency,
                PriceMinor = result.FinalMinor,
                OriginalPriceMinor = result.InitialMinor,
                LastSeenUtc = now,
                LastUpdatedUtc = now
            };
            _db.StoreOffers.Add(existing);
            await _db.SaveChangesAsync();
        }
        else
        {
            existing.Title = result.Name;
            existing.Url = $"https://store.steampowered.com/app/{game.SteamAppId}/";
            existing.Currency = result.Currency;
            existing.PriceMinor = result.FinalMinor;
            existing.OriginalPriceMinor = result.InitialMinor;
            existing.LastSeenUtc = now;
            existing.LastUpdatedUtc = now;
            await _db.SaveChangesAsync();
        }

        if (!string.IsNullOrWhiteSpace(result.Currency) && result.FinalMinor > 0)
        {
            _db.OfferPricePoints.Add(new OfferPricePoint
            {
                StoreOfferId = existing.Id,
                AtUtc = now,
                Currency = result.Currency,
                PriceMinor = result.FinalMinor,
                OriginalPriceMinor = result.InitialMinor
            });
            await _db.SaveChangesAsync();
        }
    }

    private async Task SyncCheapSharkAsync(TrackedGame game)
    {
        var deals = await _cheapShark.GetDealsBySteamAppIdAsync(game.SteamAppId!, pageSize: 20, onSaleOnly: false);
        if (deals.Count == 0)
            return;

        var now = DateTime.UtcNow;
        var region = "GLOBAL";

        foreach (var d in deals)
        {
            var externalId = d.DealId;
            var existing = await _db.StoreOffers.FirstOrDefaultAsync(o =>
                o.Store == "CheapShark" &&
                o.ExternalId == externalId &&
                o.Region == region);

            var priceMinor = d.SalePriceUsd is null ? (long?)null : (long)Math.Round(d.SalePriceUsd.Value * 100m, MidpointRounding.AwayFromZero);
            var originalMinor = d.NormalPriceUsd is null ? (long?)null : (long)Math.Round(d.NormalPriceUsd.Value * 100m, MidpointRounding.AwayFromZero);

            var title = string.IsNullOrWhiteSpace(d.Title) ? (game.Name ?? "Deal") : d.Title;
            var url = CheapSharkClient.RedirectUrl(d.DealId);

            if (existing is null)
            {
                existing = new StoreOffer
                {
                    TrackedGameId = game.Id,
                    Store = "CheapShark",
                    Platform = "PC",
                    Region = region,
                    ExternalId = externalId,
                    Title = title,
                    Url = url,
                    Currency = "USD",
                    PriceMinor = priceMinor,
                    OriginalPriceMinor = originalMinor,
                    LastSeenUtc = now,
                    LastUpdatedUtc = now
                };
                _db.StoreOffers.Add(existing);
            }
            else
            {
                existing.Title = title;
                existing.Url = url;
                existing.Currency = "USD";
                existing.PriceMinor = priceMinor;
                existing.OriginalPriceMinor = originalMinor;
                existing.LastSeenUtc = now;
                existing.LastUpdatedUtc = now;
            }
        }

        await _db.SaveChangesAsync();
    }

    private async Task SyncUaMarketplacesAsync(TrackedGame game, string platform, string query)
    {
        if (!_uaScraper.Enabled)
            return;

        var offers = new List<UaMarketOffer>();
        offers.AddRange(await _uaScraper.SearchAsync("prom", platform, query));
        offers.AddRange(await _uaScraper.SearchAsync("olx", platform, query));

        var now = DateTime.UtcNow;
        foreach (var o in offers)
        {
            var externalId = $"{o.Store}:{o.Url}".ToLowerInvariant();
            var existing = await _db.StoreOffers.FirstOrDefaultAsync(x =>
                x.Store == o.Store &&
                x.ExternalId == externalId &&
                x.Region == "UA");

            if (existing is null)
            {
                existing = new StoreOffer
                {
                    TrackedGameId = game.Id,
                    Store = o.Store,
                    Platform = platform,
                    Region = "UA",
                    ExternalId = externalId,
                    Title = o.Title,
                    Url = o.Url,
                    Currency = o.Currency,
                    PriceMinor = o.PriceMinor,
                    OriginalPriceMinor = null,
                    LastSeenUtc = now,
                    LastUpdatedUtc = now
                };
                _db.StoreOffers.Add(existing);
            }
            else
            {
                existing.Title = o.Title;
                existing.Url = o.Url;
                existing.Currency = o.Currency;
                existing.PriceMinor = o.PriceMinor;
                existing.LastSeenUtc = now;
                existing.LastUpdatedUtc = now;
            }
        }

        await _db.SaveChangesAsync();
    }

    private static string? TryInferSteamAppId(GameDetails details)
    {
        // RAWG often provides store links like https://store.steampowered.com/app/1245620/...
        foreach (var link in details.ExternalStores)
        {
            if (!link.Url.Contains("store.steampowered.com", StringComparison.OrdinalIgnoreCase))
                continue;

            var uriOk = Uri.TryCreate(link.Url, UriKind.Absolute, out var uri);
            if (!uriOk || uri is null)
                continue;

            var segments = uri.AbsolutePath.Trim('/').Split('/', StringSplitOptions.RemoveEmptyEntries);
            // Expect: app/{appid}/...
            for (var i = 0; i < segments.Length - 1; i++)
            {
                if (!segments[i].Equals("app", StringComparison.OrdinalIgnoreCase))
                    continue;
                var id = segments[i + 1];
                if (int.TryParse(id, out _))
                    return id;
            }
        }

        return null;
    }
}
```

## Б.14 Клієнт Steam Store

Файл: `GamesInfoSys/GamesInfoSys/Services/SteamStoreClient.cs`

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace GamesInfoSys.Services;

public sealed class SteamStoreClient
{
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNameCaseInsensitive = true
    };

    private readonly HttpClient _http;

    public SteamStoreClient(HttpClient http)
    {
        _http = http;
    }

    public async Task<SteamPriceResult?> GetAppPriceAsync(string appId, string countryCode)
    {
        var metadata = await GetAppMetadataAsync(appId, countryCode);
        if (metadata?.Price is null)
            return null;

        return new SteamPriceResult(
            Name: metadata.Name ?? $"Steam app {appId}",
            Currency: metadata.Price.Currency ?? "",
            FinalMinor: metadata.Price.Final,
            InitialMinor: metadata.Price.Initial,
            DiscountPercent: metadata.Price.DiscountPercent
        );
    }

    public async Task<SteamAppMetadata?> GetAppMetadataAsync(string appId, string countryCode = "UA")
    {
        if (string.IsNullOrWhiteSpace(appId))
            return null;
        if (!int.TryParse(appId, out _))
            return null;

        countryCode = string.IsNullOrWhiteSpace(countryCode) ? "UA" : countryCode.Trim().ToLowerInvariant();

        var url = $"api/appdetails?appids={Uri.EscapeDataString(appId)}&cc={Uri.EscapeDataString(countryCode)}&l=english";
        using var res = await _http.GetAsync(url);
        res.EnsureSuccessStatusCode();

        var json = await res.Content.ReadAsStringAsync();

        var dict = JsonSerializer.Deserialize<Dictionary<string, SteamAppDetailsEnvelope>>(json, JsonOptions);
        if (dict is null || !dict.TryGetValue(appId, out var envelope))
            return null;
        if (!envelope.Success || envelope.Data is null)
            return null;

        return new SteamAppMetadata(
            envelope.Data.Name,
            envelope.Data.HeaderImage,
            envelope.Data.PriceOverview is null
                ? null
                : new SteamPriceSnapshot(
                    envelope.Data.PriceOverview.Currency ?? "",
                    envelope.Data.PriceOverview.Final,
                    envelope.Data.PriceOverview.Initial,
                    envelope.Data.PriceOverview.DiscountPercent)
        );
    }

    private sealed class SteamAppDetailsEnvelope
    {
        public bool Success { get; set; }
        public SteamAppData? Data { get; set; }
    }

    private sealed class SteamAppData
    {
        public string? Name { get; set; }

        [JsonPropertyName("header_image")]
        public string? HeaderImage { get; set; }

        [JsonPropertyName("price_overview")]
        public SteamPriceOverview? PriceOverview { get; set; }
    }

    private sealed class SteamPriceOverview
    {
        public string? Currency { get; set; }
        public long Final { get; set; }
        public long Initial { get; set; }

        [JsonPropertyName("discount_percent")]
        public int DiscountPercent { get; set; }
    }
}

public sealed record SteamAppMetadata(
    string? Name,
    string? HeaderImage,
    SteamPriceSnapshot? Price
);

public sealed record SteamPriceSnapshot(
    string Currency,
    long Final,
    long Initial,
    int DiscountPercent
);

public sealed record SteamPriceResult(
    string Name,
    string Currency,
    long FinalMinor,
    long InitialMinor,
    int DiscountPercent
);
```

## Б.15 Клієнт CheapShark

Файл: `GamesInfoSys/GamesInfoSys/Services/CheapSharkClient.cs`

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;
using Microsoft.Extensions.Caching.Memory;

namespace GamesInfoSys.Services;

public sealed class CheapSharkClient
{
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNameCaseInsensitive = true
    };

    private readonly HttpClient _http;
    private readonly IMemoryCache _cache;

    public CheapSharkClient(HttpClient http, IMemoryCache cache)
    {
        _http = http;
        _cache = cache;
    }

    public async Task<IReadOnlyList<CheapSharkDeal>> GetDealsBySteamAppIdAsync(string steamAppId, int pageSize = 20, bool onSaleOnly = false)
    {
        if (string.IsNullOrWhiteSpace(steamAppId))
            return [];
        if (!int.TryParse(steamAppId, out _))
            return [];

        pageSize = Math.Clamp(pageSize, 1, 60);

        var cacheKey = $"cheapshark:deals:steam={steamAppId}:ps={pageSize}:sale={onSaleOnly}";
        var cached = await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);

            var url = $"api/1.0/deals?steamAppID={Uri.EscapeDataString(steamAppId)}&pageNumber=0&pageSize={pageSize}&sortBy=Price&desc=0";
            if (onSaleOnly)
                url += "&onSale=1";

            using var res = await _http.GetAsync(url);
            res.EnsureSuccessStatusCode();

            await using var stream = await res.Content.ReadAsStreamAsync();
            return await JsonSerializer.DeserializeAsync<List<DealDto>>(stream, JsonOptions) ?? [];
        });

        return (cached ?? [])
            .Where(d => !string.IsNullOrWhiteSpace(d.DealId))
            .Select(d => new CheapSharkDeal(
                DealId: d.DealId!,
                StoreId: d.StoreId ?? "",
                Title: d.Title ?? "",
                SalePriceUsd: ParseMoney(d.SalePrice),
                NormalPriceUsd: ParseMoney(d.NormalPrice),
                SavingsPercent: ParseMoney(d.Savings),
                SteamAppId: d.SteamAppId ?? steamAppId
            ))
            .ToList();
    }

    public static string RedirectUrl(string dealId)
    {
        return $"https://www.cheapshark.com/redirect?dealID={Uri.EscapeDataString(dealId)}";
    }

    private static decimal? ParseMoney(string? s)
    {
        if (string.IsNullOrWhiteSpace(s))
            return null;
        if (decimal.TryParse(s, System.Globalization.NumberStyles.Any, System.Globalization.CultureInfo.InvariantCulture, out var x))
            return x;
        return null;
    }

    private sealed class DealDto
    {
        [JsonPropertyName("dealID")]
        public string? DealId { get; set; }

        public string? StoreId { get; set; }
        public string? Title { get; set; }

        public string? SalePrice { get; set; }
        public string? NormalPrice { get; set; }
        public string? Savings { get; set; }

        public string? SteamAppId { get; set; }
    }
}

public sealed record CheapSharkDeal(
    string DealId,
    string StoreId,
    string Title,
    decimal? SalePriceUsd,
    decimal? NormalPriceUsd,
    decimal? SavingsPercent,
    string SteamAppId
);
```

## Б.16 Оброблення українських маркетплейсів

Файл: `GamesInfoSys/GamesInfoSys/Services/UaMarketplaceScraper.cs`

```csharp
using System.Text.Json;
using System.Text.RegularExpressions;
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.Options;

namespace GamesInfoSys.Services;

public sealed record UaMarketOffer(
    string Store,
    string Platform,
    string Title,
    string Url,
    string Currency,
    long PriceMinor
);

public sealed class UaMarketplaceScraper
{
    private readonly HttpClient _http;
    private readonly IMemoryCache _cache;
    private readonly ScrapingOptions _options;

    public UaMarketplaceScraper(HttpClient http, IMemoryCache cache, IOptions<ScrapingOptions> options)
    {
        _http = http;
        _cache = cache;
        _options = options.Value;
    }

    public bool Enabled => _options.Enabled;

    public async Task<IReadOnlyList<UaMarketOffer>> SearchAsync(string store, string platform, string query)
    {
        if (!Enabled)
            return [];
        if (string.IsNullOrWhiteSpace(store) || string.IsNullOrWhiteSpace(platform) || string.IsNullOrWhiteSpace(query))
            return [];

        store = store.Trim().ToLowerInvariant();
        platform = platform.Trim();
        query = query.Trim();

        var cacheKey = $"ua:scrape:{store}:{platform}:{query}";
        var minutes = Math.Clamp(_options.CacheMinutes, 1, 24 * 60);

        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(minutes);

            return store switch
            {
                "prom" => await SearchPromAsync(platform, query),
                "olx" => await SearchOlxAsync(platform, query),
                // Rozetka search is often Cloudflare-protected; keep disabled unless you add a real browser worker.
                _ => []
            };
        }) ?? [];
    }

    private async Task<IReadOnlyList<UaMarketOffer>> SearchPromAsync(string platform, string query)
    {
        var url = $"https://prom.ua/ua/search?search_term={Uri.EscapeDataString(query)}";
        var html = await _http.GetStringAsync(url);

        var stateJson = ExtractJsonAssignment(html, "window.__STATE__");
        if (stateJson is null)
            return [];

        using var doc = JsonDocument.Parse(stateJson);
        var found = new List<UaMarketOffer>();
        ExtractOffersFromUnknownJson(
            root: doc.RootElement,
            acceptUrl: u => u.Contains("prom.ua", StringComparison.OrdinalIgnoreCase),
            normalizeUrl: u => u.StartsWith("http", StringComparison.OrdinalIgnoreCase) ? u : $"https://prom.ua{u}",
            storeName: "Prom.ua",
            platform: platform,
            found: found
        );

        return found
            .GroupBy(o => o.Url, StringComparer.OrdinalIgnoreCase)
            .Select(g => g.First())
            .OrderBy(o => o.PriceMinor)
            .Take(Math.Clamp(_options.MaxResultsPerStore, 1, 30))
            .ToList();
    }

    private async Task<IReadOnlyList<UaMarketOffer>> SearchOlxAsync(string platform, string query)
    {
        var slug = SlugForOlx(query);
        var url = $"https://www.olx.ua/uk/list/q-{slug}/";
        var html = await _http.GetStringAsync(url);

        var stateJson = ExtractJsonAssignment(html, "window.__PRERENDERED_STATE__");
        if (stateJson is null)
            return [];

        using var doc = JsonDocument.Parse(stateJson);
        var found = new List<UaMarketOffer>();
        ExtractOffersFromUnknownJson(
            root: doc.RootElement,
            acceptUrl: u => u.Contains("olx.ua", StringComparison.OrdinalIgnoreCase),
            normalizeUrl: u =>
            {
                if (u.StartsWith("http", StringComparison.OrdinalIgnoreCase))
                    return u;
                if (!u.StartsWith("/"))
                    u = "/" + u;
                return $"https://www.olx.ua{u}";
            },
            storeName: "OLX",
            platform: platform,
            found: found
        );

        return found
            .GroupBy(o => o.Url, StringComparer.OrdinalIgnoreCase)
            .Select(g => g.First())
            .OrderBy(o => o.PriceMinor)
            .Take(Math.Clamp(_options.MaxResultsPerStore, 1, 30))
            .ToList();
    }

    private static string? ExtractJsonAssignment(string html, string varName)
    {
        // Matches: window.__STATE__ = {...};
        var pattern = Regex.Escape(varName) + @"\s*=\s*";
        var start = Regex.Match(html, pattern, RegexOptions.IgnoreCase);
        if (!start.Success)
            return null;

        var idx = start.Index + start.Length;
        // Find JSON object/array start
        while (idx < html.Length && char.IsWhiteSpace(html[idx])) idx++;
        if (idx >= html.Length)
            return null;

        var open = html[idx];
        if (open != '{' && open != '[')
            return null;

        var endIdx = FindJsonEnd(html, idx);
        if (endIdx < 0)
            return null;

        return html.Substring(idx, endIdx - idx + 1);
    }

    private static int FindJsonEnd(string s, int startIdx)
    {
        var depth = 0;
        var inString = false;
        var escape = false;

        for (var i = startIdx; i < s.Length; i++)
        {
            var c = s[i];
            if (inString)
            {
                if (escape)
                {
                    escape = false;
                    continue;
                }
                if (c == '\\')
                {
                    escape = true;
                    continue;
                }
                if (c == '"')
                    inString = false;
                continue;
            }

            if (c == '"')
            {
                inString = true;
                continue;
            }

            if (c == '{' || c == '[') depth++;
            if (c == '}' || c == ']') depth--;

            if (depth == 0)
                return i;
        }

        return -1;
    }

    private static void ExtractOffersFromUnknownJson(
        JsonElement root,
        Func<string, bool> acceptUrl,
        Func<string, string> normalizeUrl,
        string storeName,
        string platform,
        List<UaMarketOffer> found)
    {
        var stack = new Stack<JsonElement>();
        stack.Push(root);

        while (stack.Count > 0 && found.Count < 2000)
        {
            var el = stack.Pop();

            if (el.ValueKind == JsonValueKind.Object)
            {
                string? title = null;
                string? url = null;
                long? priceMinor = null;

                foreach (var prop in el.EnumerateObject())
                {
                    if (prop.Value.ValueKind is JsonValueKind.Object or JsonValueKind.Array)
                        stack.Push(prop.Value);

                    if (title is null && prop.NameEquals("name") && prop.Value.ValueKind == JsonValueKind.String)
                        title = prop.Value.GetString();
                    if (title is null && prop.NameEquals("title") && prop.Value.ValueKind == JsonValueKind.String)
                        title = prop.Value.GetString();

                    if (url is null && (prop.NameEquals("url") || prop.NameEquals("href")) && prop.Value.ValueKind == JsonValueKind.String)
                        url = prop.Value.GetString();

                    if (priceMinor is null && prop.NameEquals("price"))
                    {
                        priceMinor = TryParsePriceMinor(prop.Value);
                    }
                    if (priceMinor is null && prop.NameEquals("price_uah"))
                    {
                        priceMinor = TryParsePriceMinor(prop.Value);
                    }
                }

                if (!string.IsNullOrWhiteSpace(title) &&
                    !string.IsNullOrWhiteSpace(url) &&
                    acceptUrl(url!) &&
                    priceMinor is not null &&
                    priceMinor.Value > 0)
                {
                    found.Add(new UaMarketOffer(
                        Store: storeName,
                        Platform: platform,
                        Title: title!.Trim(),
                        Url: normalizeUrl(url!),
                        Currency: "UAH",
                        PriceMinor: priceMinor.Value
                    ));
                }
            }
            else if (el.ValueKind == JsonValueKind.Array)
            {
                foreach (var x in el.EnumerateArray())
                    stack.Push(x);
            }
        }
    }

    private static long? TryParsePriceMinor(JsonElement el)
    {
        // Accept: number (major), string like "1 999 ₴", object containing amount/value
        switch (el.ValueKind)
        {
            case JsonValueKind.Number:
                if (el.TryGetDecimal(out var dec))
                    return (long)Math.Round(dec * 100m, MidpointRounding.AwayFromZero);
                return null;
            case JsonValueKind.String:
                return ParseUahStringToMinor(el.GetString());
            case JsonValueKind.Object:
                {
                    if (el.TryGetProperty("value", out var v) || el.TryGetProperty("amount", out v))
                    {
                        if (v.ValueKind == JsonValueKind.Number && v.TryGetDecimal(out var dec2))
                            return (long)Math.Round(dec2 * 100m, MidpointRounding.AwayFromZero);
                        if (v.ValueKind == JsonValueKind.String)
                            return ParseUahStringToMinor(v.GetString());
                    }
                    if (el.TryGetProperty("uah", out var u) && u.ValueKind == JsonValueKind.String)
                        return ParseUahStringToMinor(u.GetString());
                    return null;
                }
            default:
                return null;
        }
    }

    private static long? ParseUahStringToMinor(string? s)
    {
        if (string.IsNullOrWhiteSpace(s))
            return null;

        s = s.Replace("\u00A0", " ");
        // Extract digits and optional decimal part.
        var m = Regex.Match(s, @"(\d[\d\s]*)([.,](\d{1,2}))?");
        if (!m.Success)
            return null;

        var whole = m.Groups[1].Value.Replace(" ", "");
        var frac = m.Groups[3].Success ? m.Groups[3].Value : "0";
        if (!long.TryParse(whole, out var hryvnia))
            return null;

        var kop = 0L;
        if (frac.Length == 1) frac += "0";
        if (frac.Length >= 2 && long.TryParse(frac.Substring(0, 2), out var kopParsed))
            kop = kopParsed;

        return hryvnia * 100 + kop;
    }

    private static string SlugForOlx(string query)
    {
        query = query.Trim().ToLowerInvariant();
        query = query.Replace('’', '\'').Replace('–', '-').Replace('—', '-');
        query = Regex.Replace(query, "\\s+", "-");
        query = Regex.Replace(query, "[^a-z0-9\\-]+", "");
        return query;
    }
}
```

## Б.17 Клієнт валютних курсів НБУ

Файл: `GamesInfoSys/GamesInfoSys/Services/NbuRatesClient.cs`

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.Options;

namespace GamesInfoSys.Services;

public sealed class NbuRatesClient
{
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNameCaseInsensitive = true
    };

    private readonly HttpClient _http;
    private readonly IMemoryCache _cache;

    public NbuRatesClient(HttpClient http, IMemoryCache cache)
    {
        _http = http;
        _cache = cache;
    }

    public Task<Dictionary<string, decimal>> GetRatesToUahAsync()
    {
        // Map: ISO currency -> UAH per 1 unit of currency.
        return GetRatesToUahCoreAsync();
    }

    private async Task<Dictionary<string, decimal>> GetRatesToUahCoreAsync()
    {
        var rates = await _cache.GetOrCreateAsync("rates:nbu:to_uah", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(6);

            // NBU endpoint (JSON): /NBUStatService/v1/statdirectory/exchange?json
            using var res = await _http.GetAsync("NBUStatService/v1/statdirectory/exchange?json");
            res.EnsureSuccessStatusCode();

            await using var stream = await res.Content.ReadAsStreamAsync();
            var list = await JsonSerializer.DeserializeAsync<List<NbuRate>>(stream, JsonOptions) ?? [];

            var dict = new Dictionary<string, decimal>(StringComparer.OrdinalIgnoreCase)
            {
                ["UAH"] = 1m
            };

            foreach (var r in list)
            {
                if (string.IsNullOrWhiteSpace(r.Cc))
                    continue;
                if (r.Rate <= 0)
                    continue;
                dict[r.Cc] = r.Rate;
            }

            return dict;
        });

        return rates ?? new Dictionary<string, decimal>(StringComparer.OrdinalIgnoreCase) { ["UAH"] = 1m };
    }

    private sealed class NbuRate
    {
        [JsonPropertyName("cc")]
        public string? Cc { get; set; }

        [JsonPropertyName("rate")]
        public decimal Rate { get; set; }
    }
}
```

## Б.18 Конвертація цін

Файл: `GamesInfoSys/GamesInfoSys/Services/CurrencyConverter.cs`

```csharp
namespace GamesInfoSys.Services;

public sealed class CurrencyConverter
{
    private readonly NbuRatesClient _nbu;

    public CurrencyConverter(NbuRatesClient nbu)
    {
        _nbu = nbu;
    }

    public async Task<decimal?> ToUahAsync(decimal amount, string fromCurrency)
    {
        if (amount <= 0)
            return null;
        if (string.IsNullOrWhiteSpace(fromCurrency))
            return null;

        fromCurrency = fromCurrency.Trim().ToUpperInvariant();
        if (fromCurrency == "UAH")
            return amount;

        var rates = await _nbu.GetRatesToUahAsync();
        if (!rates.TryGetValue(fromCurrency, out var rateToUah) || rateToUah <= 0)
            return null;

        return amount * rateToUah;
    }
}
```

## Б.19 Визначення регіону

Файл: `GamesInfoSys/GamesInfoSys/Services/RegionResolver.cs`

```csharp
using Microsoft.Extensions.Options;

namespace GamesInfoSys.Services;

public enum GamePlatform
{
    Pc,
    PlayStation,
    Xbox,
    Switch,
    Mobile
}

public sealed class RegionResolver
{
    private readonly PricingOptions _options;

    public RegionResolver(IOptions<PricingOptions> options)
    {
        _options = options.Value;
    }

    public string DefaultRegion => NormalizeRegion(_options.DefaultRegion) ?? "UA";

    public string ForPlatform(GamePlatform platform)
    {
        var key = platform.ToString();
        if (_options.PlatformRegions.TryGetValue(key, out var region))
            return NormalizeRegion(region) ?? DefaultRegion;

        return DefaultRegion;
    }

    private static string? NormalizeRegion(string? region)
    {
        if (string.IsNullOrWhiteSpace(region))
            return null;
        return region.Trim().ToUpperInvariant();
    }
}
```

## Б.20 Локалізація інтерфейсу

Файл: `GamesInfoSys/GamesInfoSys/Services/UiText.cs`

```csharp
using System.Globalization;

namespace GamesInfoSys.Services;

public sealed class UiText
{
    private static readonly IReadOnlyDictionary<string, string> English = new Dictionary<string, string>(StringComparer.Ordinal)
    {
        ["SiteName"] = "GamesInfoSys",
        ["NavHome"] = "Home",
        ["NavBrowse"] = "Browse",
        ["NavPrivacy"] = "Privacy",
        ["Language"] = "Language",
        ["LangEnglish"] = "English",
        ["LangUkrainian"] = "Ukrainian",
        ["FooterPrivacy"] = "Privacy",
        ["HomeTitle"] = "Home",
        ["HeroTitle"] = "Game Info Aggregator",
        ["HeroText"] = "Search and browse games, open details, and keep everything in one place.",
        ["BrowseGames"] = "Browse games",
        ["FastSearch"] = "Fast search",
        ["FastSearchText"] = "Type a title and jump straight to a game page.",
        ["DemoMode"] = "Demo mode",
        ["DemoModeText"] = "Works without API keys; add a RAWG key later for live data.",
        ["PrivacyTitle"] = "Privacy Policy",
        ["PrivacyLead"] = "This site does not require accounts and does not intentionally collect personal data.",
        ["PrivacyItem1"] = "When configured with a RAWG API key, the server requests game metadata from RAWG and shows it in the UI.",
        ["PrivacyItem2"] = "In demo mode (no API key), the site uses a small local dataset.",
        ["PrivacyItem3"] = "Links to external sites (official game websites, images) may have their own privacy practices.",
        ["BrowseTitle"] = "Browse",
        ["BrowseHeading"] = "Browse games",
        ["Live"] = "Live",
        ["DataViaRawg"] = "Data via RAWG.",
        ["SetRawgApiKey"] = "Set Rawg:ApiKey to fetch live data.",
        ["SearchPlaceholder"] = "Search (e.g. Elden Ring)",
        ["Search"] = "Search",
        ["TipTopGames"] = "Tip: leave empty for top games.",
        ["Clear"] = "Clear",
        ["NoGamesFound"] = "No games found.",
        ["CoverAlt"] = "cover",
        ["GameTitleFallback"] = "Game",
        ["GameNotFound"] = "Game not found.",
        ["Back"] = "Back",
        ["Released"] = "Released",
        ["Metacritic"] = "Metacritic",
        ["Rating"] = "Rating",
        ["OfficialWebsite"] = "Official website",
        ["BackToBrowse"] = "Back to browse",
        ["PricesHeading"] = "Prices (Region: UA, Switch: ZA)",
        ["PricesSubheading"] = "Official stores and marketplaces. Add identifiers to enable more platforms.",
        ["BrowseButton"] = "Browse",
        ["DemoModeNote"] = "Demo mode note: store buttons below come from the demo JSON (not live platform APIs).",
        ["OpenStore"] = "Open {0}",
        ["BestMarketplaceDeal"] = "Best marketplace deal:",
        ["Open"] = "Open",
        ["NoOffers"] = "No offers yet. If RAWG provides a Steam link it will be detected automatically; otherwise add a Steam App ID below.",
        ["Platform"] = "Platform",
        ["Store"] = "Store",
        ["Region"] = "Region",
        ["Price"] = "Price",
        ["Marketplace"] = "Marketplace",
        ["OpenDeal"] = "Open deal",
        ["SteamMappingOptional"] = "Steam mapping (optional)",
        ["AddOrChange"] = "Add / change",
        ["SteamLinkOrAppId"] = "Steam link or App ID",
        ["SteamInputPlaceholder"] = "https://store.steampowered.com/app/1245620/  or  1245620",
        ["SteamTip"] = "Tip: open Steam in browser and copy the URL.",
        ["SteamValidation"] = "Paste a Steam app link or an App ID.",
        ["SaveRefresh"] = "Save & refresh",
        ["About"] = "About",
        ["NoDescription"] = "No description available.",
        ["FindInUkraine"] = "Find in Ukraine",
        ["FindInUkraineText"] = "Quick searches for physical copies or gift cards. Prices may differ by edition/platform.",
        ["CheapSharkDeals"] = "CheapShark deals",
        ["PsStoreOfficial"] = "PS Store (official)",
        ["XboxStoreOfficial"] = "Xbox Store (official)",
        ["NintendoStoreOfficial"] = "Nintendo eShop ZA (official)",
        ["YouTubeReview"] = "YouTube review",
        ["YouTubeGameplay"] = "YouTube gameplay",
        ["PlatformSearches"] = "Platform searches",
        ["PromPs"] = "Prom (PS)",
        ["OlxPs"] = "OLX (PS)",
        ["RozetkaPs"] = "Rozetka (PS)",
        ["PromXbox"] = "Prom (Xbox)",
        ["OlxXbox"] = "OLX (Xbox)",
        ["RozetkaXbox"] = "Rozetka (Xbox)",
        ["PromSwitch"] = "Prom (Switch)",
        ["OlxSwitch"] = "OLX (Switch)",
        ["RozetkaSwitch"] = "Rozetka (Switch)",
        ["Genres"] = "Genres",
        ["Platforms"] = "Platforms",
        ["Screenshots"] = "Screenshots",
        ["ScreenshotAlt"] = "Screenshot",
        ["ErrorTitle"] = "Error",
        ["ErrorHeading"] = "Error.",
        ["ErrorText"] = "An error occurred while processing your request.",
        ["RequestId"] = "Request ID:",
        ["DevelopmentMode"] = "Development Mode",
        ["DevelopmentModeText1"] = "Swapping to the Development environment displays detailed information about the error that occurred.",
        ["DevelopmentModeText2"] = "The Development environment shouldn't be enabled for deployed applications.",
        ["DevelopmentModeText3"] = "It can result in displaying sensitive information from exceptions to end users.",
        ["DevelopmentModeText4"] = "For local debugging, enable the Development environment by setting the ASPNETCORE_ENVIRONMENT environment variable to Development and restarting the app."
    };

    private static readonly IReadOnlyDictionary<string, string> Ukrainian = new Dictionary<string, string>(StringComparer.Ordinal)
    {
        ["SiteName"] = "GamesInfoSys",
        ["NavHome"] = "Головна",
        ["NavBrowse"] = "Каталог",
        ["NavPrivacy"] = "Конфіденційність",
        ["Language"] = "Мова",
        ["LangEnglish"] = "Англійська",
        ["LangUkrainian"] = "Українська",
        ["FooterPrivacy"] = "Конфіденційність",
        ["HomeTitle"] = "Головна",
        ["HeroTitle"] = "Агрегатор ігор",
        ["HeroText"] = "Шукайте ігри, переглядайте каталог, відкривайте деталі та тримайте все в одному місці.",
        ["BrowseGames"] = "Переглянути ігри",
        ["FastSearch"] = "Швидкий пошук",
        ["FastSearchText"] = "Введіть назву і швидко перейдіть на сторінку гри.",
        ["DemoMode"] = "Демо-режим",
        ["DemoModeText"] = "Працює без API-ключів; для живих даних можна пізніше додати ключ RAWG.",
        ["PrivacyTitle"] = "Політика конфіденційності",
        ["PrivacyLead"] = "Сайт не вимагає облікових записів і навмисно не збирає персональні дані.",
        ["PrivacyItem1"] = "Коли налаштовано ключ RAWG API, сервер запитує метадані ігор у RAWG і показує їх в інтерфейсі.",
        ["PrivacyItem2"] = "У демо-режимі (без API-ключа) сайт використовує невеликий локальний набір даних.",
        ["PrivacyItem3"] = "Посилання на зовнішні сайти (офіційні сайти ігор, зображення) можуть мати власні правила конфіденційності.",
        ["BrowseTitle"] = "Каталог",
        ["BrowseHeading"] = "Перегляд ігор",
        ["Live"] = "Наживо",
        ["DataViaRawg"] = "Дані через RAWG.",
        ["SetRawgApiKey"] = "Вкажіть Rawg:ApiKey, щоб отримувати живі дані.",
        ["SearchPlaceholder"] = "Пошук (наприклад, Elden Ring)",
        ["Search"] = "Пошук",
        ["TipTopGames"] = "Порада: залиште поле порожнім, щоб побачити топ ігор.",
        ["Clear"] = "Очистити",
        ["NoGamesFound"] = "Ігор не знайдено.",
        ["CoverAlt"] = "обкладинка",
        ["GameTitleFallback"] = "Гра",
        ["GameNotFound"] = "Гру не знайдено.",
        ["Back"] = "Назад",
        ["Released"] = "Реліз",
        ["Metacritic"] = "Metacritic",
        ["Rating"] = "Рейтинг",
        ["OfficialWebsite"] = "Офіційний сайт",
        ["BackToBrowse"] = "Назад до каталогу",
        ["PricesHeading"] = "Ціни (Регіон: UA, Switch: ZA)",
        ["PricesSubheading"] = "Офіційні магазини та маркетплейси. Додайте ідентифікатори, щоб увімкнути більше платформ.",
        ["BrowseButton"] = "Каталог",
        ["DemoModeNote"] = "Примітка демо-режиму: кнопки магазинів нижче беруться з demo JSON, а не з живих API платформ.",
        ["OpenStore"] = "Відкрити {0}",
        ["BestMarketplaceDeal"] = "Найкраща пропозиція маркетплейсу:",
        ["Open"] = "Відкрити",
        ["NoOffers"] = "Пропозицій поки немає. Якщо RAWG надає Steam-посилання, воно буде визначене автоматично; інакше додайте Steam App ID нижче.",
        ["Platform"] = "Платформа",
        ["Store"] = "Магазин",
        ["Region"] = "Регіон",
        ["Price"] = "Ціна",
        ["Marketplace"] = "Маркетплейс",
        ["OpenDeal"] = "Відкрити пропозицію",
        ["SteamMappingOptional"] = "Steam-зв'язка (необов'язково)",
        ["AddOrChange"] = "Додати / змінити",
        ["SteamLinkOrAppId"] = "Steam-посилання або App ID",
        ["SteamInputPlaceholder"] = "https://store.steampowered.com/app/1245620/  або  1245620",
        ["SteamTip"] = "Порада: відкрийте Steam у браузері та скопіюйте URL.",
        ["SteamValidation"] = "Вставте Steam-посилання або App ID.",
        ["SaveRefresh"] = "Зберегти й оновити",
        ["About"] = "Про гру",
        ["NoDescription"] = "Опис недоступний.",
        ["FindInUkraine"] = "Пошук в Україні",
        ["FindInUkraineText"] = "Швидкі пошуки фізичних копій або подарункових карток. Ціни можуть відрізнятися залежно від видання та платформи.",
        ["CheapSharkDeals"] = "Пропозиції CheapShark",
        ["PsStoreOfficial"] = "PS Store (офіційно)",
        ["XboxStoreOfficial"] = "Xbox Store (офіційно)",
        ["NintendoStoreOfficial"] = "Nintendo eShop ZA (офіційно)",
        ["YouTubeReview"] = "Огляд на YouTube",
        ["YouTubeGameplay"] = "Геймплей на YouTube",
        ["PlatformSearches"] = "Пошук за платформами",
        ["PromPs"] = "Prom (PS)",
        ["OlxPs"] = "OLX (PS)",
        ["RozetkaPs"] = "Rozetka (PS)",
        ["PromXbox"] = "Prom (Xbox)",
        ["OlxXbox"] = "OLX (Xbox)",
        ["RozetkaXbox"] = "Rozetka (Xbox)",
        ["PromSwitch"] = "Prom (Switch)",
        ["OlxSwitch"] = "OLX (Switch)",
        ["RozetkaSwitch"] = "Rozetka (Switch)",
        ["Genres"] = "Жанри",
        ["Platforms"] = "Платформи",
        ["Screenshots"] = "Скріншоти",
        ["ScreenshotAlt"] = "Скріншот",
        ["ErrorTitle"] = "Помилка",
        ["ErrorHeading"] = "Помилка.",
        ["ErrorText"] = "Під час обробки вашого запиту сталася помилка.",
        ["RequestId"] = "ID запиту:",
        ["DevelopmentMode"] = "Режим розробки",
        ["DevelopmentModeText1"] = "Перехід у середовище Development показує детальну інформацію про помилку.",
        ["DevelopmentModeText2"] = "Середовище Development не слід вмикати на розгорнутих застосунках.",
        ["DevelopmentModeText3"] = "Це може призвести до показу чутливої інформації з винятків кінцевим користувачам.",
        ["DevelopmentModeText4"] = "Для локального налагодження увімкніть середовище Development, встановивши змінну ASPNETCORE_ENVIRONMENT у значення Development, а потім перезапустіть застосунок."
    };

    public string this[string key] => Get(key);

    public string Get(string key)
    {
        var texts = IsUkrainian() ? Ukrainian : English;
        return texts.TryGetValue(key, out var value)
            ? value
            : English.TryGetValue(key, out var fallback)
                ? fallback
                : key;
    }

    public string Format(string key, params object[] args)
    {
        return string.Format(CultureInfo.CurrentUICulture, Get(key), args);
    }

    private static bool IsUkrainian()
    {
        return string.Equals(CultureInfo.CurrentUICulture.TwoLetterISOLanguageName, "uk", StringComparison.OrdinalIgnoreCase);
    }
}
```

## Б.21 Спільний макет сторінок

Файл: `GamesInfoSys/GamesInfoSys/Pages/Shared/_Layout.cshtml`

```html
@{
    var returnUrl = $"{Context.Request.Path}{Context.Request.QueryString}";
    var currentCulture = CultureInfo.CurrentUICulture.TwoLetterISOLanguageName;
}
<!DOCTYPE html>
<html lang="@currentCulture">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] - @Text["SiteName"]</title>
    <script type="importmap"></script>
    <link rel="stylesheet" href="~/lib/bootstrap/dist/css/bootstrap.min.css" />
    <link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
    <link rel="stylesheet" href="~/GamesInfoSys.styles.css" asp-append-version="true" />
</head>
<body>
    <header>
        <nav class="navbar navbar-expand-sm navbar-toggleable-sm navbar-light bg-white border-bottom box-shadow mb-3">
            <div class="container">
                <a class="navbar-brand" asp-area="" asp-page="/Index">@Text["SiteName"]</a>
                <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target=".navbar-collapse" aria-controls="navbarSupportedContent"
                        aria-expanded="false" aria-label="Toggle navigation">
                    <span class="navbar-toggler-icon"></span>
                </button>
                <div class="navbar-collapse collapse d-sm-inline-flex justify-content-between">
                    <ul class="navbar-nav flex-grow-1">
                        <li class="nav-item">
                            <a class="nav-link text-dark" asp-area="" asp-page="/Index">@Text["NavHome"]</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link text-dark" asp-area="" asp-page="/Games/Index">@Text["NavBrowse"]</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link text-dark" asp-area="" asp-page="/Privacy">@Text["NavPrivacy"]</a>
                        </li>
                    </ul>
                    <div class="d-flex align-items-center gap-2">
                        <span class="small text-body-secondary">@Text["Language"]</span>
                        <a class="btn btn-sm @(currentCulture == "en" ? "btn-primary" : "btn-outline-secondary")" href="/set-language?culture=en&returnUrl=@Uri.EscapeDataString(returnUrl)">@Text["LangEnglish"]</a>
                        <a class="btn btn-sm @(currentCulture == "uk" ? "btn-primary" : "btn-outline-secondary")" href="/set-language?culture=uk&returnUrl=@Uri.EscapeDataString(returnUrl)">@Text["LangUkrainian"]</a>
                    </div>
                </div>
            </div>
        </nav>
    </header>
    <div class="container">
        <main role="main" class="pb-3">
            @RenderBody()
        </main>
    </div>

    <footer class="border-top footer text-muted">
        <div class="container">
            &copy; 2026 - @Text["SiteName"] - <a asp-area="" asp-page="/Privacy">@Text["FooterPrivacy"]</a>
        </div>
    </footer>

    <script src="~/lib/jquery/dist/jquery.min.js"></script>
    <script src="~/lib/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
    <script src="~/js/site.js" asp-append-version="true"></script>

    @await RenderSectionAsync("Scripts", required: false)
</body>
</html>
```

## Б.22 Стилі користувацького інтерфейсу

Файл: `GamesInfoSys/GamesInfoSys/wwwroot/css/site.css`

```css
html {
  font-size: 14px;
}

@media (min-width: 768px) {
  html {
    font-size: 16px;
  }
}

.btn:focus, .btn:active:focus, .btn-link.nav-link:focus, .form-control:focus, .form-check-input:focus {
  box-shadow: 0 0 0 0.1rem white, 0 0 0 0.25rem #258cfb;
}

html {
  position: relative;
  min-height: 100%;
}

body {
  margin-bottom: 60px;
}

.form-floating > .form-control-plaintext::placeholder, .form-floating > .form-control::placeholder {
  color: var(--bs-secondary-color);
  text-align: end;
}

.form-floating > .form-control-plaintext:focus::placeholder, .form-floating > .form-control:focus::placeholder {
  text-align: start;
}

.gis-hero {
  background: radial-gradient(1200px circle at 10% 10%, rgba(37, 140, 251, 0.35), transparent 40%),
    radial-gradient(900px circle at 90% 30%, rgba(108, 117, 125, 0.35), transparent 45%),
    linear-gradient(135deg, rgba(33, 37, 41, 1), rgba(13, 110, 253, 0.15));
}

.gis-cover {
  aspect-ratio: 16 / 9;
  object-fit: cover;
}

.gis-card {
  transition: transform 120ms ease, box-shadow 120ms ease;
}

.gis-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 0.5rem 1.25rem rgba(0, 0, 0, 0.12);
}

.gis-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.35rem;
}

.gis-shot {
  aspect-ratio: 16 / 9;
  object-fit: cover;
}
```

