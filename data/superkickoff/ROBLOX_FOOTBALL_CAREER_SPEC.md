# Football Career Simulation Game - Roblox Version
## Complete Specification for Pure UI-Based Roblox Game

## 🎮 Game Overview
A **pure UI-based** football (soccer) career simulation game built entirely in Roblox using ScreenGuis. Players manage a footballer's career from lower leagues to top-tier competitions. No 3D gameplay - all interactions happen through UI menus and screens.

---

## 🛠️ Core Technologies

### Roblox Platform
- **Language**: Lua (Luau)
- **UI System**: ScreenGuis with Frames, TextLabels, TextButtons, ScrollingFrames
- **Data Persistence**: DataStore Service (or ProfileService/DataStore2 for advanced users)
- **Architecture**: ModuleScripts for code organization
- **Client-Server**: RemoteEvents and RemoteFunctions for communication

### Key Roblox Services Used
```lua
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerStorage = game:GetService("ServerStorage")
local DataStoreService = game:GetService("DataStoreService")
local HttpService = game:GetService("HttpService")
```

---

## 📊 Data System

### Datapack Loading
The game loads football data from this CDN:
```
https://cdn.jsdelivr.net/gh/pren8/footballdatabase/data2/datapack-september-2025_25-09-2025_156.json
```

### Raw Datapack Structure
The JSON uses a `colonne` (columns) and `dati` (data) format:

```json
{
  "metadata": {
    "id": "099873",
    "datapackName": "Datapack 2025/26",
    "description": "...",
    "creationDate": "2025-09-22",
    "version": "1.0",
    "targetAppVersion": 126,
    "startingYear": 2025
  },
  "leagues": {
    "colonne": ["id", "name", "division", "nation", "continent", "number_of_teams", "type", ...],
    "dati": [
      ["ARGC", "Liga Profesional de Fútbol", 1, "Argentina", "SA", 30, "apertura_clausura_1", ...]
    ]
  },
  "teams": {
    "colonne": ["id", "name", "nameCode", "leagueId", "logo", "stadiumId", "managerId", "teamValue"],
    "dati": [
      [3211, "River Plate", "RPL", "ARGC", {...}, 1009, 19439, 86052000]
    ]
  },
  "players": {
    "colonne": ["id", "name", "surname", "birthdate", "jerseyNumber", "overall", "marketValue", "roles", "foot", "nationality", "height", "endContract", "teamId"],
    "dati": [
      [329601, "Miguel", "Borja", "1993/01/26", "9", "74.46", 3600000, [...], "right", "Colombia", 183, "2025/12/31", 3211]
    ]
  },
  "stadiums": {
    "colonne": ["id", "name", "capacity"],
    "dati": [[1009, "Estadio Mâs Monumental", 85018]]
  },
  "managers": {
    "colonne": ["id", "name", "country"],
    "dati": [[19439, "Marcelo Gallardo", "Argentina"]]
  }
}
```

### Indexed Datapack (Lua Tables)
Convert raw data to efficient lookup tables using Lua:

```lua
-- ModuleScript: DatapackLoader
local DatapackLoader = {}

DatapackLoader.IndexedDatapack = {
    metadata = {},
    leagues = {}, -- {[id] = {id, name, division, ...}}
    teams = {}, -- {[id] = {id, name, leagueId, ...}}
    players = {}, -- {[id] = {id, name, surname, ...}}
    stadiums = {}, -- {[id] = {id, name, capacity}}
    managers = {}, -- {[id] = {id, name, country}}
    teamsByLeague = {}, -- {[leagueId] = {teamId1, teamId2, ...}}
    playersByTeam = {}, -- {[teamId] = {playerId1, playerId2, ...}}
}

function DatapackLoader:ParseDatapack(rawData)
    local indexed = {
        metadata = rawData.metadata,
        leagues = {},
        teams = {},
        players = {},
        stadiums = {},
        managers = {},
        teamsByLeague = {},
        playersByTeam = {}
    }
    
    -- Parse leagues
    local leagueColumns = rawData.leagues.colonne
    for _, row in ipairs(rawData.leagues.dati) do
        local league = {}
        for i, col in ipairs(leagueColumns) do
            league[col] = row[i]
        end
        indexed.leagues[league.id] = league
        indexed.teamsByLeague[league.id] = {}
    end
    
    -- Parse teams
    local teamColumns = rawData.teams.colonne
    for _, row in ipairs(rawData.teams.dati) do
        local team = {}
        for i, col in ipairs(teamColumns) do
            team[col] = row[i]
        end
        indexed.teams[team.id] = team
        table.insert(indexed.teamsByLeague[team.leagueId], team.id)
        indexed.playersByTeam[team.id] = {}
    end
    
    -- Parse players
    local playerColumns = rawData.players.colonne
    for _, row in ipairs(rawData.players.dati) do
        local player = {}
        for i, col in ipairs(playerColumns) do
            player[col] = row[i]
        end
        indexed.players[player.id] = player
        table.insert(indexed.playersByTeam[player.teamId], player.id)
    end
    
    -- Parse stadiums
    local stadiumColumns = rawData.stadiums.colonne
    for _, row in ipairs(rawData.stadiums.dati) do
        local stadium = {}
        for i, col in ipairs(stadiumColumns) do
            stadium[col] = row[i]
        end
        indexed.stadiums[stadium.id] = stadium
    end
    
    -- Parse managers
    local managerColumns = rawData.managers.colonne
    for _, row in ipairs(rawData.managers.dati) do
        local manager = {}
        for i, col in ipairs(managerColumns) do
            manager[col] = row[i]
        end
        indexed.managers[manager.id] = manager
    end
    
    return indexed
end

function DatapackLoader:LoadDatapack()
    local HttpService = game:GetService("HttpService")
    local success, result = pcall(function()
        return HttpService:GetAsync("https://cdn.jsdelivr.net/gh/pren8/footballdatabase/data2/datapack-september-2025_25-09-2025_156.json")
    end)
    
    if success then
        local rawData = HttpService:JSONDecode(result)
        return self:ParseDatapack(rawData)
    else
        warn("Failed to load datapack:", result)
        return nil
    end
end

return DatapackLoader
```

---

## 👤 Career Player System

### Player Creation
When starting a new career, the player creates their character:

**Input Fields:**
- First Name (TextBox)
- Last Name (TextBox)
- Nationality (Dropdown/ScrollingFrame selection)
- Position (Dropdown: ST, CF, LW, RW, CAM, CM, CDM, LB, RB, CB, GK)

