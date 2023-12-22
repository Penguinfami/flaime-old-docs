# Steps For Updating the Database

# Creating a new table
1. Create a corresponding Data class in Flaime.EntityModels.Data, with class fields representing each column in the table. By default, a new column takes the name (matching case) of the class field. To provide a different name (ex. forgoing camel case used for .NET in favour of snake case for Postgres) for the table/columns, use annotations (ex. [Table("new_table")], [Column("col")]). If a field is not meant to be mapped to a column, use the [NotMapped] annotation.
``` cs
    // create this class
   namespace Flaime.EntityModels.Data
    {
        [Table("pies")]
        public class Pie: AbstractData
        {
            #region properties
            [Key]
            [Column("id")]
            public override int Id { get => base.Id; set => base.Id = value; }

            [Column("flavour")]
            public string Flavour { get; set; } = string.Empty;

            [Column("quantity")]
            public int Quantity { get; set; }

            [NotMapped]
            public bool HasFlavour
            {
                get
                {
                    return string.IsNullOrWhiteSpace(Flavour);
                }
            }
            #endregion
        }
    }
```
2. Add the corresponding DbSet field in the FlaimeDbContext class at Flaime.DataContext

``` cs
namespace Flaime.DataContext
{
    public class FlaimeDbContext : IdentityDbContext<FlaimeUser>
    {
        ...

        #region properties
        public string InstanceId { get { return _instanceId; } }

        public bool IsDisposed { get; private set; } = false;

        public string DatabaseConnectionString { get; set; } = String.Empty;

        public DbSet<AllergensWarning>? AllergensWarnings { get; set; }

        ...

        public DbSet<StoreProductImage>? StoreProductImages { get; set; }

        public DbSet<StoreProductNutritionFact>? StoreProductNutritionFacts { get; set; }

        public DbSet<Pie>? Pies { get; set; } // Add this 
        #endregion

        ...
    }
}


```
### Not required for updating the database, but should still include 
**For schema configurations**

3. Define/update relationships, data type attributes, collation usage, default values in the corresponding EntityConfiguration class in Flaime.DataContext.EntityTypeConfigurations

``` cs
// create this class
namespace Flaime.DataContext.EntityTypeConfigurations
{
    internal class PieEntityConfiguration: IEntityTypeConfiguration<Pie>
    {
        #region methods
        public void Configure(EntityTypeBuilder<Pie> builder)
        {
            builder.Property(o => o.CreatedDateTime)
                .HasColumnType("timestamp with time zone") 
                .HasDefaultValueSql("CURRENT_TIMESTAMP")
                .ValueGeneratedOnAdd();

            builder.Property(o => o.ModifiedDateTime)
                .HasColumnType("timestamp with time zone")
                .HasDefaultValueSql("CURRENT_TIMESTAMP")
                .ValueGeneratedOnAddOrUpdate();

            builder.Property(o => o.Deleted).HasDefaultValueSql("FALSE").ValueGeneratedOnAdd();
           
            builder.Property(o => o.Flavour).HasMaxLength(256).UseCollation("english_ci");
            builder.Property(o => o.Quantity).HasDefaultValueSql("0").ValueGeneratedOnAdd();
        }
        #endregion
    }
}

```
If a new configuration class was created, a line would then need to be added to the `ModelBuilderManager.cs` class to apply the configuration
``` cs
namespace Flaime.DataContext
{
    internal class ModelBuilderManager : AbstractModelBuilderManager
    {
        #region members
        private static ModelBuilderManager _Instance = new ModelBuilderManager();
        #endregion

        #region properties
        internal static ModelBuilderManager Instance
        {
            get { return _Instance; }
        }
        #endregion

        #region methods
        public void CreateModels()
        {
            try
            {
                ModelBuilder.ApplyConfiguration(new FlaimeUserEntityConfiguration());
                
                ...
                
                ModelBuilder.ApplyConfiguration(new ProductNutritionFactEntityConfiguration());
                ModelBuilder.ApplyConfiguration(new StoreProductNutritionFactEntityConfiguration());
                ModelBuilder.ApplyConfiguration(new PieEntityConfiguration()); // Add this
            }
            catch (Exception ex)
            {
                throw new Exception(ex.Message);
            }
        }
        #endregion
    }
}
```
**Used for WebAPI development**

4. Create a corresponding Entity class in Flaime.EntityModels.Entities.  The Data fetched (in the form of the Data class) from the database context gets mapped to Entity classes when they get passed to the Service layer, meaning the API returns response objects stemming from these EntityModel classes. Therefore, we want to include fields representing related Entity classes based on the foreign key relationships defined in the Entity configuration file, since we want the API services to have the ability to return data from related tables as well. (For example, the StoreProductService endpoints for searching may return related data for the store product's category, nutrient info, scrape batch, store etc)

``` cs
// create this class
namespace Flaime.EntityModels.Entities
{
    public class PieEntity : BaseEntity
    {
        #region properties
        public override int Id { get => base.Id; set => base.Id = value; }
        public string Flavour { get; set; } = string.Empty;
        public int Quantity { get; set; }
        public bool HasFlavour
        {
            get
            {
                return string.IsNullOrWhiteSpace(Flavour);
            }
        }
        #endregion
    }
}
```

### Migrating changes to the database
5. The **appsettings.json** in Flaime.App has a variable "FlaimeDbContextConnection" which defines the db host, port, user, database, password etc to connect.
   - Flaime.WebAPIs and the Unit Test projects also contain their respective **appsettings.json** files which would define the database used for the web API and unit tests

``` cs
"ConnectionStrings": {
  "FlaimeDbContextConnection": "Host=172.17.10.95;Port=5432;User ID=postgres;Password=flaimedotnet;Database=flaime_db;Pooling=true",
  ...
}
```
6. See the commands in Flaime.DataContext.Utilities.CodeFirstCommands.txt, run them in Visual Studio's Package Manager Console with Flaime.App set as the startup project and Flaime.DataContext set as the default project:
```
Add new migration:
PM>add-migration "<new unique migration name>" -context FlaimeDbContext

Appliy new migration to database
PM>update-database

Generate upgrade scripts:
PM>script-migration -from "<previous migration name of the target migration>" -to "<target migration name>" -context FlaimeDbContext

remove migration:
remove-migration
```
`add-migration` generates the migration C# file in the **migrations** folder. 

`script-migration` generates the SQL file representing the migration, which is recommended to verify the correctness of the changes to be made. 

`update-database` applies the migration to the database. 

`remove-migration` reverses the `add-migration` command, and cannot undo changes already applied (create a new migration to undo the changes in that case). 

# Updating a table

Ex. Adding a new "size" column, and removing the "quantity" column

``` cs
   namespace Flaime.EntityModels.Data
    {
        [Table("pies")]
        public class Pie: AbstractData
        {
            #region properties
            [Key]
            [Column("id")]
            public override int Id { get => base.Id; set => base.Id = value; }

            [Column("size")] // Name the new column "size"
            public int Size {get; set; } // The column has an integer type

            [Column("flavour")]
            public string Flavour { get; set; } = string.Empty;

            // Remove the "Quantity" field

            [NotMapped]
            public bool HasFlavour
            {
                get
                {
                    return string.IsNullOrWhiteSpace(Flavour);
                }
            }
            #endregion
        }
    }
```
The corresponding EntityModel and EntityTypeConfiguration classes will also need to be updated to include the new columns and remove the columns that no longer exist.