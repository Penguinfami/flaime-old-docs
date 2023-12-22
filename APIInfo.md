# Updating the API

## WebAPI logic flow

1. **Controller** file defines the api's routes. Calls the service.

**Example: CategoryServiceController.cs**
``` cs
namespace Flaime.WebAPIs.Controllers
{

    [Route("api/[controller]")] // defines the service route as api/CategoryService
    [ApiController]
    public class CategoryServiceController : ControllerBase
    {
        #region member

        #endregion

        #region constructor
        public CategoryServiceController()
        { }

        public static CategoryServiceController Instance { get; } = new CategoryServiceController();
        #endregion

        #region Web APIs     

        ...

        [HttpGet("GetCategoriesByPaginationAsync")] // defines a GET request with url GetCategoriesByPaginationAsync
        public async Task<ActionResult<CategoryServiceResponse>> GetCategoriesByPaginationAsync(int pageNumber = 1, int pageSize = 20) // This request takes parameters pageNumber and pageSize. An example request would use the full url api/CategoryService/GetCategoriesByPaginationAsync?pageNumber=2&pageSize=10
        {
            var service = FlaimeServiceFactory.Instance.CategoryService;

            if (service == null)
            {
                return new CategoryServiceResponse
                {
                    StatusCode = 500,
                    Succeeded = false,
                    Message = "Instance of CategoryService is null"
                };
            }

            var resp = await service.GetCategoriesAsync(pageNumber, pageSize);

            return Ok(resp);
        }
        ...

        #endregion
    }
}

```
2. **Service** file performs the business logic. Uses the DataRepository and constructs the API response

**Example: CategoryService.cs**
``` cs
namespace Flaime.Services
{
    public class CategoryService : BaseService, ICategoryService
    {
        #region members
        private ICategoryDataRepository _CategoryDataRepository;
        #endregion

        #region constructor
        public CategoryService(FlaimeDbContext ctx) : base(ctx)
        {
            _CategoryDataRepository = new CategoryDataRepository(ctx);
        }
        #endregion

        ...

        public async Task<CategoryServiceResponse> GetCategoriesAsync(int pageNumber = 1, int pageSize = 20)
        {
            var cm = MethodBase.GetCurrentMethod();
            var response = new CategoryServiceResponse();

            try
            {
                var result = await _CategoryDataRepository.GetCategoryEntitiesByPaginationAsync(pageNumber, pageSize); // Calls the data repository method "GetCategoryEntitiesByPaginationAsync"

                if (result.Results != null && result.Results.Count > 0)
                {
                    response.ResponseObjects = result.Results.OrderBy(o => o.Name).ToList();
                }

                response.Succeeded = true;
                response.StatusCode = SUCCEEDED_CODE;
                response.Message = "Succeeded to get all categories.";
            }
            catch (Exception ex)
            {
                response.Succeeded = false;
                response.StatusCode = FAILED_CODE;
                response.Message = $"Failed to get all categories: {ex.Message}";

                Logger.Instance.LogException(ex, cm);
            }

            return response;
        }

        ...

        #endregion
    }
}
```
New services should be added to the FlaimeServiceFactory
``` cs
namespace Flaime.Services.Factories
{
    public class FlaimeServiceFactory
    {
        #region members
        AppSettingsManager? _AppSettingsManager;
        
        // declare service members
        BrandService? _BrandService = null;
        CategoryService? _CategoryService = null;
...
        public ICategoryService? CategoryService
        {
            get
            {
                SetCategoryServiceObject();
                return _CategoryService;
            }
        }
...
        private void SetCategoryServiceObject()
        {
            if (ServiceProvider != null)
            {
                _CategoryService = ServiceProvider.GetService<CategoryService>();
            }

            SetFlaimeDbContextObject();
        }
...
    private void BuildServiceCollection(FlaimeDbContext ? ctx = null)
    {
        var connString = string.Empty;

        if (FlaimeDbConnectionString != null)
        {
            connString = FlaimeDbConnectionString;
        }

        ServiceCollection = new ServiceCollection();

        if (!string.IsNullOrEmpty(connString))
        {
            ServiceCollection.AddDbContext<FlaimeDbContext>(options => options.UseNpgsql(connString), ServiceLifetime.Transient);

            // start to register service from here...
            ServiceCollection.AddTransient<BrandService>();
            ServiceCollection.AddTransient<CategoryService>();
            ServiceCollection.AddTransient<StoreService>();
            ServiceCollection.AddTransient<StoreProductService>();
            ServiceCollection.AddTransient<SubcategoryService>();
            ServiceCollection.AddTransient<ReportService>();

            // get service provider 
            ServiceProvider = ServiceCollection.BuildServiceProvider();
        }
    }
```
3. The **DataRepository** file at Flaime.DataRepositories for a table fetches table data. Calls queries, converting the query results from Data classes to Entity classes.
``` cs
namespace Flaime.DataRepositories
{
    public class CategoryDataRepository : BaseDataRepository, ICategoryDataRepository
    {
        #region members
        private QueryCategoryData? _QueryCategoryData;
        #endregion

        ...

        public async Task<PagedResult<CategoryEntity>> GetCategoryEntitiesByPaginationAsync(int pageNumber = 1, int pageSize = 20)
        {
            var results = new List<CategoryEntity>();
            var cm = MethodBase.GetCurrentMethod();
            var pagedResult = new PagedResult<CategoryEntity>();

            QueryCategoryData.QueryableData = FlaimeDbContext == null ? null : FlaimeDbContext.Categories;

            try
            {
                var qry = QueryCategoryData == null ? null : QueryCategoryData
                    .QueryAll() // Calls the Query method
                    .IncludeCategoryEntityProfile(); // Creates the object representation (Entity class) of the queried data

                if (qry != null)
                {
                    // add results to the query result object
                    pagedResult.CurrentPageNumber = pageNumber;
                    pagedResult.PageSize = pageSize;

                    pagedResult.TotalRowCount = qry.Count();
                    results = await qry.Skip((pageNumber - 1) * pageSize).Take(pageSize).ToListAsync();
                    pagedResult.Results = results;
                }
            }
            catch (Exception ex)
            {
                Logger.Instance.LogException(ex, cm);
                throw new Exception(ex.Message);
            }

            return pagedResult;
        }

        ...

    }
}
```