**Initial Attributes:**
```lua
-- ModuleScript: CareerManager
local CareerManager = {}

function CareerManager:GeneratePlayerAbilities()
    return {
        pace = math.random(45, 65),
        shooting = math.random(40, 60),
        passing = math.random(40, 60),
        dribbling = math.random(40, 60),
        defending = math.random(35, 55),
        physical = math.random(40, 60),
        positioning = math.random(40, 60),
        technique = math.random(40, 60),
    }
end

function CareerManager:CreateCareerPlayer(firstName, lastName, nationality, position, datapack)
    -- Start in a lower league (division 3-4)
    local eligibleTeams = {}
    for _, team in pairs(datapack.teams) do
        local league = datapack.leagues[team.leagueId]
        if league and league.division >= 3 then
            table.insert(eligibleTeams, team)
        end
    end
    
    local startTeam = eligibleTeams[math.random(1, #eligibleTeams)]
    
    return {
        id = HttpService:GenerateGUID(false),
        firstName = firstName,
        lastName = lastName,
        nationality = nationality,
        position = position,
        birthdate = "2006/03/15", -- 18-19 years old
        currentTeamId = startTeam.id,
        jerseyNumber = math.random(1, 99),
        contractExpiry = {year = 2028, month = 6, day = 30},
        abilities = self:GeneratePlayerAbilities(),
        stamina = 100,
        stats = {
            matchesPlayed = 0,
            goals = 0,
            assists = 0,
            cleanSheets = 0,
            yellowCards = 0,
            redCards = 0,
            averageRating = 0,
            totalRating = 0,
        },
        relationships = {
            fans = 50,
            teammates = 50,
            manager = 50,
            board = 50,
        },
        injury = nil, -- {type = "injury"/"suspension", daysRemaining = X, reason = "..."}
    }
end

return CareerManager
```

### Stamina System
**Core Mechanic:**
- Starts at 100
- Decreases by 10-20 per match played
- Recovers through training (+15 to +30 per session)
- Low stamina (<50) reduces match performance

**Stamina Impact on Performance:**
```lua
function MatchSimulator:ApplyStaminaPenalty(baseRating, stamina)
    if stamina >= 70 then
        return baseRating -- No penalty
    elseif stamina >= 50 then
        return baseRating * 0.9 -- -10%
    elseif stamina >= 30 then
        return baseRating * 0.75 -- -25%
    else
        return baseRating * 0.6 -- -40%
    end
end
```

### Overall Rating Calculation
Position-based weighted average:

```lua
function CareerManager:CalculateOverall(player)
    local abilities = player.abilities
    local position = player.position
    
    local weights = {
        ST = {shooting = 0.25, pace = 0.20, positioning = 0.20, dribbling = 0.15, technique = 0.10, physical = 0.10},
        CF = {shooting = 0.20, dribbling = 0.20, passing = 0.15, positioning = 0.15, pace = 0.15, technique = 0.15},
        LW = {pace = 0.25, dribbling = 0.25, shooting = 0.15, positioning = 0.15, passing = 0.10, technique = 0.10},
        RW = {pace = 0.25, dribbling = 0.25, shooting = 0.15, positioning = 0.15, passing = 0.10, technique = 0.10},
        CAM = {passing = 0.25, dribbling = 0.20, shooting = 0.15, technique = 0.15, positioning = 0.15, pace = 0.10},
        CM = {passing = 0.25, technique = 0.20, dribbling = 0.15, physical = 0.15, defending = 0.15, pace = 0.10},
        CDM = {defending = 0.25, passing = 0.20, physical = 0.20, positioning = 0.15, technique = 0.10, pace = 0.10},
        LB = {defending = 0.25, pace = 0.20, physical = 0.15, passing = 0.15, positioning = 0.15, technique = 0.10},
        RB = {defending = 0.25, pace = 0.20, physical = 0.15, passing = 0.15, positioning = 0.15, technique = 0.10},
        CB = {defending = 0.30, physical = 0.25, positioning = 0.20, pace = 0.15, technique = 0.10},
        GK = {positioning = 0.30, technique = 0.30, physical = 0.20, defending = 0.20},
    }
    
    local posWeights = weights[position] or weights.CM
    local overall = 0
    
    for ability, weight in pairs(posWeights) do
        overall = overall + (abilities[ability] * weight)
    end
    
    return math.floor(overall)
end
```

### Player Progression (End of Season)
```lua
function CareerManager:ImprovePlayerAbilities(player)
    local age = self:CalculateAge(player.birthdate)
    local performanceBonus = 0
    
    if player.stats.averageRating >= 7.5 then
        performanceBonus = 3
    elseif player.stats.averageRating >= 7.0 then
        performanceBonus = 2
    elseif player.stats.averageRating >= 6.5 then
        performanceBonus = 1
    end
    
    -- Age-based progression
    local basePoints
    if age <= 21 then
        basePoints = 8 + performanceBonus
    elseif age <= 25 then
        basePoints = 5 + performanceBonus
    elseif age <= 29 then
        basePoints = 2 + performanceBonus
    else
        basePoints = math.max(0, performanceBonus - 1)
    end
    
    -- Distribute points randomly
    local abilities = {"pace", "shooting", "passing", "dribbling", "defending", "physical", "positioning", "technique"}
    for i = 1, basePoints do
        local ability = abilities[math.random(1, #abilities)]
        if player.abilities[ability] < 99 then
            player.abilities[ability] = player.abilities[ability] + 1
        end
    end
    
    return player
end
```

---

## ⚽ Match Simulation Engine

### Team Strength Calculation
```lua
-- ModuleScript: MatchSimulator
local MatchSimulator = {}

function MatchSimulator:CalculateTeamStrength(teamId, datapack)
    local playerIds = datapack.playersByTeam[teamId] or {}
    local totalOverall = 0
    local count = 0
    
    for _, playerId in ipairs(playerIds) do
        local player = datapack.players[playerId]
        if player then
            totalOverall = totalOverall + tonumber(player.overall)
            count = count + 1
        end
    end
    
    return count > 0 and (totalOverall / count) or 50
end
```

### Goal Generation (Poisson Distribution)
```lua
function MatchSimulator:CalculateExpectedGoals(teamStrength, opponentStrength)
    local strengthDiff = teamStrength - opponentStrength
    local baseGoals = 1.3
    return math.max(0.2, baseGoals + (strengthDiff / 30))
end

function MatchSimulator:PoissonGoals(lambda)
    local L = math.exp(-lambda)
    local k = 0
    local p = 1
    
    repeat
        k = k + 1
        p = p * math.random()
    until p <= L
    
    return k - 1
end
```

