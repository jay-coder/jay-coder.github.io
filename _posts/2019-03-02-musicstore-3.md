---
layout: post
title: MusicStore - Part3 - EF to SQL Server
redirect_from:
  - /musicstore-3/
---

In [MusicStore - Part1 - Init](/2019/02/26/musicstore-1/), in memory resources and test users had been used for test purpose.


```csharp
// Startup.cs in IdentityServer
public void ConfigureServices(IServiceCollection services)
{
    ......
    services.AddIdentityServer()
	.AddDeveloperSigningCredential()
	.AddInMemoryIdentityResources(Resources.GetIdentityResources())
	.AddInMemoryApiResources(Resources.GetApiResources())
	.AddInMemoryClients(new PortalClientFactory(Configuration).GetClients())
	.AddTestUsers(TestUsers.Users);
    ......
}
```

In real world, credentials always be stored in database. Thus, I will use SQL Server + Entity Framework to replace in memory resources.

1. Create a new database called MusicStore_IdentityServer

	```json
	// appsettings.Development.json
	"ConnectionStrings": {
		"IdentityServer4Connection": "Data Source=.;Initial Catalog=MusicStore_IdentityServer;Integrated Security=False;User ID=musicstore_user;Password=P@ssword!@#$;"
	}
	```

2. Migrate resources to SQL Server

	* Run CMD and navigate to Projects > IdentityServer folder
	* Use EF Code First to migrate PersistedGrantDb schema to C# script
		
		```shell
		dotnet ef migrations add InitialIdentityServerPersistedGrantDbMigration -c PersistedGrantDbContext -o Migrations/IdentityServer/PersistedGrantDb
		```

	* Use EF Code First to migrate ConfigurationDb schema to C# script

		```shell
		dotnet ef migrations add InitialIdentityServerConfigurationDbMigration -c ConfigurationDbContext -o Migrations/IdentityServer/ConfigurationDb
		```

	* Import EF scripts to database in IdentityServer start-up
		```csharp
		// Startup.cs
		public void Configure(IApplicationBuilder app)
		{
			// Init Database using EF Code First
			InitializeDatabase(app);
			app.UseStaticFiles();
			app.UseIdentityServer();
			app.UseMvcWithDefaultRoute();
		}

		private void InitializeDatabase(IApplicationBuilder app)
		{
			using (var serviceScope = app.ApplicationServices.GetService<IServiceScopeFactory>().CreateScope())
			{
				//PersistedGrant DB Context
				var persistedGrantContext = serviceScope.ServiceProvider.GetRequiredService<PersistedGrantDbContext>();
				persistedGrantContext.Database.Migrate();
				//Configuration DB Context
				var configContext = serviceScope.ServiceProvider.GetRequiredService<ConfigurationDbContext>();
				configContext.Database.Migrate();
				if (!configContext.Clients.Any())
				{
					var clientList = new PortalClientFactory(Configuration).GetClients();
					if (clientList != null &amp;&amp; clientList.Any())
					{
						foreach (var client in clientList)
						{
							configContext.Clients.Add(client.ToEntity());
						}
						configContext.SaveChanges();
					}
				}

				if (!configContext.IdentityResources.Any())
				{
					foreach (var resource in Resources.GetIdentityResources())
					{
						configContext.IdentityResources.Add(resource.ToEntity());
					}
					configContext.SaveChanges();
				}

				if (!configContext.ApiResources.Any())
				{
					foreach (var resource in Resources.GetApiResources())
					{
						configContext.ApiResources.Add(resource.ToEntity());
					}
					configContext.SaveChanges();
				}
			}
		}
		```
	* Run IdentityServer project
	* Check if database tables are auto generated
	* Check if clients are imported correctly

		<img src='{{ "/public/assets/img/musicstore_sql_clients.png" | relative_url }}' alt="SQL Select Clients" />

