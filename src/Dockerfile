# Use the official .NET 5 SDK image as the build environment
FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build
WORKDIR /src

# Copy the .csproj files and restore any dependencies
COPY DevOpsChallenge.SalesApi.Business/*.csproj ./DevOpsChallenge.SalesApi.Business/
COPY DevOpsChallenge.SalesApi.Database/*.csproj ./DevOpsChallenge.SalesApi.Database/
COPY DevOpsChallenge.SalesApi/*.csproj ./DevOpsChallenge.SalesApi/
RUN dotnet restore ./DevOpsChallenge.SalesApi/DevOpsChallenge.SalesApi.csproj

# Copy the rest of the project files
COPY DevOpsChallenge.SalesApi.Business ./DevOpsChallenge.SalesApi.Business/
COPY DevOpsChallenge.SalesApi.Database ./DevOpsChallenge.SalesApi.Database/
COPY DevOpsChallenge.SalesApi ./DevOpsChallenge.SalesApi/

# Build the application
WORKDIR /src/DevOpsChallenge.SalesApi
RUN dotnet publish -c Release -o /app/publish

# Use the official ASP.NET Core runtime image as the runtime environment
FROM mcr.microsoft.com/dotnet/aspnet:5.0 AS runtime
WORKDIR /app

# Copy the build output from the build environment
COPY --from=build /app/publish .

# Expose the port the application runs on
EXPOSE 80
EXPOSE 90

# Set the entry point to run the application
ENTRYPOINT ["dotnet", "DevOpsChallenge.SalesApi.dll"]