### Match Events Generation
```lua
function MatchSimulator:GenerateMatchEvents(homeGoals, awayGoals, homeTeamId, awayTeamId, datapack)
    local events = {}
    
    -- Goals
    for i = 1, homeGoals do
        local minute = math.random(1, 90)
        local scorer = self:GetRandomPlayer(homeTeamId, datapack, {"ST", "CF", "LW", "RW", "CAM"})
        if scorer then
            table.insert(events, {
                type = "goal",
                minute = minute,
                playerId = scorer.id,
                playerName = scorer.name .. " " .. scorer.surname,
                teamId = homeTeamId
            })
            
            -- 70% chance of assist
            if math.random() < 0.7 then
                local assister = self:GetRandomPlayer(homeTeamId, datapack, {"CM", "CAM", "LW", "RW"})
                if assister and assister.id ~= scorer.id then
                    table.insert(events, {
                        type = "assist",
                        minute = minute,
                        playerId = assister.id,
                        playerName = assister.name .. " " .. assister.surname,
                        teamId = homeTeamId
                    })
                end
            end
        end
    end
    
    -- Similar for away goals...
    
    -- Yellow cards (3-20% chance)
    for _, teamId in ipairs({homeTeamId, awayTeamId}) do
        if math.random() < 0.15 then
            local player = self:GetRandomPlayer(teamId, datapack)
            if player then
                table.insert(events, {
                    type = "yellow_card",
                    minute = math.random(1, 90),
                    playerId = player.id,
                    playerName = player.name .. " " .. player.surname,
                    teamId = teamId
                })
            end
        end
    end
    
    -- Red cards (1-3% chance)
    -- Injuries (2-5% chance)
    -- etc...
    
    return events
end
```

### Player Ratings
```lua
function MatchSimulator:CalculatePlayerRatings(events, homeTeamId, awayTeamId, datapack)
    local ratings = {} -- {[playerId] = rating}
    
    -- Base rating: 6.0
    local homePlayerIds = datapack.playersByTeam[homeTeamId] or {}
    local awayPlayerIds = datapack.playersByTeam[awayTeamId] or {}
    
    for _, playerId in ipairs(homePlayerIds) do
        ratings[playerId] = 6.0 + (math.random() * 0.5 - 0.25)
    end
    for _, playerId in ipairs(awayPlayerIds) do
        ratings[playerId] = 6.0 + (math.random() * 0.5 - 0.25)
    end
    
    -- Adjust based on events
    for _, event in ipairs(events) do
        if ratings[event.playerId] then
            if event.type == "goal" then
                ratings[event.playerId] = ratings[event.playerId] + 1.0
            elseif event.type == "assist" then
                ratings[event.playerId] = ratings[event.playerId] + 0.7
            elseif event.type == "yellow_card" then
                ratings[event.playerId] = ratings[event.playerId] - 0.3
            elseif event.type == "red_card" then
                ratings[event.playerId] = ratings[event.playerId] - 1.5
            end
        end
    end
    
    -- Find Man of the Match
    local motm = nil
    local maxRating = 0
    for playerId, rating in pairs(ratings) do
        if rating > maxRating then
            maxRating = rating
            motm = playerId
        end
    end
    
    return ratings, motm
end
```

### Complete Match Simulation
```lua
function MatchSimulator:SimulateMatch(homeTeamId, awayTeamId, matchday, season, datapack)
    local homeStrength = self:CalculateTeamStrength(homeTeamId, datapack)
    local awayStrength = self:CalculateTeamStrength(awayTeamId, datapack)
    
    local homeExpected = self:CalculateExpectedGoals(homeStrength, awayStrength)
    local awayExpected = self:CalculateExpectedGoals(awayStrength, homeStrength)
    
    local homeGoals = self:PoissonGoals(homeExpected)
    local awayGoals = self:PoissonGoals(awayExpected)
    
    local events = self:GenerateMatchEvents(homeGoals, awayGoals, homeTeamId, awayTeamId, datapack)
    local ratings, motm = self:CalculatePlayerRatings(events, homeTeamId, awayTeamId, datapack)
    
    return {
        id = HttpService:GenerateGUID(false),
        homeTeamId = homeTeamId,
        awayTeamId = awayTeamId,
        homeGoals = homeGoals,
        awayGoals = awayGoals,
        matchday = matchday,
        season = season,
        events = events,
        playerRatings = ratings,
        manOfTheMatch = motm,
        played = true
    }
end
```

---

## 🔄 Transfer System

### Free Agent Status
Player becomes a free agent when:
- Contract expires (contract end date reached)
- Player manually terminates contract (optional feature)

### Transfer Offer Structure
```lua
-- Table structure
TransferOffer = {
    id = "offer_123",
    fromTeamId = 3211, -- current team
    toTeamId = 3250, -- offering team
    offeredSalary = 75000,
    jerseyNumber = 10,
    dateOffered = 45, -- game day
    expiresInDays = 7,
    contractLength = 3, -- years
    bonuses = 50000, -- optional
    signOnFee = 100000, -- for free agents only
}
```

### Offer Generation
```lua
-- ModuleScript: TransferManager
local TransferManager = {}

function TransferManager:GenerateTransferOffers(player, datapack, currentDay, isFreeAgent)
    local playerOverall = CareerManager:CalculateOverall(player)
    local offers = {}
    
    local numOffers = isFreeAgent and math.random(2, 5) or math.random(1, 3)
    
    local currentTeam = datapack.teams[player.currentTeamId]
    local currentLeague = currentTeam and datapack.leagues[currentTeam.leagueId]
    
    -- Find eligible teams
    local eligibleTeams = {}
    for _, team in pairs(datapack.teams) do
        if (not isFreeAgent and team.id == player.currentTeamId) then
            -- Skip current team
        else
            local league = datapack.leagues[team.leagueId]
            if league and currentLeague then
                local divDiff = math.abs(league.division - currentLeague.division)
                if divDiff <= 2 then -- Max 2 divisions difference
                    local teamValue = team.teamValue or 0
                    local currentValue = currentTeam.teamValue or teamValue
                    if teamValue >= currentValue * 0.6 and teamValue <= currentValue * 1.5 then
                        table.insert(eligibleTeams, team)
                    end
                end
            end
        end
    end
    
    for i = 1, numOffers do
        if #eligibleTeams == 0 then break end
        
        local targetTeam = eligibleTeams[math.random(1, #eligibleTeams)]
        local baseSalary = playerOverall * 1000
        local variation = 0.8 + (math.random() * 0.4) -- 80-120%
        
        table.insert(offers, {
            id = HttpService:GenerateGUID(false),
            fromTeamId = player.currentTeamId,
            toTeamId = targetTeam.id,
            offeredSalary = math.floor(baseSalary * variation),
            jerseyNumber = math.random(1, 99),
            dateOffered = currentDay,
            expiresInDays = isFreeAgent and 14 or 7,
            contractLength = math.random(2, 4),
            signOnFee = isFreeAgent and math.floor(baseSalary * variation * 10 * (math.random() + 0.5)) or nil,
            bonuses = (math.random() > 0.5) and math.floor(baseSalary * 20 * math.random()) or nil,
        })
    end
    
    return offers
end

function TransferManager:CalculateBenchRisk(player, teamId, datapack)
    local playerOverall = CareerManager:CalculateOverall(player)
    local teamPlayerIds = datapack.playersByTeam[teamId] or {}
    
    local totalOverall = 0
    local count = 0
    
    for _, playerId in ipairs(teamPlayerIds) do
        local p = datapack.players[playerId]
        if p then
            totalOverall = totalOverall + tonumber(p.overall)
            count = count + 1
        end
    end
    
    local teamAvg = count > 0 and (totalOverall / count) or 50
    local benchRisk = math.max(0, math.min(100, (teamAvg - playerOverall) * 2))
    
    return math.floor(benchRisk)
end

return TransferManager
```

