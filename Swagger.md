# Swagger

## What is Swagger and how can you integrate it with .NET Core Web API?

## Answer:

Swagger: An open-source tool that provides documentation for your API endpoints.

## Integration with .NET Core:
Install Swashbuckle.AspNetCore package.
   
## Configure in Startup.cs:
  
```
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });
    });
}

public void Configure(IApplicationBuilder app)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    
    app.UseSwagger();
    app.UseSwaggerUI(c => c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1"));
    
    app.UseRouting();
    app.UseEndpoints(endpoints => endpoints.MapControllers());
}
```
