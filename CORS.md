# CORS 

CORS (Cross-Origin Resource Sharing): Allows resources to be requested from another domain outside the domain from which the first resource was served.

Configuration:

```
public void ConfigureServices(IServiceCollection services)
{
    services.AddCors(options =>
    {
        options.AddPolicy("AllowSpecificOrigin", 
            builder => builder.WithOrigins("http://example.com")
                              .AllowAnyHeader()
                              .AllowAnyMethod());
    });
    services.AddControllers();
}

public void Configure(IApplicationBuilder app)
{
    app.UseCors("AllowSpecificOrigin");
    app.UseRouting();
    app.UseEndpoints(endpoints => endpoints.MapControllers());
}
```