---

## 📅 Season & Date System

### Game Date Structure
```lua
GameDate = {
    year = 2025,
    month = 8, -- 1-12
    day = 15, -- 1-31
}
```

### Date Manager
```lua
-- ModuleScript: DateManager
local DateManager = {}

DateManager.DAYS_IN_MONTH = {31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31}

function DateManager:IsLeapYear(year)
    return (year % 4 == 0 and year % 100 ~= 0) or (year % 400 == 0)
end

function DateManager:GetDaysInMonth(month, year)
    if month == 2 and self:IsLeapYear(year) then
        return 29
    end
    return self.DAYS_IN_MONTH[month]
end

function DateManager:IncrementDate(gameDate)
    local newDate = {
        year = gameDate.year,
        month = gameDate.month,
        day = gameDate.day + 1
    }
    
    local daysInMonth = self:GetDaysInMonth(newDate.month, newDate.year)
    
    if newDate.day > daysInMonth then
        newDate.day = 1
        newDate.month = newDate.month + 1
        
        if newDate.month > 12 then
            newDate.month = 1
            newDate.year = newDate.year + 1
        end
    end
    
    return newDate
end

function DateManager:CalculateAge(birthdate, currentDate)
    -- birthdate format: "YYYY/MM/DD"
    local parts = {}
    for part in string.gmatch(birthdate, "[^/]+") do
        table.insert(parts, tonumber(part))
    end
    
    local birthYear, birthMonth, birthDay = parts[1], parts[2], parts[3]
    
    local age = currentDate.year - birthYear
    
    if currentDate.month < birthMonth or (currentDate.month == birthMonth and currentDate.day < birthDay) then
        age = age - 1
    end
    
    return age
end

function DateManager:IsBirthday(birthdate, currentDate)
    local parts = {}
    for part in string.gmatch(birthdate, "[^/]+") do
        table.insert(parts, tonumber(part))
    end
    
    local birthMonth, birthDay = parts[2], parts[3]
    return currentDate.month == birthMonth and currentDate.day == birthDay
end

function DateManager:FormatDate(gameDate)
    return string.format("%02d/%02d/%04d", gameDate.day, gameDate.month, gameDate.year)
end

function DateManager:GetSeasonStartDate(year)
    return {year = year, month = 8, day = 1} -- August 1st
end

function DateManager:GetSeasonEndDate(year)
    return {year = year + 1, month = 5, day = 31} -- May 31st next year
end

return DateManager
```

### Realistic Football Season Timeline
```lua
-- ModuleScript: SeasonManager
local SeasonManager = {}

function SeasonManager:GenerateMatchSchedule(leagueId, datapack, startDate)
    local teamIds = datapack.teamsByLeague[leagueId] or {}
    local numTeams = #teamIds
    
    if numTeams == 0 then return {} end
    
    -- Round-robin algorithm
    local matches = {}
    local matchday = 1
    local currentDate = {year = startDate.year, month = startDate.month, day = startDate.day}
    
    -- First half of season
    for round = 1, numTeams - 1 do
        for i = 1, math.floor(numTeams / 2) do
            local home = teamIds[i]
            local away = teamIds[numTeams - i + 1]
            
            table.insert(matches, {
                homeTeamId = home,
                awayTeamId = away,
                matchday = matchday,
                scheduledDate = {year = currentDate.year, month = currentDate.month, day = currentDate.day},
                played = false
            })
        end
        
        -- Rotate teams (keep first team fixed)
        local temp = teamIds[numTeams]
        for i = numTeams, 3, -1 do
            teamIds[i] = teamIds[i - 1]
        end
        teamIds[2] = temp
        
        matchday = matchday + 1
        
        -- Advance date by 7 days (weekly matches)
        for _ = 1, 7 do
            currentDate = DateManager:IncrementDate(currentDate)
        end
    end
    
    -- Second half (reverse fixtures)
    local firstHalfMatches = {}
    for _, match in ipairs(matches) do
        table.insert(firstHalfMatches, match)
    end
    
    for _, match in ipairs(firstHalfMatches) do
        table.insert(matches, {
            homeTeamId = match.awayTeamId,
            awayTeamId = match.homeTeamId,
            matchday = matchday,
            scheduledDate = {year = currentDate.year, month = currentDate.month, day = currentDate.day},
            played = false
        })
        
        matchday = matchday + 1
        for _ = 1, 7 do
            currentDate = DateManager:IncrementDate(currentDate)
        end
    end
    
    return matches
end

function SeasonManager:CalculateStandings(matches, leagueId, datapack)
    local standings = {}
    local teamIds = datapack.teamsByLeague[leagueId] or {}
    
    -- Initialize
    for _, teamId in ipairs(teamIds) do
        standings[teamId] = {
            teamId = teamId,
            played = 0,
            wins = 0,
            draws = 0,
            losses = 0,
            goalsFor = 0,
            goalsAgainst = 0,
            goalDifference = 0,
            points = 0
        }
    end
    
    -- Calculate from matches
    for _, match in ipairs(matches) do
        if match.played then
            local home = standings[match.homeTeamId]
            local away = standings[match.awayTeamId]
            
            if home and away then
                home.played = home.played + 1
                away.played = away.played + 1
                
                home.goalsFor = home.goalsFor + match.homeGoals
                home.goalsAgainst = home.goalsAgainst + match.awayGoals
                away.goalsFor = away.goalsFor + match.awayGoals
                away.goalsAgainst = away.goalsAgainst + match.homeGoals
                
                if match.homeGoals > match.awayGoals then
                    home.wins = home.wins + 1
                    home.points = home.points + 3
                    away.losses = away.losses + 1
                elseif match.homeGoals < match.awayGoals then
                    away.wins = away.wins + 1
                    away.points = away.points + 3
                    home.losses = home.losses + 1
                else
                    home.draws = home.draws + 1
                    away.draws = away.draws + 1
                    home.points = home.points + 1
                    away.points = away.points + 1
                end
                
                home.goalDifference = home.goalsFor - home.goalsAgainst
                away.goalDifference = away.goalsFor - away.goalsAgainst
            end
        end
    end
    
    -- Convert to array and sort
    local standingsArray = {}
    for _, standing in pairs(standings) do
        table.insert(standingsArray, standing)
    end
    
    table.sort(standingsArray, function(a, b)
        if a.points ~= b.points then
            return a.points > b.points
        elseif a.goalDifference ~= b.goalDifference then
            return a.goalDifference > b.goalDifference
        else
            return a.goalsFor > b.goalsFor
        end
    end)
    
    return standingsArray
end

return SeasonManager
```

---

## 📰 News System

### News Types
1. **Player Performance** (after match)
2. **Team News** (league position change)
3. **Transfer Rumors** (offers received)
4. **Injury News** (player injured)
5. **Contract News** (expiring contract)
6. **Starting XI** (starting status)
7. **Birthday** (player birthday)

