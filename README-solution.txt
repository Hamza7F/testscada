Agar .sln file nahi hai to command line se bana lo:

cd scada_demo_test
dotnet new sln -n test_today_one
dotnet sln add src/scada_demo_test.Domain/scada_demo_test.Domain.csproj
dotnet sln add src/scada_demo_test.Application/scada_demo_test.Application.csproj
dotnet sln add src/scada_demo_test.Infrastructure/scada_demo_test.Infrastructure.csproj
dotnet sln add src/scada_demo_test.API/scada_demo_test.API.csproj
dotnet sln add src/scada_demo_test.Web/scada_demo_test.Web.csproj