To support unit tests, add the DataRepository to the FlaimeDbContextFactory and DataRepositoryFactory files in Flaime.DataRepositories.Factories

FlaimeDbContextFactory.cs
``` cs
namespace Flaime.DataRepositories.Factories
{
    /// <summary>
    /// support unit test
    /// </summary>
    public class FlaimeDbContextFactory
    {
        #region members
        AppSettingsManager? _AppSettingsManager;
        IConfiguration? Configuration { get; set; }

        // declare data repository members
        IBrandDataRepository? _BrandDataRepository;
        ICategoryDataRepository? _CategoryDataRepository;
        IStoreDataRepository? _StoreDataRepository;
        IStoreProductDataRepository? _StoreProductDataRepository = null;
        ISubcategoryDataRepository? _SubcategoryDataRepository;
        IStoreProductNutritionFactDataRepository? _StoreProductNutritionFactDataRepository;

...

        public ICategoryDataRepository? CategoryDataRepository
        {
            get
            {
                return _CategoryDataRepository;
            }
        }

...
        private void Initialization()
        {
            var cm = MethodBase.GetCurrentMethod();
            
            try
            {
                _AppSettingsManager = AppSettingsManager.Instance;

                if (_AppSettingsManager != null)
                {
                    _AppSettingsManager.LoadAppSettings();

                    FlaimeDbConnectionString = _AppSettingsManager.FlaimeDbConnectionString;
                        
                    if (string.IsNullOrEmpty(FlaimeDbConnectionString))
                    {
                        throw new Exception("Flaime database connection string was not found");
                    }
                }
                
                if(FlaimeDbContext != null && !FlaimeDbContext.IsDisposed)
                {
                    FlaimeDbContext.Dispose();
                }

                BuildServiceCollection();

                if(ServiceProvider != null)
                {
                    FlaimeDbContext = ServiceProvider.GetService<FlaimeDbContext>();

                    if (FlaimeDbContext != null)
                    {
                        FlaimeDbContext.DatabaseConnectionString = string.IsNullOrEmpty(FlaimeDbConnectionString) ? String.Empty : FlaimeDbConnectionString;

                        // start to get instance of repository objects
                        _BrandDataRepository = ServiceProvider.GetService<BrandDataRepository>();
                        _CategoryDataRepository = ServiceProvider.GetService<CategoryDataRepository>();

...
        private void BuildServiceCollection()
        {
            var connString = string.Empty;

            if (FlaimeDbConnectionString != null)
            {
                connString = FlaimeDbConnectionString;
            }

            /************************************************
            // WQ: following code is for testing purpose:
            var builder = WebApplication.CreateBuilder(args);
            ServiceCollection = builder.Services;                        
            *************************************************/

            ServiceCollection = new ServiceCollection();

            if (!string.IsNullOrEmpty(connString))
            {
                ServiceCollection.AddDbContext<FlaimeDbContext>(options => options.UseNpgsql(connString));

                // inject data repository object from here...
                ServiceCollection.AddScoped<BrandDataRepository>();
                ServiceCollection.AddScoped<CategoryDataRepository>();
```
DataRepositoryFactory.cs