### News Structure
```lua
NewsItem = {
    id = "news_123",
    type = "player_performance", -- or "team", "transfer", "injury", "contract", "starting_xi", "birthday"
    title = "Match-Winning Performance!",
    description = "You scored 2 goals in a crucial 3-1 victory...",
    date = {year = 2025, month = 9, day = 15},
    priority = "high", -- "low", "medium", "high"
}
```

### News Generation
```lua
-- ModuleScript: NewsGenerator
local NewsGenerator = {}

function NewsGenerator:GenerateMatchNews(player, match, playerRating, datapack)
    local news = {}
    
    if playerRating >= 8.0 then
        table.insert(news, {
            id = HttpService:GenerateGUID(false),
            type = "player_performance",
            title = "Outstanding Performance!",
            description = string.format("You received a rating of %.1f in the match!", playerRating),
            date = match.scheduledDate,
            priority = "high"
        })
    end
    
    -- Check for goals/assists in events
    local goals = 0
    local assists = 0
    for _, event in ipairs(match.events) do
        if event.playerId == player.id then
            if event.type == "goal" then goals = goals + 1 end
            if event.type == "assist" then assists = assists + 1 end
        end
    end
    
    if goals > 0 then
        table.insert(news, {
            id = HttpService:GenerateGUID(false),
            type = "player_performance",
            title = goals > 1 and "Multi-Goal Performance!" or "Goal Scored!",
            description = string.format("You scored %d goal%s!", goals, goals > 1 and "s" or ""),
            date = match.scheduledDate,
            priority = "high"
        })
    end
    
    return news
end

function NewsGenerator:GenerateBirthdayNews(player, currentDate)
    return {
        id = HttpService:GenerateGUID(false),
        type = "birthday",
        title = "Happy Birthday!",
        description = string.format("You turn %d years old today!", DateManager:CalculateAge(player.birthdate, currentDate)),
        date = currentDate,
        priority = "medium"
    }
end

function NewsGenerator:GenerateTransferNews(offer, datapack)
    local team = datapack.teams[offer.toTeamId]
    return {
        id = HttpService:GenerateGUID(false),
        type = "transfer",
        title = "Transfer Offer Received!",
        description = string.format("%s is interested in signing you.", team.name),
        date = offer.dateOffered,
        priority = "high"
    }
end

return NewsGenerator
```

---

## 🏋️ Training System

### Training Activities
```lua
-- ModuleScript: TrainingManager
local TrainingManager = {}

TrainingManager.TRAINING_ACTIVITIES = {
    {
        id = "sprint_drills",
        name = "Sprint Drills",
        description = "Improve pace and acceleration",
        affectedAbilities = {"pace"},
        abilityGain = {min = 1, max = 2},
        staminaRecovery = {min = 15, max = 25},
        duration = "1 day"
    },
    {
        id = "shooting_practice",
        name = "Shooting Practice",
        description = "Enhance finishing and shooting power",
        affectedAbilities = {"shooting"},
        abilityGain = {min = 1, max = 2},
        staminaRecovery = {min = 15, max = 25},
        duration = "1 day"
    },
    {
        id = "passing_drills",
        name = "Passing Drills",
        description = "Improve passing accuracy and vision",
        affectedAbilities = {"passing", "technique"},
        abilityGain = {min = 0, max = 1},
        staminaRecovery = {min = 20, max = 30},
        duration = "1 day"
    },
    {
        id = "dribbling_practice",
        name = "Dribbling Practice",
        description = "Enhance ball control and dribbling",
        affectedAbilities = {"dribbling", "technique"},
        abilityGain = {min = 0, max = 1},
        staminaRecovery = {min = 20, max = 30},
        duration = "1 day"
    },
    {
        id = "defensive_training",
        name = "Defensive Training",
        description = "Improve tackling and positioning",
        affectedAbilities = {"defending", "positioning"},
        abilityGain = {min = 0, max = 1},
        staminaRecovery = {min = 20, max = 30},
        duration = "1 day"
    },
    {
        id = "gym_session",
        name = "Gym Session",
        description = "Build strength and physical attributes",
        affectedAbilities = {"physical"},
        abilityGain = {min = 1, max = 2},
        staminaRecovery = {min = 15, max = 25},
        duration = "1 day"
    },
    {
        id = "rest_recovery",
        name = "Rest & Recovery",
        description = "Focus on stamina recovery",
        affectedAbilities = {},
        abilityGain = {min = 0, max = 0},
        staminaRecovery = {min = 25, max = 35},
        duration = "1 day"
    },
}

function TrainingManager:PerformTraining(player, activityId)
    local activity = nil
    for _, act in ipairs(self.TRAINING_ACTIVITIES) do
        if act.id == activityId then
            activity = act
            break
        end
    end
    
    if not activity then return player end
    
    -- Recover stamina
    local staminaGain = math.random(activity.staminaRecovery.min, activity.staminaRecovery.max)
    player.stamina = math.min(100, player.stamina + staminaGain)
    
    -- Improve abilities
    for _, ability in ipairs(activity.affectedAbilities) do
        local gain = math.random(activity.abilityGain.min, activity.abilityGain.max)
        if player.abilities[ability] and player.abilities[ability] < 99 then
            player.abilities[ability] = math.min(99, player.abilities[ability] + gain)
        end
    end
    
    return player
end

return TrainingManager
```

---

## 🎮 Game Loop ("Next Day" System)

