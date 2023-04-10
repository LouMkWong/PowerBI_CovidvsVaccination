> A Power BI dashboard that shows the relation between vaccination rate and covid related death rate

![covidvsvac_dashboard](https://user-images.githubusercontent.com/86810684/230792682-4556256c-c84f-4913-ad2f-e4909d001729.png)

>data model

![datamodel1](https://user-images.githubusercontent.com/86810684/230792589-92caadcf-bb1e-4fdd-951e-a7d07c3a40a8.jpg)

>data preparation in SQL
````sql
select *
from coviddeaths
order by 3,4

select *
from covidvaccinations
order by 3,4

COPY coviddeaths
FROM 'C:\Users\Louis W\Desktop\little project\Covid_project\CovidDeaths.csv'
WITH (FORMAT CSV, HEADER)

COPY covidvaccination
FROM 'C:\Users\Louis W\Desktop\little project\Covid_project\CovidVaccination.csv'
WITH (FORMAT CSV, HEADER)

-- Selecting Data for visualizing

Select location, date, total_cases, new_cases, total_deaths, population
From coviddeaths
order by 1,2

-- Using Death table --

-- Total Cases vs Total Deaths
-- death rate in canada
Create View DeathRate as
Select 		location, date, total_cases, total_deaths, (total_deaths/total_cases)*100 as DeathPercentage
From 		coviddeaths
--Where location = 'Canada'
order by 	1,2

--Total Death by country
Create View DeathByCountry as
Select 		location, sum(new_cases) as TotalCases, sum(new_deaths) as TotalDeaths, sum(new_deaths)/sum(new_cases)*100 as Deathpercentage
From 		coviddeaths
where 		continent is not null and total_deaths is not null
Group by 	location



-- Total Cases vs Population 
Create View InfectionRate as
Select 		location, date, total_cases, population, (total_cases/population)*100 as InfectedPercentage
From 		coviddeaths
--Where location = 'Canada' 
order by 	1,2

-- infectedPercentage Canada vs USA
Select 		location, date, total_cases, population, (total_cases/population)*100 as InfectedPercentage
From 		coviddeaths
Where 		location = 'Canada' or location = 'United States'
order by 	1,2

-- MAX infectedPercentage World
Create View InfectionRateByCountry3 as
Select 		location, population, MAX(total_cases) as HighestInfectionCount, MAX((total_cases/population))*100 as InfectedPercentage
From 		coviddeaths
WHERE 		total_cases is not null 
			and population is not null 
			and continent is not null
Group by 	Location, Population
order by 	location desc

-- Countires with highest infection rate compared to population
Create View Top20infectionRate as
Select 		location, population, MAX(total_cases) as HighestInfectionCount, MAX((total_cases/population))*100 as InfectedPercentage
From 		coviddeaths
WHERE 		total_cases is not null and population is not null and population > 50000
Group by 	Location, Population
order by 	InfectedPercentage desc
limit 20

--Countries with highest Death count per population, ignoring null data
Select 		location, Max(Total_deaths) as TotalDeathCount, Max(Total_deaths)/Max(population)*100 as deathrate
From 		coviddeaths
where 		Total_deaths is not null and continent is not null
Group by 	location
order by 	TotalDeathCount desc

Create View FatalityRate as
Select 		location, Max(total_cases) as TotalInfected, Max(Total_deaths)/Max(total_cases)*100 as deathrate
From 		coviddeaths
where 		Total_deaths is not null and continent is not null
Group by 	location
order by 	deathrate desc

--exploring by continent
--continent with highest death rate
Create view DeathCountByRegion as
Select 		location, MAX(Total_deaths) as TotalDeathCount
From 		coviddeaths
where 		continent is null
Group by 	location
order by 	TotalDeathcount desc

--Global
Create View Globalstats as
Select 		sum(new_cases) as TotalCases, sum(new_deaths) as TotalDeaths, sum(new_deaths)/sum(new_cases)*100 as Deathpercentage
From 		coviddeaths
where 		continent is not null

-- Using Vaccinate table --

-- Total population vs vaccinations
	-- create CTE for population vs total vaccination

With 	PopVsVac (Continent, Location, Date, population, New_vaccinations, Total_Vaccinated_Rolling)
as 
	(
	Select 	dea.continent, dea.location, dea.date, dea.population, vac.new_vaccinations, 
			SUM(vac.new_vaccinations) OVER (Partition by dea.location Order by dea.location, dea.date) as Total_Vaccinated_Rolling
	from 	coviddeaths as dea
	join 	covidvaccinations as vac
			on dea.location = vac.location
			and dea.date = vac.date
	Where 	dea.continent is not null 
			-- and vac.new_vaccinations is not null 
			-- and dea.location = 'Canada'
	order 	by 2,3
	)

Select 	*, (Total_Vaccinated_Rolling/population)*100 as VaccinationPercentage
From 	PopVsVac
		Where location = 'Canada'

-- fully vaccinated rate as of MAX.date

Select 		dea.location, dea.population, Max(dea.date), Max(vac.people_fully_vaccinated) as Total_Fully_vaccinated, 
			(Max(vac.people_fully_vaccinated)/dea.population)*100 as Fully_vaccinated_percentage
from 		coviddeaths as dea
join 		covidvaccinations as vac
  			on dea.location = vac.location
			and dea.date = vac.date
Where 		dea.continent is not null 
			-- and dea.location = 'Canada'
Group by 	dea.location, dea.population
order by 	Fully_vaccinated_percentage desc

-- creating view for later , power bi

Create view FullyVaccinatedRate as
Select 		dea.location, dea.population, Max(dea.date), Max(vac.people_fully_vaccinated) as Total_Fully_vaccinated, 
			(Max(vac.people_fully_vaccinated)/dea.population)*100 as Fully_vaccinated_percentage
from 		coviddeaths as dea
join 		covidvaccinations as vac
  			on dea.location = vac.location
			and dea.date = vac.date
Where 		dea.continent is not null 
			-- and dea.location = 'Canada'
Group by 	dea.location, dea.population
order by 	Fully_vaccinated_percentage desc

Select *
from fullyvaccinatedrate
order by date desc

-- Vaccination vs deaths per million
Create View VaccinatioinRateVsNewDeathsPerMillionGlobal as
Select 		dea.location, dea.population, dea.date, vac.people_vaccinated/dea.population*100 as VacRate, dea.new_deaths_per_million
from 		coviddeaths as dea
join 		covidvaccinations as vac
			on dea.location = vac.location
			and dea.date = vac.date
Where 		dea.continent is not null
	-- and dea.location = 'Canada'
order by 	location

-- Top 20 Fully Vaccinated countries
Create View Top20FullyVaccinatedCountries as
Select 		dea.location, dea.population,
			(Max(vac.people_fully_vaccinated)/dea.population)*100 as Fully_vaccinated_percentage
from 		coviddeaths as dea
join 		covidvaccinations as vac
  			on dea.location = vac.location
			and dea.date = vac.date
Where 		dea.continent is not null 
			and vac.people_fully_vaccinated is not null
			and dea.population is not null
			and dea.population > 50000
Group by 	dea.location, dea.population
order by 	Fully_vaccinated_percentage desc
limit 20
````

