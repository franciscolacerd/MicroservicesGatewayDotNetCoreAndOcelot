# MicroservicesGatewayDotNetCoreAndOcelot
TOOLS - How to create microservices gateway with DotNetCore, Ocelot and Swagger 


1. Create ASP.NET Core Web App (Model-View-Controller)  project 3.1.

2. Install Nugets:

>MMLib.SwaggerForOcelot (3.2.1)

>MMLib.Ocelot.Provider.AppConfiguration (1.1.0)

>Ocelot (16.0.1)

>Swashbuckle.AspNetCore (6.1.2)

3. Add to Program.cs in ConfigureAppConfiguration((hostingContext, config) =>

             .AddOcelotWithSwaggerSupport((o) => o.Folder = hostingContext.HostingEnvironment.IsDevelopment() ? "Configuration/Dev" : "Configuration/Prod")

```
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using MMLib.SwaggerForOcelot.DependencyInjection;

namespace Gateway
{
    public static class Program
    {
        public static void Main(string[] args)
        {
            CreateHostBuilder(args).Build().Run();
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureAppConfiguration((hostingContext, config) =>
                {
                    config
                        .SetBasePath(hostingContext.HostingEnvironment.ContentRootPath)
                        .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                        .AddJsonFile($"appsettings.{hostingContext.HostingEnvironment.EnvironmentName}.json",
                            optional: true, reloadOnChange: true)
                        .AddJsonFile($"appsettings.local.json", optional: true, reloadOnChange: true)
                        .AddOcelotWithSwaggerSupport((o) => o.Folder = hostingContext.HostingEnvironment.IsDevelopment() ? "Configuration/Dev" : "Configuration/Prod")
                        .AddEnvironmentVariables();
                })
                .ConfigureWebHostDefaults(webBuilder => webBuilder.UseStartup<Startup>()).ConfigureLogging(logging => logging.AddConsole());
    }
}
```

4. Add to Startup.cs in ConfigureServices(IServiceCollection services)

            services.AddOcelot();
            services.AddSwaggerGen();
            services.AddSwaggerForOcelot(Configuration);
            
5. Add to Startup.cs in Configure(IApplicationBuilder app, IWebHostEnvironment env)

            app.UseSwaggerForOcelotUI(opt => opt.PathToSwaggerGenerator = "/swagger/docs");
            app.UseSwagger();
            app.UseSwaggerUI(c =>
            {
                c.SwaggerEndpoint("/swagger/v1/swagger.json", "Gateways");
            });      
            
6. Add to Startup.cs, at bottom in Configure(IApplicationBuilder app, IWebHostEnvironment env)

            app.UseOcelot().Wait();

```
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Ocelot.DependencyInjection;
using Ocelot.Middleware;

namespace Gateway
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddOcelot();
            services.AddSwaggerGen();
            services.AddSwaggerForOcelot(Configuration);
            services.AddControllersWithViews();
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            app.UseSwaggerForOcelotUI(opt => opt.PathToSwaggerGenerator = "/swagger/docs");

            app.UseSwagger();

            app.UseSwaggerUI(c =>
            {
                c.SwaggerEndpoint("/swagger/v1/swagger.json", "Gateways");
            });

            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }
            else
            {
                app.UseExceptionHandler("/Home/Error");
                // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
                app.UseHsts();
            }
            app.UseHttpsRedirection();
            app.UseStaticFiles();

            app.UseRouting();

            app.UseAuthorization();

            app.UseOcelot().Wait();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllerRoute(
                    name: "default",
                    pattern: "{controller=Home}/{action=Index}/{id?}");
            });
        }
    }
}
```

7. Add configuration files at:

![image](https://user-images.githubusercontent.com/6674269/115422570-f043cf00-a1f4-11eb-97cd-67496f4c1a76.png)


8. With configurations:


ocelot.Development.json
```
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 44312
        }
      ],
      "UpstreamPathTemplate": "/{everything}",
      "UpstreamHttpMethod": [],
      "SwaggerKey": "ABCApi"
    }
  ]
}
```


ocelot.global.json
```
{
  "GlobalConfiguration": {
    "BaseUrl": "https://localhost"
  }
}
```

ocelot.SwaggerEndPoints.json
```
{
  "SwaggerEndPoints": [
    {
      "Key": "ABCApi",
      "Config": [
        {
          "Name": "ABC Api",
          "Version": "v1",
          "Url": "https://localhost:44312/swagger/v1/swagger.json"
        }
      ]
    }
  ]
}
```

9. Set Development enviroment at:

![image](https://user-images.githubusercontent.com/6674269/115423606-df478d80-a1f5-11eb-97c5-3314fec0567d.png)

10. Configure "Tokenize in Archive" at "release pipeline" in Azure Devops CI/CD from ocelot.json transformation:

![image](https://user-images.githubusercontent.com/6674269/115299165-5d545780-a156-11eb-82bc-ae9ab76873cd.png)

11. Configure Variables at "release pipeline" in Azure Devops CI/CD from ocelot.json transformation:

![image](https://user-images.githubusercontent.com/6674269/115299382-a3a9b680-a156-11eb-9e9a-d3e6ae829e08.png)

12. Add script to ocelot.json:

```
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "#{Host}#",
          "Port": #{Port}#
        }
      ],
      "UpstreamPathTemplate": "/{everything}",
      "UpstreamHttpMethod": [],
      "SwaggerKey": "ABCApi"
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "#{BaseUrl}#"
  },
  "SwaggerEndPoints": [
    {
      "Key": "ABCApi",
      "Config": [
        {
          "Name": "ABC Api",
          "Version": "v1",
          "Url": "#{SwaggerUrl}#/swagger/v1/swagger.json"
        }
      ]
    }
  ]
}

```