### Daily Sequence
```lua
-- Script: GameLoop (ServerScript or LocalScript depending on architecture)
local function NextDay(gameState, player, datapack)
    -- 1. Increment date
    gameState.currentDate = DateManager:IncrementDate(gameState.currentDate)
    
    -- 2. Check for birthday
    if DateManager:IsBirthday(player.birthdate, gameState.currentDate) then
        local news = NewsGenerator:GenerateBirthdayNews(player, gameState.currentDate)
        table.insert(gameState.news, news)
    end
    
    -- 3. Decrement injury/suspension
    if player.injury then
        player = CareerManager:DecrementInjuryDays(player)
    end
    
    -- 4. Check for match today
    local todayMatch = nil
    for _, match in ipairs(gameState.fixtures) do
        if not match.played and 
           match.scheduledDate.year == gameState.currentDate.year and
           match.scheduledDate.month == gameState.currentDate.month and
           match.scheduledDate.day == gameState.currentDate.day then
            if match.homeTeamId == player.currentTeamId or match.awayTeamId == player.currentTeamId then
                todayMatch = match
                break
            end
        end
    end
    
    if todayMatch then
        -- Simulate match
        local simulatedMatch = MatchSimulator:SimulateMatch(
            todayMatch.homeTeamId,
            todayMatch.awayTeamId,
            todayMatch.matchday,
            gameState.season,
            datapack
        )
        
        -- Update match in fixtures
        for i, match in ipairs(gameState.fixtures) do
            if match == todayMatch then
                gameState.fixtures[i] = simulatedMatch
                break
            end
        end
        
        -- Update player stats
        local playerRating = simulatedMatch.playerRatings[player.id] or 6.0
        player = CareerManager:UpdatePlayerStatsFromMatch(player, simulatedMatch, true)
        
        -- Generate news
        local matchNews = NewsGenerator:GenerateMatchNews(player, simulatedMatch, playerRating, datapack)
        for _, news in ipairs(matchNews) do
            table.insert(gameState.news, news)
        end
        
        -- Keep only last 50 news items
        if #gameState.news > 50 then
            table.remove(gameState.news, 1)
        end
    end
    
    -- 5. Check for season end
    local allMatchesPlayed = true
    for _, match in ipairs(gameState.fixtures) do
        if not match.played then
            allMatchesPlayed = false
            break
        end
    end
    
    if allMatchesPlayed then
        -- End of season
        gameState.season = gameState.season + 1
        
        -- Calculate final standings
        local standings = SeasonManager:CalculateStandings(gameState.fixtures, player.currentLeagueId, datapack)
        gameState.standings = standings
        
        -- Improve player abilities
        player = CareerManager:ImprovePlayerAbilities(player)
        
        -- Reset stamina
        player.stamina = 100
        
        -- Generate new fixtures
        local nextSeasonStart = DateManager:GetSeasonStartDate(gameState.currentDate.year + 1)
        gameState.fixtures = SeasonManager:GenerateMatchSchedule(player.currentLeagueId, datapack, nextSeasonStart)
        
        -- Generate season end news
        table.insert(gameState.news, {
            id = HttpService:GenerateGUID(false),
            type = "season_end",
            title = "Season Complete!",
            description = string.format("Season %d has ended. Time to prepare for the next season!", gameState.season - 1),
            date = gameState.currentDate,
            priority = "high"
        })
    end
    
    -- 6. Expire old transfer offers
    for i = #gameState.offers, 1, -1 do
        local offer = gameState.offers[i]
        local daysSinceOffered = DateManager:DaysBetween(offer.dateOffered, gameState.currentDate)
        if daysSinceOffered >= offer.expiresInDays then
            table.remove(gameState.offers, i)
        end
    end
    
    return gameState, player
end
```

---

## 💾 Data Persistence (DataStore)

### Save Structure
```lua
-- Script: DataManager (ServerScript)
local DataStoreService = game:GetService("DataStoreService")
local PlayerDataStore = DataStoreService:GetDataStore("PlayerCareerData")

local DataManager = {}

function DataManager:SavePlayerData(userId, data)
    local success, err = pcall(function()
        PlayerDataStore:SetAsync("Player_" .. userId, {
            player = data.player,
            gameState = data.gameState,
            lastSaved = os.time()
        })
    end)
    
    if not success then
        warn("Failed to save data:", err)
    end
    
    return success
end

function DataManager:LoadPlayerData(userId)
    local success, data = pcall(function()
        return PlayerDataStore:GetAsync("Player_" .. userId)
    end)
    
    if success and data then
        return data
    else
        warn("Failed to load data or no data found")
        return nil
    end
end

function DataManager:DeletePlayerData(userId)
    local success, err = pcall(function()
        PlayerDataStore:RemoveAsync("Player_" .. userId)
    end)
    
    return success
end

-- Auto-save every 5 minutes
game:GetService("RunService").Heartbeat:Connect(function()
    -- Implement auto-save logic
end)

return DataManager
```

### Data Structure to Save
```lua
SaveData = {
    player = {
        -- Full CareerPlayer table
    },
    gameState = {
        currentDate = {year = 2025, month = 9, day = 15},
        season = 1,
        fixtures = {}, -- Array of Match tables
        standings = {}, -- Array of Standing tables
        offers = {}, -- Array of TransferOffer tables
        news = {}, -- Array of NewsItem tables (last 50)
    },
    lastSaved = 1234567890 -- Unix timestamp
}
```

---

## 🎨 UI System (Pure ScreenGuis)

### UI Hierarchy
```
StarterGui
├── MainScreenGui (ScreenGui)
│   ├── MainMenu (Frame) - Player creation
│   ├── Dashboard (Frame) - Main game screen
│   ├── TeamSelection (Frame) - Browse and join teams
│   ├── TeamBrowser (Frame) - View other team squads
│   ├── MatchSimulation (Frame) - View match results
│   ├── Training (Frame) - Training activities
│   ├── TeamInfo (Frame) - Your team's squad
│   ├── LeagueExplorer (Frame) - League standings
│   ├── TransferOffers (Frame) - View and accept offers
│   ├── NewsConsole (Frame) - News feed
│   └── CareerStats (Frame) - Career statistics
```