3. Integrate ASP.NET Identity
	* Create a new .NET Standard project under Core folder called Domain
	* Create a new class "ApplicationUser" to inherit from IdentityUser
		```csharp
		public class ApplicationUser : IdentityUser
		{
			public string FirstName { get; set; }
			public string LastName { get; set; }
			public DateTime DOB { get; set; }
			public int Gender { get; set; }
			public string Avatar { get; set; }
			public int Language { get; set; }
			public bool Enabled { get; set; }
			public EnumUserType UserType { get; set; }
		}
		```

	* Create a new .NET Standard project under Foundations folder called SQLServer
	* Create a new class "AspNetIdentityDbContext" to inherit from IdentityDbContext
		
		```csharp
		public class AspNetIdentityDbContext : IdentityDbContext<ApplicationUser>
		{
			......
		}
		```

	* Run CMD and navigate to Foundations > SQLServer
	* Use EF Code First to migrate AspNetIdentityDbContext schema to C# script

		```shell
		dotnet ef migrations add InitialAspNetIdentityDbMigration -c AspNetIdentityDbContext -o Migrations/AspNetIdentityDb -s ../../Projects/IdentityServer
		```

		<p class="message">
			NOTE: As SQLServer is not a executable project, it needs to refer to a startup project (--startup-project or -s)
		</p>

	* Import EF scripts for AspNet Identity

		Append this code snip to InitializeDatabase method

		```csharp
		// AspNet Identity DB Context
		var idContext = serviceScope.ServiceProvider.GetRequiredService<AspNetIdentityDbContext>();
		if (!idContext.Users.Any())
		{
			var userManager = serviceScope.ServiceProvider.GetService<UserManager<ApplicationUser>>();
			foreach (var user in Resources.GetApplicationUsers())
			{
				var result1 = userManager.CreateAsync(user , "Test123!").Result;
			}
		}
		idContext.Database.Migrate();
		```

	* Run IdentityServer project
	* Check if database tables and fields are auto generated

		<img src='{{ "/public/assets/img/musicstore_aspnet_identity.png" | relative_url }}' alt="ASP.NET Identity" />

4. Replace in memory resources
	
	* Init DbContext ConnectionString

		```csharp
		//Startup.cs
		public void ConfigureServices(IServiceCollection services)
		{
				......
			var connectionString = Configuration.GetConnectionString("IdentityServer4Connection");
			// Init DbContext ConnectionString
			var aspnetIdentityAssembly = typeof(AspNetIdentityDbContext).GetTypeInfo().Assembly.GetName().Name;
			services.AddDbContext<AspNetIdentityDbContext>(options =>
				options.UseSqlServer(connectionString, sql => sql.MigrationsAssembly(aspnetIdentityAssembly))
			);
				......
		}
		```

	* Authenticate via AspNet Identity

		```csharp
		//Startup.cs
		public void ConfigureServices(IServiceCollection services)
		{
			......
			// Apply AspNetIdentity as default token provider
			// UserManager can be used to manage users
			services.AddIdentity<ApplicationUser, IdentityRole>(options =>
			{
				// Password settings
				options.Password.RequireDigit = true;
				options.Password.RequiredLength = 8;
				options.Password.RequireLowercase = true;
				options.Password.RequireNonAlphanumeric = true;
				options.Password.RequireUppercase = true;
				// User settings
				options.User.RequireUniqueEmail = true;
			})
			.AddEntityFrameworkStores<AspNetIdentityDbContext>()
			.AddDefaultTokenProviders();
		}
		```

	* Load resources from SQL Server

		```csharp

		//Startup.cs
		public void ConfigureServices(IServiceCollection services)
		{
			......
			// Load resources from DB
			var identityServerAssembly = typeof(Startup).GetTypeInfo().Assembly.GetName().Name;
			var identityServer = services
				.AddIdentityServer()
				.AddAspNetIdentity<ApplicationUser>()
				// this adds the config data from DB (clients, resources, CORS)
				.AddConfigurationStore(options =>
				{
					options.ConfigureDbContext = builder =>
						builder.UseSqlServer(connectionString,
							sql => sql.MigrationsAssembly(identityServerAssembly));
				})
				// this adds the operational data from DB (codes, tokens, consents)
				.AddOperationalStore(options =>
				{
					options.ConfigureDbContext = builder =>
						builder.UseSqlServer(connectionString,
							sql => sql.MigrationsAssembly(identityServerAssembly));
					// this enables automatic token cleanup. this is optional.
					options.EnableTokenCleanup = true;
					// options.TokenCleanupInterval = 15; // interval in seconds. 15 seconds useful for debugging
				});
				......
		}
		```

		<p class="message">NOTE: There are 2 migrations from 2 projects - IdentityServer and SQLServer, in order to set the connection string, the assembly name needs to be specified as the second parameter to identify where the context comes from.  For example:</p>

		```csharp
		services.AddDbContext<AspNetIdentityDbContext>(options =>
			options.UseSqlServer(connectionString, sql => sql.MigrationsAssembly(aspnetIdentityAssembly))
		);
		```


5. Test

	Use either <strong>bob / Test123$</strong> or <strong>alice / Test123$</strong> to login, new password "Test123$" has been applied because of the password settings in #4