``` cs
namespace Flaime.DataRepositories.Factories
{
    /// <summary>
    /// support FlaimeService
    /// </summary>
    public class DataRepositoryFactory
    {
        #region members
        IBrandDataRepository? _BrandDataRepository;
        ICategoryDataRepository? _CategoryDataRepository;
        IStoreDataRepository? _StoreDataRepository;

...

    public ICategoryDataRepository GetCategoryDataRepository(FlaimeDbContext ctx)
    {
        try
        {
            CheckFlaimeDbContext();

            _CategoryDataRepository = _ServiceProvider == null ? null : _ServiceProvider.GetService<CategoryDataRepository>();

            if (_CategoryDataRepository != null)
            {
                _CategoryDataRepository.SetFlaimeDbContext(ctx);
                return _CategoryDataRepository;
            }

            throw new InvalidOperationException("CategoryDataRepository object is null");
        }
        catch (Exception ex)
        {
            throw new Exception(ex.Message);
        }
    }
...

    #region support methods
    /// <summary>
    /// VERY IMPORTANT TO INJECT DATA REPOSITORY INSTANCE FROM THIS METHOD
    /// </summary>
    private void LoadServiceCollection()
    {
        _ServiceCollection = new ServiceCollection();

        _ServiceCollection.AddScoped<BrandDataRepository>();
        _ServiceCollection.AddScoped<CategoryDataRepository>();
        _ServiceCollection.AddScoped<StoreDataRepository>();
        _ServiceCollection.AddScoped<StoreProductDataRepository>();
        _ServiceCollection.AddScoped<StoreProductNutritionFactDataRepository>();
        _ServiceCollection.AddScoped<CategoryReportDataRepository>();

        _ServiceProvider = _ServiceCollection.BuildServiceProvider();
    }
```

4. The **Query** file for a table defines the individual queries.
``` cs
namespace Flaime.DataRepositories.Queries
{
    internal class QueryCategoryData : BaseQuery<Category>
    {
        #region methods

        ...

        internal QueryCategoryData QueryAll()
        {
            base.CheckDbContext();
            base.FilterOutDeletedData();

            QueryableData = QueryableData == null ? null : QueryableData.OrderBy(x => x.Name);

            return this;
        }

        ...
        
        internal IQueryable<CategoryEntity>? IncludeCategoryEntityProfile()
        {
            var qry = QueryableData == null ? null : QueryableData // map the query data to their entity object representations
                .Select(obj => new CategoryEntity
                {
                    Id = obj.Id,
                    Name = obj.Name,
                    OrigName = obj.Name,
                    CreatedBy = obj.CreatedBy,
                    CreatedDateTime = obj.CreatedDateTime,
                    Deleted = obj.Deleted,
                    DeletedBy = obj.DeletedBy,
                    DeletedDateTime = obj.DeletedDateTime,
                    ModifiedBy = obj.ModifiedBy,
                    ModifiedDateTime = obj.ModifiedDateTime,
                });

            return qry;
        }

        internal IQueryable<CategoryEntity>? IncludeCategoryEntityProfileWithScucategoryEntities()
        {
            var qry = QueryableData == null ? null : QueryableData
                .Select(cat => new CategoryEntity
                {
                    Id = cat.Id,
                    Name = cat.Name,
                    OrigName = cat.Name,
                    CreatedBy = cat.CreatedBy,
                    CreatedDateTime = cat.CreatedDateTime,
                    Deleted = cat.Deleted,
                    DeletedBy = cat.DeletedBy,
                    DeletedDateTime = cat.DeletedDateTime,
                    ModifiedBy = cat.ModifiedBy,
                    ModifiedDateTime = cat.ModifiedDateTime,

                    SubcategoryEntities = cat.Subcategories.Select(sub => new SubcategoryEntity { 
                        Id = sub.Id,
                        Name = sub.Name,
                        Code = sub.Code,
                        CategoryEntityId = sub.CategoryId,
                        CreatedBy = sub.CreatedBy,
                        CreatedDateTime = sub.CreatedDateTime,
                        Deleted = sub.Deleted,
                        DeletedBy = sub.DeletedBy,
                        DeletedDateTime = sub.DeletedDateTime,
                        ModifiedBy = sub.ModifiedBy,
                        ModifiedDateTime = sub.ModifiedDateTime, 
                        OrigCode = sub.OrigCode,
                        OrigName = sub.OrigName
                    }).ToList()
                });

            return qry;
        }
        #endregion

        #region constract
        internal QueryCategoryData(FlaimeDbContext? ctx) : base(ctx)
        {
            base.QueryableData = ctx == null ? null: ctx.Categories; // all queries defined in this class use the Categories table
        }

        #endregion
    }
}
```