### Example UI Structure (Dashboard)
```lua
-- LocalScript: UIManager
local function CreateDashboard(screenGui, player, gameState, datapack)
    local dashboard = Instance.new("Frame")
    dashboard.Name = "Dashboard"
    dashboard.Size = UDim2.new(1, 0, 1, 0)
    dashboard.BackgroundColor3 = Color3.fromRGB(15, 23, 42) -- Dark background
    dashboard.Parent = screenGui
    
    -- Header
    local header = Instance.new("Frame")
    header.Name = "Header"
    header.Size = UDim2.new(1, 0, 0, 80)
    header.BackgroundColor3 = Color3.fromRGB(30, 41, 59)
    header.Parent = dashboard
    
    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(0.5, 0, 1, 0)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = player.firstName .. " " .. player.lastName
    titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    titleLabel.TextSize = 24
    titleLabel.Font = Enum.Font.GothamBold
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Parent = header
    
    local dateLabel = Instance.new("TextLabel")
    dateLabel.Size = UDim2.new(0.5, 0, 1, 0)
    dateLabel.Position = UDim2.new(0.5, 0, 0, 0)
    dateLabel.BackgroundTransparency = 1
    dateLabel.Text = DateManager:FormatDate(gameState.currentDate)
    dateLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    dateLabel.TextSize = 18
    dateLabel.Font = Enum.Font.Gotham
    dateLabel.TextXAlignment = Enum.TextXAlignment.Right
    dateLabel.Parent = header
    
    -- Stamina Bar
    local staminaFrame = Instance.new("Frame")
    staminaFrame.Size = UDim2.new(0.8, 0, 0, 30)
    staminaFrame.Position = UDim2.new(0.1, 0, 0, 100)
    staminaFrame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    staminaFrame.Parent = dashboard
    
    local staminaBar = Instance.new("Frame")
    staminaBar.Size = UDim2.new(player.stamina / 100, 0, 1, 0)
    staminaBar.BackgroundColor3 = player.stamina >= 70 and Color3.fromRGB(34, 197, 94) or
                                   player.stamina >= 50 and Color3.fromRGB(234, 179, 8) or
                                   Color3.fromRGB(239, 68, 68)
    staminaBar.Parent = staminaFrame
    
    local staminaLabel = Instance.new("TextLabel")
    staminaLabel.Size = UDim2.new(1, 0, 1, 0)
    staminaLabel.BackgroundTransparency = 1
    staminaLabel.Text = "Stamina: " .. player.stamina .. "%"
    staminaLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    staminaLabel.TextSize = 16
    staminaLabel.Font = Enum.Font.GothamBold
    staminaLabel.Parent = staminaFrame
    
    -- Next Day Button
    local nextDayBtn = Instance.new("TextButton")
    nextDayBtn.Size = UDim2.new(0.3, 0, 0, 60)
    nextDayBtn.Position = UDim2.new(0.35, 0, 0, 150)
    nextDayBtn.BackgroundColor3 = Color3.fromRGB(59, 130, 246)
    nextDayBtn.Text = "Next Day"
    nextDayBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    nextDayBtn.TextSize = 20
    nextDayBtn.Font = Enum.Font.GothamBold
    nextDayBtn.Parent = dashboard
    
    nextDayBtn.MouseButton1Click:Connect(function()
        -- Trigger NextDay logic
        gameState, player = NextDay(gameState, player, datapack)
        -- Refresh UI
        dashboard:Destroy()
        CreateDashboard(screenGui, player, gameState, datapack)
    end)
    
    -- Navigation Buttons
    local navFrame = Instance.new("Frame")
    navFrame.Size = UDim2.new(0.9, 0, 0.5, 0)
    navFrame.Position = UDim2.new(0.05, 0, 0.4, 0)
    navFrame.BackgroundTransparency = 1
    navFrame.Parent = dashboard
    
    local navButtons = {
        {name = "Team Selection", page = "TeamSelection"},
        {name = "Training", page = "Training"},
        {name = "Team Info", page = "TeamInfo"},
        {name = "League Table", page = "LeagueExplorer"},
        {name = "Transfer Offers", page = "TransferOffers"},
        {name = "News", page = "NewsConsole"},
        {name = "Career Stats", page = "CareerStats"},
        {name = "Browse Teams", page = "TeamBrowser"},
    }
    
    local buttonWidth = 0.45
    local buttonHeight = 0.15
    local buttonPaddingX = 0.05
    local buttonPaddingY = 0.05
    
    for i, btnInfo in ipairs(navButtons) do
        local col = (i - 1) % 2
        local row = math.floor((i - 1) / 2)
        
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(buttonWidth, 0, buttonHeight, 0)
        btn.Position = UDim2.new(col * (buttonWidth + buttonPaddingX), 0, row * (buttonHeight + buttonPaddingY), 0)
        btn.BackgroundColor3 = Color3.fromRGB(71, 85, 105)
        btn.Text = btnInfo.name
        btn.TextColor3 = Color3.fromRGB(255, 255, 255)
        btn.TextSize = 18
        btn.Font = Enum.Font.Gotham
        btn.Parent = navFrame
        
        btn.MouseButton1Click:Connect(function()
            -- Hide dashboard, show target page
            dashboard.Visible = false
            screenGui[btnInfo.page].Visible = true
        end)
    end
    
    return dashboard
end
```

### Color Scheme (Consistent Theme)
```lua
-- ModuleScript: UITheme
local UITheme = {}

UITheme.Colors = {
    Background = Color3.fromRGB(15, 23, 42),
    Surface = Color3.fromRGB(30, 41, 59),
    Primary = Color3.fromRGB(59, 130, 246),
    Secondary = Color3.fromRGB(147, 51, 234),
    Success = Color3.fromRGB(34, 197, 94),
    Warning = Color3.fromRGB(234, 179, 8),
    Danger = Color3.fromRGB(239, 68, 68),
    Text = Color3.fromRGB(255, 255, 255),
    TextMuted = Color3.fromRGB(148, 163, 184),
    Border = Color3.fromRGB(51, 65, 85),
}

UITheme.Fonts = {
    Bold = Enum.Font.GothamBold,
    Regular = Enum.Font.Gotham,
    Light = Enum.Font.GothamLight,
}

return UITheme
```

---

## 📱 UI Pages (Detailed)

### 1. Player Creation (MainMenu)
**Components:**
- TextBox for First Name
- TextBox for Last Name
- Dropdown/ScrollingFrame for Nationality selection
- Dropdown for Position selection (ST, CF, LW, RW, CAM, CM, CDM, LB, RB, CB, GK)
- "Create Career" Button

### 2. Game Dashboard
**Components:**
- Player name and overall rating
- Current date display
- Stamina bar (color-coded: green >70, yellow 50-70, red <50)
- "Next Day" button (large, prominent)
- Navigation buttons to all other pages
- Quick stats display (matches, goals, assists)

### 3. Team Selection Page
**Components:**
- Search/filter by league
- List of teams with:
  - Team name, league, division
  - Average team overall
  - Bench risk percentage (color-coded)
- "Join Team" button for each team
- Confirmation dialog

### 4. Team Browser Page
**Components:**
- Search bar for teams
- Filter by league dropdown
- Scrollable team list
- Click team to view squad:
  - Player name, position, overall, nationality
  - Age, contract expiry
- Back button

### 5. Match Simulation Page
**Components:**
- Match result (Home vs Away, score)
- Match events timeline (goals, assists, cards, injuries)
- Player rating display
- Man of the Match highlight
- "Back to Dashboard" button

### 6. Training Page
**Components:**
- List of training activities (cards/buttons)
- Each activity shows:
  - Name
  - Description
  - Affected abilities
  - Stamina recovery
- "Start Training" button
- Confirmation: "Training complete! +X stamina, +Y ability"

### 7. Team Info Page
**Components:**
- Team name, league, stadium
- Manager information
- Team overall average
- Scrollable squad list with:
  - Player name, position, overall
  - Nationality, age
- Back button

### 8. League Explorer Page
**Components:**
- League selection dropdown
- Standings table:
  - Position, Team, Played, Wins, Draws, Losses, GF, GA, GD, Points
- Highlight player's team
- Filter by division
- Back button

### 9. Transfer Offers Page
**Components:**
- List of active offers (cards)
- Each offer shows:
  - Offering team name and logo
  - Salary, contract length, jersey number
  - Sign-on fee and bonuses (if applicable)
  - Days remaining until expiration
- "Accept" and "Decline" buttons
- Empty state: "No offers at the moment"

### 10. News Console Page
**Components:**
- Scrollable list of news items
- Each news item shows:
  - Title (bold)
  - Description
  - Date
  - Priority icon/color
- Filter by type (all, performance, transfer, injury, etc.)
- Back button

### 11. Career Stats Page
**Components:**
- Overall career statistics:
  - Seasons played
  - Total matches, goals, assists
  - Average rating
  - Yellow/red cards
  - Teams played for
- Season-by-season breakdown
- Back button

---

## 🔧 Implementation Architecture

### Folder Structure
```
ServerScriptService
├── DataManager (ModuleScript) - DataStore save/load
├── DatapackLoader (ModuleScript) - Load and parse datapack
├── CareerManager (ModuleScript) - Player creation and progression
├── MatchSimulator (ModuleScript) - Match simulation logic
├── SeasonManager (ModuleScript) - Fixtures and standings
├── TransferManager (ModuleScript) - Transfer offers and team selection
├── TrainingManager (ModuleScript) - Training activities
├── DateManager (ModuleScript) - Date management
├── NewsGenerator (ModuleScript) - News generation
└── GameLoop (Script) - Main server logic

ReplicatedStorage
├── UITheme (ModuleScript) - UI colors and fonts
└── RemoteEvents (Folder)
    ├── LoadDatapackEvent (RemoteEvent)
    ├── SavePlayerDataEvent (RemoteEvent)
    ├── NextDayEvent (RemoteEvent)
    └── ... (other events)

StarterGui
└── MainScreenGui (ScreenGui)
    └── (All UI Frames created dynamically via LocalScript)

StarterPlayer
└── StarterPlayerScripts
    └── UIManager (LocalScript) - Client-side UI management
```

### Client-Server Communication
```lua
-- Example: Next Day Button
-- Client (LocalScript):
local nextDayBtn = script.Parent
local nextDayEvent = game.ReplicatedStorage.RemoteEvents.NextDayEvent

nextDayBtn.MouseButton1Click:Connect(function()
    nextDayEvent:FireServer()
end)

-- Server (Script):
local nextDayEvent = game.ReplicatedStorage.RemoteEvents.NextDayEvent

nextDayEvent.OnServerEvent:Connect(function(player)
    local userId = player.UserId
    local data = DataManager:LoadPlayerData(userId)
    
    if data then
        data.gameState, data.player = GameLoop.NextDay(data.gameState, data.player, datapack)
        DataManager:SavePlayerData(userId, data)
        
        -- Send updated data back to client
        nextDayEvent:FireClient(player, data)
    end
end)
```

---

## ⚡ Performance Optimization

### 1. Use Tables for Fast Lookups
```lua
-- ✅ GOOD: O(1) lookup
local player = datapack.players[playerId]

-- ❌ BAD: O(n) lookup
for _, player in ipairs(datapack.players) do
    if player.id == playerId then
        break
    end
end
```

### 2. Cache Frequently Used Data
```lua
-- Cache team strength calculations
local teamStrengthCache = {}

function MatchSimulator:GetTeamStrength(teamId, datapack)
    if not teamStrengthCache[teamId] then
        teamStrengthCache[teamId] = self:CalculateTeamStrength(teamId, datapack)
    end
    return teamStrengthCache[teamId]
end
```

### 3. Limit UI Updates
```lua
-- Update UI only when visible
local dashboard = screenGui:FindFirstChild("Dashboard")
if dashboard and dashboard.Visible then
    UpdateDashboardUI(dashboard, player, gameState)
end
```

### 4. Pagination for Large Lists
```lua
-- Display only 20 teams at a time
local function DisplayTeamPage(teamList, pageNumber, itemsPerPage)
    local startIndex = (pageNumber - 1) * itemsPerPage + 1
    local endIndex = math.min(startIndex + itemsPerPage - 1, #teamList)
    
    for i = startIndex, endIndex do
        -- Create UI for teamList[i]
    end
end
```

---

## 🎯 Critical Implementation Rules

### Data Integrity
1. **Always validate DataStore reads** - Provide default values
2. **Type checking** - Use `type()` to verify data types
3. **Error handling** - Wrap DataStore calls in `pcall()`
4. **Backup saves** - Consider saving to multiple DataStores

### Game Balance
1. **Stamina recovery** - Training should be essential
2. **Transfer logic** - Offers should be realistic (division-based)
3. **Match realism** - Use Poisson distribution for goals
4. **Progression curve** - Young players improve faster

### User Experience
1. **Loading states** - Show "Loading..." when fetching datapack
2. **Confirmations** - Ask before major decisions (team changes, contract termination)
3. **Auto-save** - Save every 5 minutes and after important events
4. **Feedback** - Show toast notifications for actions

---

## 🧪 Testing Checklist

### Core Systems
- [ ] Datapack loads successfully from CDN
- [ ] Player creation works with all positions
- [ ] Match simulation generates realistic results
- [ ] Stamina decreases after matches and recovers with training
- [ ] Overall rating calculates correctly for each position
- [ ] Transfer offers generate for appropriate teams
- [ ] Fixtures generate in correct round-robin format
- [ ] Standings calculate correctly (points, GD, GF)

### UI/UX
- [ ] All pages are accessible via navigation
- [ ] Back buttons work correctly
- [ ] Stamina bar color-codes properly
- [ ] News feed displays recent items
- [ ] Transfer offers show all details
- [ ] Team browser displays all squads
- [ ] Responsive to different screen sizes (mobile/desktop)

### Data Persistence
- [ ] Player data saves to DataStore
- [ ] Player data loads correctly on rejoin
- [ ] Auto-save triggers every 5 minutes
- [ ] Delete career works properly

### Game Logic
- [ ] Birthday system detects correct dates
- [ ] Date increments correctly (leap years, month boundaries)
- [ ] Season ends trigger progression
- [ ] Injuries/suspensions decrement daily
- [ ] Contract expiry leads to free agency

---

## 🚀 Implementation Priority

### Phase 1: Foundation (Week 1)
1. Create UI theme and basic structure
2. Implement datapack loading and parsing
3. Build player creation page
4. Set up DataStore save/load system

### Phase 2: Core Gameplay (Week 2)
1. Implement match simulation engine
2. Create game loop (Next Day system)
3. Build dashboard and navigation
4. Add date manager and season system

### Phase 3: Features (Week 3)
1. Implement training system
2. Create transfer offer system
3. Build team selection page
4. Add news generation

### Phase 4: Polish (Week 4)
1. Build remaining UI pages (League Explorer, Team Browser, etc.)
2. Add birthday system
3. Implement career stats tracking
4. Optimize performance and fix bugs

### Phase 5: Advanced Features (Optional)
1. Add achievements system
2. Implement relationship system effects
3. Add more training activities
4. Create career milestones

---

## 🎮 Success Criteria

✅ Pure UI-based game (no 3D gameplay required)  
✅ Complete career simulation from creation to retirement  
✅ Realistic match results and player progression  
✅ Persistent save system via DataStore  
✅ Smooth UX with responsive navigation  
✅ Balanced gameplay (stamina, transfers, progression)  
✅ Comprehensive news and statistics tracking  
✅ Works on both PC and mobile Roblox  

---

## 📝 Notes for Developers

1. **Roblox DataStore has rate limits** - Implement queuing for saves
2. **HttpService must be enabled** - Enable in Game Settings to load datapack
3. **UI scaling** - Use `UDim2` for responsive layouts
4. **Performance** - Avoid excessive `:GetChildren()` calls in loops
5. **Testing** - Test in Roblox Studio first, then publish for testing on multiple devices
6. **Mobile considerations** - Ensure buttons are large enough (minimum 60x60 pixels)

---

**This specification is complete and ready for implementation in Roblox!** 🎉